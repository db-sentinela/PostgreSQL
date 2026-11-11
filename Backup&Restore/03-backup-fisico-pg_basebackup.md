# 03 — Backup físico (pg_basebackup)

## O que é

Backup físico é uma **cópia dos arquivos** do cluster — os mesmos arquivos que vivem
no `data_dir`, byte a byte. Não é SQL: é a estrutura em disco inteira. A ferramenta
nativa é o **`pg_basebackup`**.

## Por que importa

- **Restaura rápido**: recuperar é basicamente "copiar de volta e subir" — ótimo
  para **RTO baixo** em bancos grandes.
- É a **base** para o PITR (doc [05](05-pitr.md)) e para criar réplicas.
- Captura o cluster **exatamente** como está (todos os bancos, config incluída).

## Lógico x físico (a diferença que confunde todo mundo)

| | Lógico (`pg_dump`) | Físico (`pg_basebackup`) |
|---|---|---|
| Copia | Comandos SQL / dados | Arquivos do cluster |
| Granularidade | 1 objeto, 1 banco | **Cluster inteiro** |
| Restaura | Recriando objetos (lento) | Copiando de volta (rápido) |
| Entre versões major | Sim (portável) | **Não** (mesma major) |
| Serve para PITR | Não | **Sim** |

## Como reproduzir

```bash
PG=/usr/pgsql-18/bin
# formato "plain" (um data_dir pronto), com WAL via streaming:
PGPASSWORD=Repl#2026 $PG/pg_basebackup \
  -h 192.168.1.15 -p 5432 -U replicator \
  -D /var/lib/pgsql/backups/basebackup \
  -Fp -Xs -c fast -P
```

- `-Fp` = formato plain (sai um `data_dir` completo). `-Ft` = tar.
- `-Xs` = **stream** do WAL junto (o backup já vem consistente e startável).
- `-c fast` = checkpoint imediato (não espera o próximo natural).
- `-P` = barra de progresso.

> Conexão de **replicação**: `pg_basebackup` usa o protocolo de replicação. O
> `pg_hba.conf` precisa de uma linha `replication` para o IP/usuário (no lab:
> `host replication replicator 192.168.1.0/24 scram-sha-256`). Por isso usamos o IP
> da LAN, e não `127.0.0.1` (que não está nessa regra).

## O que você vê (saída real)

```
waiting for checkpoint
113540/113540 kB (100%), 1/1 tablespace
```

E o resultado é um **data_dir completo**, pronto para subir:

```
backup_label      <- marca o início do backup (essencial p/ recuperação)
backup_manifest   <- lista de arquivos + checksums (para verificar integridade)
base/  global/  pg_wal/  pg_xact/ ...
PG_VERSION = 18
tamanho: 127M
```

O `backup_label` e o `backup_manifest` são o que diferenciam um backup físico
**válido** de uma simples cópia de pasta: eles registram o ponto de início e
permitem **verificar** a integridade.

## Formato tar comprimido (e uma pegadinha real)

```bash
PGPASSWORD=Repl#2026 $PG/pg_basebackup -h 192.168.1.15 -U replicator \
  -D /destino -Ft -z -Xs -c fast
```

⚠️ **Pegadinha que aconteceu no teste:** se o diretório de destino for criado por
`root`, o `pg_basebackup` (que roda como `postgres`) falha:

```
pg_basebackup: error: could not create compressed file ".../base.tar.gz": Permission denied
```

**Correção:** o diretório de destino tem que pertencer ao usuário `postgres`:

```bash
sudo -u postgres install -d /destino      # cria já com o dono certo
```

## Verificar a integridade de um backup

```bash
pg_verifybackup /var/lib/pgsql/backups/basebackup
```

Lê o `backup_manifest` e confere os checksums de todos os arquivos. Rode isso para
ter **certeza** de que o backup físico não está corrompido.

## Como restaurar (resumo)

1. Pare o PostgreSQL de destino (ou use um `data_dir` novo).
2. Copie o conteúdo do backup para o `data_dir`.
3. Ajuste permissões (`chown -R postgres:postgres`, `chmod 700`).
4. Suba o PostgreSQL — ele recupera a partir do `backup_label` e abre.

> Para recuperar até um **instante** (e não só "o momento do backup"), você soma o
> **WAL arquivado** → isso é PITR, docs [04](04-wal-e-archiving.md) e
> [05](05-pitr.md).

## Lições

- Físico = cópia dos arquivos; restaura rápido; é o cluster **inteiro**.
- `pg_basebackup -Fp -Xs -c fast` já entrega um `data_dir` consistente e startável.
- `backup_label` + `backup_manifest` são a marca de um backup físico válido;
  `pg_verifybackup` confere.

## Pegadinhas

- Não restaura "uma tabela" — para granularidade, use backup **lógico** (doc [02]).
- Não atravessa versões **major** nem arquiteturas diferentes.
- Destino do backup deve pertencer ao usuário `postgres` (a pegadinha acima).
- Copiar o `data_dir` "na mão" com o banco **ligado** (sem `pg_basebackup` nem
  `pg_backup_start`) gera um backup **inconsistente**. Use a ferramenta certa.
