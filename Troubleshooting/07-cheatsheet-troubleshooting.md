# 07 — Cheat sheet de troubleshooting

A cola para colar na parede. Todas as queries rodam no **primary** (os locks de
escrita só existem lá).

## Ver o que está rodando

```sql
-- tudo que não está ocioso, do mais antigo para o mais novo:
SELECT pid, usename, application_name, state,
       wait_event_type, wait_event,
       now() - query_start AS rodando_ha,
       left(query, 60) AS query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY rodando_ha DESC NULLS LAST;
```

## Quem bloqueia quem

```sql
-- bloqueados e seus bloqueadores:
SELECT a.pid AS bloqueado, a.application_name, a.wait_event_type,
       pg_blocking_pids(a.pid) AS bloqueado_pelos_pids,
       left(a.query,50) AS query
FROM pg_stat_activity a
WHERE cardinality(pg_blocking_pids(a.pid)) > 0;
```

```sql
-- o TOPO da cadeia (quem bloqueia mas não é bloqueado):
SELECT DISTINCT b AS pid_topo
FROM pg_stat_activity a, LATERAL unnest(pg_blocking_pids(a.pid)) AS b
WHERE cardinality(pg_blocking_pids(a.pid)) > 0
  AND cardinality(pg_blocking_pids(b)) = 0;
```

## Transações perigosas

```sql
-- sessões 'idle in transaction' (segurando locks sem fazer nada):
SELECT pid, application_name, now() - xact_start AS tx_aberta_ha, left(query,50) AS ultima
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;

-- transações mais longas (ativas ou não):
SELECT pid, state, now() - xact_start AS tx_aberta_ha
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start LIMIT 10;
```

## Agir sobre uma sessão

```sql
SELECT pg_cancel_backend(<pid>);     -- cancela a QUERY (educado) -> tente primeiro
SELECT pg_terminate_backend(<pid>);  -- derruba a CONEXÃO (força) -> p/ idle in transaction
```

## Locks crus de um objeto

```sql
SELECT locktype, mode, granted, pid
FROM pg_locks WHERE relation = 'tshoot.contas'::regclass
ORDER BY granted DESC;
```

## Deadlock (onde achar)

```sql
SELECT pg_current_logfile();         -- arquivo de log atual
-- no shell:
--   grep -A14 'deadlock detected' /var/lib/pgsql/18/data/log/*.log
```

## Analisar uma query

```sql
EXPLAIN (ANALYZE, BUFFERS) <sua_query>;   -- cuidado: ANALYZE executa de verdade!
```

## Estatísticas úteis

```sql
-- tabelas com muita varredura sequencial e/ou dead tuples:
SELECT relname, seq_scan, idx_scan, n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables ORDER BY seq_scan DESC;

-- tamanho das tabelas:
SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) AS tamanho
FROM pg_stat_user_tables ORDER BY pg_total_relation_size(relid) DESC LIMIT 10;
```

## Defesas para ligar em produção (`ALTER SYSTEM` + `pg_reload_conf()`)

```sql
ALTER SYSTEM SET log_min_duration_statement = '500ms';          -- loga queries lentas
ALTER SYSTEM SET idle_in_transaction_session_timeout = '5min';  -- mata idle-in-tx
ALTER SYSTEM SET statement_timeout = '0';     -- (defina por app/role, não global cego)
ALTER SYSTEM SET lock_timeout = '5s';         -- desiste de esperar lock após 5s
SELECT pg_reload_conf();
```

> Em cluster Patroni, prefira alterar parâmetros pelo
> `patronictl edit-config` (eles valem para todo o cluster e sobrevivem a failover),
> em vez de `ALTER SYSTEM` em um nó só.

## Sistema operacional (quando não é o banco)

```bash
htop                 # CPU e memória
iostat -x 2          # I/O de disco (veja %util perto de 100%)
df -h                # disco cheio? (data_dir sem espaço = banco para)
journalctl -u patroni -n 50
tail -f /var/lib/pgsql/18/data/log/*.log
```

## A ordem mental (resuma tudo)

1. `pg_stat_activity` → o que roda e o que espera.
2. `pg_blocking_pids` → quem bloqueia quem (ache o topo).
3. Log do PostgreSQL → deadlocks, erros, queries lentas.
4. SO (CPU/disco/memória) → se não for o banco.
5. **Aja** com `cancel` antes de `terminate`.
