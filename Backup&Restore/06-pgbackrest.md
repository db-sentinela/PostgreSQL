# 06 — pgBackRest (a ferramenta de produção)

## O que é

`pg_basebackup` e o archiving manual ensinam **como** o backup físico/PITR funciona
— mas montar isso "na unha" em produção é trabalhoso e fácil de errar. O
**pgBackRest** é a ferramenta **padrão da indústria** que automatiza tudo: backups
**full/diferencial/incremental**, archiving, retenção, verificação de integridade,
paralelismo, compressão e **restore/PITR com um comando**.

## Por que usar em vez do manual

| Manual (`pg_basebackup` + scripts) | pgBackRest |
|---|---|
| Você escreve/monitora o `archive_command` | Ele cuida do `archive-push`/`archive-get` |
| Só full (cópia inteira toda vez) | Full, **diferencial** e **incremental** |
| Retenção na mão | `repo1-retention-full` automático |
| Verificação manual | Checksums e `verify` embutidos |
| PITR montado à mão | `restore --type=time --target=...` |

## Instalação (com a pegadinha real)

No Fedora/PGDG, o pacote **não** está no repo da versão do Postgres — está no
repositório **`pgdg-common`**. Foi exatamente o que travou na primeira tentativa
(`pgbackrest: command not found`). A correção:

```bash
cat > /etc/yum.repos.d/pgdg-common.repo <<EOF
[pgdg-common]
name=PostgreSQL common for Fedora 42 - x86_64
baseurl=https://download.postgresql.org/pub/repos/yum/common/fedora/fedora-42-x86_64/
enabled=1
gpgcheck=1
gpgkey=https://download.postgresql.org/pub/repos/yum/keys/PGDG-RPM-GPG-KEY-Fedora
EOF
dnf install -y pgbackrest
pgbackrest version          # -> pgBackRest 2.58.0
```

## Conceitos

- **Repositório** (`repo`): onde os backups e o WAL ficam guardados (disco local,
  NFS, S3, Azure, GCS...).
- **Stanza**: a configuração de **um** cluster PostgreSQL a ser copiado (um nome,
  ex.: `demo`).

## Configuração (`/etc/pgbackrest/pgbackrest.conf`)

```ini
[global]
repo1-path=/var/lib/pgsql/pgbr/repo
repo1-retention-full=2      # mantém os 2 últimos fulls
start-fast=y                # checkpoint imediato

[demo]
pg1-path=/var/lib/pgsql/pgbr/pgdata
pg1-port=5502
```

E no PostgreSQL, o archiving aponta para o pgBackRest:

```ini
archive_mode = on
archive_command = 'pgbackrest --stanza=demo archive-push %p'
```

## Inicializar e verificar

```bash
pgbackrest --stanza=demo stanza-create    # cria a stanza no repositório
pgbackrest --stanza=demo check            # valida config + archiving (faça sempre)
```

## Fazer backup

```bash
pgbackrest --stanza=demo --type=full backup   # cópia completa
pgbackrest --stanza=demo --type=diff backup   # só o que mudou desde o último full
pgbackrest --stanza=demo --type=incr backup   # só o que mudou desde o último backup
```

## Restore e PITR (homologado — saída real)

Cenário testado: 150 linhas, backup full, `DROP TABLE` depois do instante T1,
restauração até T1.

```bash
pgbackrest --stanza=demo --delta --type=time \
   --target='2026-11-11 20:47:38' --target-action=promote restore
pg_ctl -D /var/lib/pgsql/pgbr/pgdata start
```

- `--delta` = restaura só os arquivos diferentes (mais rápido; reaproveita o que já
  está no destino).
- `--type=time --target=...` = PITR até o horário.
- `--target-action=promote` = ao atingir o alvo, abre o banco para escrita.

### Resultado real

```
############ PROVA pgBackRest PITR
tabela recuperada? -> t
linhas = 150   (o DROP ficou de fora)
```

