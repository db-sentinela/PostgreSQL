# Performance & Tuning no PostgreSQL

Base de conhecimento **didática** sobre deixar o PostgreSQL **rápido** — e, mais
importante, sobre **medir antes de mexer**. Cobre a metodologia, as ferramentas
(`pg_stat_statements`, `EXPLAIN`), parâmetros de memória, índices, `VACUUM`/bloat,
checkpoints/WAL e conexões. Tudo com cenários **testados de verdade** no laboratório.

> Laboratório: cluster `pgha` (3 nós Fedora 42, PostgreSQL 18.4, 8 GB RAM, 4 vCPU).
> Carga gerada com `pgbench`; objetos de teste no schema `perf`.

## A regra de ouro do tuning

> **Meça. Não chute.** O maior erro de iniciante é mudar `shared_buffers` "porque
> leu num blog". Tuning é: (1) **medir** onde dói, (2) mudar **uma** coisa, (3)
> **medir de novo**. Sem medição, você está adivinhando.

## Índice

| # | Documento | O que você aprende |
|---|-----------|--------------------|
| 01 | [Metodologia e ferramentas](01-metodologia-e-ferramentas.md) | Como abordar tuning; quais views olhar primeiro |
| 02 | [pg_stat_statements](02-pg_stat_statements.md) | Achar **as piores queries** do banco (a ferramenta nº 1) |
| 03 | [Parâmetros de memória](03-parametros-de-memoria.md) | `shared_buffers`, `work_mem`, etc.; sort em memória x disco (testado) |
| 04 | [EXPLAIN e planos](04-explain-e-planos.md) | Ler planos: scans, joins (hash x nested loop), custos, buffers |
| 05 | [Índices](05-indices.md) | Composto, parcial, covering, index-only, e achar índices inúteis (testado) |
| 06 | [VACUUM, autovacuum e bloat](06-vacuum-autovacuum-bloat.md) | Dead tuples, bloat, e por que o VACUUM às vezes **não** limpa (testado) |
| 07 | [Checkpoints, WAL e I/O](07-checkpoints-wal-e-io.md) | Suavizar picos de escrita em disco |
| 08 | [Conexões e pooling](08-conexoes-e-pooling.md) | O custo de cada conexão; quando usar PgBouncer |
| 09 | [Cheat sheet de tuning](09-cheatsheet-tuning.md) | Parâmetros + como aplicar num cluster Patroni |

## O caminho de investigação

```
"Está lento"
   │
   ├─ 1. Qual query? ............. pg_stat_statements (doc 02)
   ├─ 2. Por que ela é lenta? .... EXPLAIN (ANALYZE, BUFFERS) (doc 04)
   ├─ 3. Falta índice? .......... doc 05
   ├─ 4. Sort/hash em disco? .... work_mem (doc 03)
   ├─ 5. Tabela inchada? ........ VACUUM/bloat (doc 06)
   └─ 6. Picos de I/O? .......... checkpoints/WAL (doc 07)
```

## Como aplicar parâmetros neste ambiente (Patroni)

Em um cluster Patroni, **não** edite o `postgresql.conf` do nó na mão. Use:

```bash
patronictl -c /etc/patroni/patroni.yml edit-config pgha -s postgresql.parameters.work_mem=16MB --force
```

Assim o parâmetro vale para **todo o cluster** e sobrevive a failover. Detalhes no
doc [09](09-cheatsheet-tuning.md).

> Veja também os sub-repositórios irmãos: `../Patroni/` (HA),
> `../Troubleshooting/` e `../Backup&Restore/`.
