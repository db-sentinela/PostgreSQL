# 05 — PITR (Point-In-Time Recovery)

## O que é

PITR é recuperar o banco até um **instante exato** no tempo. É a resposta para o
pesadelo clássico: *"alguém rodou `DROP TABLE` em produção às 14:32"*. Com PITR você
restaura o estado de **14:31:59** — antes do erro.

## A ideia em 3 peças

1. Um **backup físico** (base backup) — a foto do cluster num momento inicial.
2. O **WAL arquivado** continuamente desde então — o "diário" de **toda** mudança.
3. A recuperação = **restaurar a foto** e **reaplicar o WAL** até o instante que
   você escolher (`recovery_target_time`), parando **antes** do desastre.

```
   base backup            WAL arquivado continuamente
   (foto inicial) ----[ T0 ... inserts ... T1 ... DROP(14:32) ... ]
                                            ^
                           recupera ATÉ AQUI (T1) e para
```

## Por que importa

Sem PITR, seu RPO é a distância até o último `pg_dump` (pode ser 24 h). Com PITR, o
RPO cai para **segundos** — e você recupera de **erros lógicos** (um `DELETE`/`DROP`
errado), que a HA/replicação **não** desfaz (a réplica copia o erro).

## Pré-requisito: WAL archiving

No `postgresql.conf` da origem (ver doc [04](04-wal-e-archiving.md)):

```ini
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /caminho/archive/%f && cp %p /caminho/archive/%f'
```

`%p` = caminho do WAL a arquivar; `%f` = só o nome do arquivo. O `test !` evita
sobrescrever um WAL já arquivado.

## Homologação real (passo a passo testado)

Feito numa instância **standalone** (porta 5500), para não mexer no cluster de HA.

```bash
# 1) base backup (a foto):
pg_basebackup -h 127.0.0.1 -p 5500 -U postgres -D /.../basebackup -Fp -Xs -c fast

# 2) depois do backup, mais dados e então o DESASTRE:
#    (inseri 50 linhas, guardei T1 = now(), esperei, e DROPEI a tabela)

# 3) restore: copiar a foto, apontar o alvo no tempo e deixar o WAL ser reaplicado:
cp -a /.../basebackup /.../restored
cat >> /.../restored/postgresql.conf <<EOF
restore_command = 'cp /.../archive/%f %p'
recovery_target_time = '2026-11-11 19:04:36.999577-03'
recovery_target_action = 'promote'
EOF
touch /.../restored/recovery.signal      # sinaliza "entre em recuperação"
pg_ctl -D /.../restored start
```

### O que você vê (saída real)

A tabela tinha **150 linhas** antes do desastre; o `DROP` aconteceu **depois** de T1.
Após o PITR:

```
############ PROVA DO PITR
tabela vendas foi RECUPERADA? -> t
linhas recuperadas = 150   (o DROP ficou de fora)
ainda em recovery? -> f
```

E o **log de recuperação** mostra o corte exato:

```
LOG:  starting point-in-time recovery to 2026-11-11 19:04:36.999577-03
LOG:  consistent recovery state reached at 0/2000120
LOG:  recovery stopping before commit of transaction 755, time 2026-11-11 19:04:39.04
LOG:  redo done at 0/3013380
LOG:  last completed transaction was at log time 2026-11-11 19:04:35.88
```

Leia: a recuperação **parou antes** da transação do `DROP` (committed às 19:04:39),
aplicando tudo até T1 (19:04:36). Resultado: dados intactos, desastre desfeito.

## Os parâmetros de alvo (você escolhe o critério de parada)

| Parâmetro | Para até... |
|-----------|-------------|
| `recovery_target_time` | um **horário** (`'2026-11-11 14:31:59'`) |
| `recovery_target_xid` | uma **transação** específica |
| `recovery_target_lsn` | uma **posição** no WAL |
| `recovery_target_name` | um **marco** criado com `pg_create_restore_point('nome')` |
| `recovery_target = 'immediate'` | o fim do **base backup** (consistência mínima) |

- `recovery_target_action`: `promote` (abre o banco — o usual), `pause` (para e
  espera você decidir), `shutdown`.
- `recovery_target_inclusive` (padrão `on`): inclui ou não a transação exatamente no
  alvo.

## Dica de ouro para a vida real

Quando o desastre acontece, **você raramente sabe o horário exato**. Estratégias:

- Use `recovery_target_action = 'pause'`, inspecione os dados, e se passou do ponto,
  ajuste o alvo e refaça — **sem** promover ainda.
- Restaure sempre num **ambiente separado** primeiro; valide; só então decida.
- `pg_create_restore_point('antes_do_deploy')` **antes** de operações de risco cria
  um alvo nomeado fácil de mirar.

## Lições

- PITR = base backup **+** WAL contínuo → recupera a **qualquer instante**.
- É a única defesa contra **erro lógico** (DROP/DELETE), que a HA não desfaz.
- O log de recovery confirma exatamente onde parou — sempre verifique.
- Alvo por tempo, XID, LSN ou nome; `pause` é seu amigo quando o horário é incerto.

## Pegadinhas

- Sem `archive_mode = on` **antes** do incidente, **não há** PITR — o WAL já foi
  reciclado. Configure archiving **hoje**, não depois.
- Monitore o `archive_command`: se ele falhar, o WAL **se acumula** no `pg_wal` e
  pode **encher o disco** (e aí o banco para). Alertas são obrigatórios.
- Restaure para um **data_dir novo**; nunca por cima do cluster vivo.
- Em um cluster Patroni, PITR é feito **fora** do controle do Patroni (instância de
  restauração à parte) — ver doc [07](07-backup-em-cluster-patroni.md).
