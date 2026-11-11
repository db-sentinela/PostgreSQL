# 02 — Backup lógico (pg_dump / pg_restore)

## O que é

Backup lógico gera um arquivo com **comandos SQL** (ou dados) capazes de **recriar**
os objetos e o conteúdo do banco. É granular e portável — ideal para copiar **um
objeto**, migrar, ou restaurar **uma tabela** específica.

## As ferramentas

- **`pg_dump`** — faz dump de **um banco** (ou objetos dele).
- **`pg_dumpall`** — dump de **todos os bancos** + os objetos **globais** (roles,
  tablespaces). Essencial: `pg_dump` **não** inclui roles/permissões globais.
- **`pg_restore`** — restaura dumps nos formatos **custom** e **directory**.
- Dump no formato **plain** (SQL puro) se restaura com `psql -f`.

## Formatos (escolha importa)

| Formato | Flag | Restaura com | Bom para |
|---------|------|--------------|----------|
| custom | `-Fc` | `pg_restore` | Padrão: comprimido, seletivo, 1 arquivo |
| directory | `-Fd` | `pg_restore` | **Paralelo** (`-j`), bancos grandes |
| plain | `-Fp` | `psql -f` | Ler/editar o SQL; simples |
| tar | `-Ft` | `pg_restore` | Arquivo único não comprimido |

> Prefira **custom** (`-Fc`) ou **directory** (`-Fd`): permitem restore **seletivo**
> (uma tabela) e paralelo. O plain (`-Fp`) é "tudo ou nada" com `psql`.

## Como reproduzir (banco de exemplo `lojadb`)

```bash
PG=/usr/pgsql-18/bin; BK=/var/lib/pgsql/backups

# custom (comprimido, 1 arquivo):
$PG/pg_dump -Fc -d lojadb -f $BK/lojadb.dump
# directory, paralelo com 2 workers:
$PG/pg_dump -Fd -j2 -d lojadb -f $BK/lojadb_dir
# plain SQL:
$PG/pg_dump -Fp -d lojadb -f $BK/lojadb.sql
# globais (roles) — rode sempre junto:
$PG/pg_dumpall --globals-only -f $BK/globals.sql
```

### O que você vê (saída real)

Tamanhos dos mesmos dados em cada formato (1000 clientes + 5000 pedidos):

```
4.0K  globals.sql
52K   lojadb.dump     <- custom (comprimido)
56K   lojadb_dir      <- directory
272K  lojadb.sql      <- plain (SQL, maior)
```

Dá para **inspecionar o conteúdo** de um dump custom sem restaurar, com
`pg_restore --list` (o "índice"/TOC do backup):

```
220; TABLE public clientes postgres
222; TABLE public pedidos postgres
3500; TABLE DATA public clientes postgres
3502; TABLE DATA public pedidos postgres
3348; CONSTRAINT public clientes clientes_pkey postgres
```

## Restore 1 — o banco inteiro

```bash
# apaga e recria o banco a partir do dump custom (-C cria o banco):
$PG/dropdb lojadb
$PG/pg_restore -C -d postgres $BK/lojadb.dump
```

### Saída real

```
-> banco lojadb APAGADO. Restaurando com: pg_restore -C -d postgres lojadb.dump
-> conferindo:
clientes=1000
pedidos=5000
```

O banco voltou **idêntico** (1000 / 5000).

## Restore 2 — só UMA tabela (a grande vantagem do lógico)

Perdeu só a tabela `pedidos`? Restaure **apenas ela**, sem tocar no resto:

```bash
$PG/pg_restore -d lojadb -t pedidos $BK/lojadb.dump
```

### Saída real

```
-> tabela pedidos dropada. Existe? f
-> restaurando SO a tabela pedidos: pg_restore -d lojadb -t pedidos lojadb.dump
pedidos_restaurada=5000
```

Isso é o que o backup **físico não faz**: granularidade de um objeto.

## Restore 3 — formato plain (com psql)

```bash
$PG/psql -d loja_copia -f $BK/lojadb.sql      # copia_clientes=1000
```

## Boas práticas

- Rode `pg_dump` **e** `pg_dumpall --globals-only` juntos — senão você restaura os
  dados mas **perde os usuários/permissões**.
- Para bancos grandes, use `-Fd -j N` (paralelo) no dump **e** `pg_restore -j N`.
- Verifique o backup: `pg_restore --list` lê o TOC sem restaurar.
- Agende (cron) e **teste o restore** periodicamente.

## Lições

- Lógico = granular e portável; restaura **um objeto**, um schema ou o banco.
- `-Fc`/`-Fd` permitem restore **seletivo** e **paralelo**; `-Fp` é SQL puro.
- **Sempre** acompanhe de `pg_dumpall --globals-only` (roles).

## Pegadinhas

- `pg_dump` **não** salva roles, tablespaces nem outros bancos — use `pg_dumpall`
  para o que é global.
- Restaurar um dump **grande** é lento (recria índices e valida constraints) — não é
  a melhor opção para RTO baixo; aí entra o backup **físico**.
- `pg_restore -t tabela` restaura a tabela e seus dados, mas cuidado com a **ordem**
  de dependências (FKs) ao restaurar objetos isolados.
