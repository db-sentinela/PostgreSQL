# 09 — Cheat sheet de tuning

A cola do dia a dia de performance.

## Achar as piores queries (pg_stat_statements)

```sql
-- top por tempo total:
SELECT substring(query,1,60) AS query, calls,
       round(total_exec_time::numeric,0) AS total_ms,
       round(mean_exec_time::numeric,2)  AS mean_ms, rows
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat_statements%'
ORDER BY total_exec_time DESC LIMIT 15;

SELECT pg_stat_statements_reset();   -- zera a janela de medição
```

## Analisar uma query

```sql
EXPLAIN (ANALYZE, BUFFERS) <query>;   -- ANALYZE EXECUTA; cuidado com escrita
-- procurar: Seq Scan em tabela grande | Sort Method: external merge Disk
--           | estimativa de rows muito diferente do real
```

## Índices

```sql
-- faltando (muita leitura sequencial)?
SELECT relname, seq_scan, idx_scan, n_live_tup
FROM pg_stat_user_tables ORDER BY seq_scan DESC LIMIT 10;

-- inúteis (nunca usados)?
SELECT relname, indexrelname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS tamanho
FROM pg_stat_user_indexes WHERE idx_scan = 0 ORDER BY pg_relation_size(indexrelid) DESC;

-- criar sem travar escrita:
CREATE INDEX CONCURRENTLY idx_nome ON tabela (col1, col2);
CREATE INDEX idx_cobertura ON t (busca) INCLUDE (retorno);   -- covering -> index only
CREATE INDEX idx_parcial ON t (col) WHERE status = 'pendente'; -- parcial
```

## Memória por sessão (sort/hash em disco)

```sql
SET work_mem = '256MB';   -- só nesta conexão, para um relatório pesado
-- ... query ...
RESET work_mem;
ALTER ROLE relatorios SET work_mem = '256MB';   -- por usuário
```

## VACUUM e bloat

```sql
-- quem tem mais lixo?
SELECT relname, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;

VACUUM (VERBOSE, ANALYZE) tabela;          -- limpa + atualiza estatísticas
-- VACUUM FULL tabela;                     -- encolhe, mas TRAVA a tabela (janela!)

-- autovacuum mais agressivo em tabela grande:
ALTER TABLE tabela SET (autovacuum_vacuum_scale_factor = 0.02);

-- por que o VACUUM não limpa? (horizonte de xmin travado)
SELECT slot_name, active, xmin FROM pg_replication_slots;         -- slot atrasado?
SELECT pid, state, backend_xmin FROM pg_stat_activity
  WHERE backend_xmin IS NOT NULL ORDER BY age(backend_xmin) DESC; -- transação antiga?
```

## Checkpoints / I/O

```sql
SELECT * FROM pg_stat_checkpointer;   -- num_requested alto => suba max_wal_size
SELECT * FROM pg_stat_io WHERE writes > 0 ORDER BY writes DESC LIMIT 10;  -- PG 16+
```

## Estatísticas do planejador

```sql
ANALYZE tabela;                                    -- atualiza estatísticas
ALTER TABLE t ALTER COLUMN c SET STATISTICS 1000;  -- mais detalhe numa coluna difícil
```

## Aplicar parâmetros NUM CLUSTER PATRONI

> Não edite o `postgresql.conf` do nó — o Patroni sobrescreve. Use `edit-config`:

```bash
P="patronictl -c /etc/patroni/patroni.yml"
# parâmetro de 'reload' (vale já, sem restart):
$P edit-config pgha -s postgresql.parameters.work_mem=16MB --force
# parâmetro que exige restart (ex.: shared_buffers, shared_preload_libraries):
$P edit-config pgha -s postgresql.parameters.shared_buffers=2GB --force
$P restart pgha db-03 --force      # réplicas primeiro
$P restart pgha db-02 --force
$P restart pgha db-01 --force      # líder por último
```

## Pontos de partida de parâmetros (servidor dedicado, SSD)

```ini
# Memória (ajuste à RAM):
shared_buffers = 25% da RAM                 # restart
effective_cache_size = 75% da RAM
work_mem = 16MB                             # global; suba por sessão
maintenance_work_mem = 512MB

# I/O / checkpoints:
checkpoint_timeout = '15min'
max_wal_size = '8GB'
checkpoint_completion_target = 0.9
wal_compression = on
random_page_cost = 1.1                      # SSD/NVMe
effective_io_concurrency = 200              # SSD/NVMe

# Observabilidade:
shared_preload_libraries = 'pg_stat_statements,auto_explain'   # restart
auto_explain.log_min_duration = '1s'
log_min_duration_statement = '500ms'
log_checkpoints = on
```

## A ordem mental do tuning

1. `pg_stat_statements` → qual query pesa mais (tempo total).
2. `EXPLAIN (ANALYZE, BUFFERS)` → por que ela é lenta.
3. Índice / reescrita da query → o maior ganho.
4. `work_mem` se há sort/hash em disco.
5. `VACUUM`/autovacuum se há bloat.
6. Checkpoints/WAL se há picos de I/O.
7. Pooling se há conexões demais.

> **Meça. Mude uma coisa. Meça de novo.**