Um **único comando** fez o que, no modo manual (doc [05](05-pitr.md)), exigiu copiar
o base backup, escrever `restore_command`, `recovery_target_time` e criar
`recovery.signal`. O pgBackRest gerou tudo isso sozinho.

## Ver o estado do repositório

```bash
pgbackrest --stanza=demo info
```

Saída real:

```
db (current)
    wal archive min/max (18): 000000010000000000000001/000000010000000000000005
    full backup: 20261111-204728F
        timestamp start/stop: 2026-11-11 20:47:28 / 20:47:37
        database size: 22.7MB, backup size: 22.7MB
        repo1: backup set size: 3MB
```

Repare que o `backup set` no repositório ficou em **3MB** (comprimido) para um banco
de 22.7MB — compressão embutida.

## Incremental e diferencial (testado) — a cadeia de dependência

Testamos os três tipos no mesmo repositório. O `info` mostra como **diff** e
**incr** se **encaixam** num full (o nome deles carrega o rótulo do full pai):

```
full backup: 20261111-205336F
    database backup size: 41.7MB   repo: 8.7MB        <- copiou o banco inteiro
diff backup: 20261111-205336F_20261111-205346D
    database backup size: 20.8MB   repo: 6.3MB        <- só o que mudou desde o FULL
incr backup: 20261111-205336F_20261111-205350I
    database backup size: 22.7MB   repo: 6.9MB        <- só o que mudou desde o backup anterior
```

- **Full** (`F`) — cópia completa; é a âncora.
- **Diferencial** (`D`) — tudo que mudou **desde o último full**. Cresce ao longo da
  semana, mas restaura rápido (full + 1 diff).
- **Incremental** (`I`) — tudo que mudou **desde o último backup** (full, diff ou
  incr). O menor de todos, mas restaurar exige a **cadeia inteira** (full + todos os
  incrementais até o ponto).

> No nosso lab o banco é pequeno e a mudança foi grande em proporção, então a
> economia pareceu modesta. **Em produção** (um banco de 500 GB que muda 2 GB por
> dia), um diff/incr diário ocupa **~2 GB** em vez de 500 GB — a economia é enorme.
> Estratégia clássica: **full semanal + incremental diário**.

> ⚠️ A cadeia importa no restore: perder o **full** (ou um incremental do meio)
> **inutiliza** os backups que dependem dele. Por isso retenção e verificação são
> críticas.

## Verificar a integridade

```bash
pgbackrest --stanza=demo verify        # confere o repositório pgBackRest
pg_verifybackup /caminho/basebackup    # confere um backup físico do pg_basebackup
```

Saída real do `pg_verifybackup` no nosso backup físico:

```
backup successfully verified
```

## Boas práticas com pgBackRest

- Rode `pgbackrest check` após configurar (valida o archiving de ponta a ponta).
- Combine **full semanal + diferencial/incremental diário** para economizar espaço.
- Use um **repositório remoto** (S3/NFS) para cumprir o 3-2-1 (doc [01]).
- Faça o backup a partir de uma **réplica** para não pesar no primary (doc [07]).
- Agende `pgbackrest verify` para checar a integridade do repositório.

## Lições

- pgBackRest automatiza backup físico + archiving + retenção + PITR + verificação.
- O pacote vem do repo **`pgdg-common`** (a pegadinha da instalação).
- `stanza-create` → `check` → `backup`; restore com `--type=time` faz PITR num comando.
- Compressão e backups incrementais economizam muito espaço.

## Pegadinhas

- Esquecer o `check` → descobrir que o archiving não funcionava **só no desastre**.
- `repo1-retention-full` baixo demais apaga fulls que o PITR ainda precisaria.
- Em cluster Patroni, configure o `archive_command` **pelo Patroni** e faça o backup
  da **réplica** — ver doc [07](07-backup-em-cluster-patroni.md).
