# 01 — A caixa de ferramentas de diagnóstico

Estas são as visões que respondem "**o que está acontecendo agora?**". Decore 3 ou 4
e você já resolve a maioria dos incidentes.

## `pg_stat_activity` — a mais importante

Mostra **uma linha por conexão** ao banco. É o seu "gerenciador de tarefas".

```sql
SELECT pid, usename, application_name, state,
       wait_event_type, wait_event,
       now() - query_start AS rodando_ha,
       left(query, 60) AS query
FROM pg_stat_activity
WHERE state <> 'idle'          -- esconde conexões ociosas (sem transação aberta)
ORDER BY rodando_ha DESC NULLS LAST;
```

Colunas-chave:

- **state** — o estado da sessão:
  - `active` = executando uma query **agora**.
  - `idle` = conectada, **sem** transação aberta (tranquilo).
  - `idle in transaction` = abriu `BEGIN` e **parou** sem `COMMIT` — **perigoso**
    (segura locks; ver doc [05](05-idle-in-transaction.md)).
  - `idle in transaction (aborted)` = transação deu erro e não foi encerrada.
- **wait_event_type / wait_event** — **por que** está esperando. Ex.: `Lock` =
  esperando um bloqueio; `Client` = esperando o cliente mandar algo; `IO` = disco.
- **query_start** — quando a query atual começou. `now() - query_start` = há quanto
  tempo roda (ache as longas!).
- **xact_start** — quando a **transação** começou (útil para achar transações longas).
- **query** — o texto da query (atual, ou a última se estiver idle).

## `pg_locks` + `pg_blocking_pids()` — bloqueios

`pg_blocking_pids(pid)` é a função que entrega **quem está bloqueando** um PID. É o
atalho que todo DBA usa:

```sql
-- quem está bloqueado e por quem:
SELECT a.pid AS bloqueado, a.application_name,
       pg_blocking_pids(a.pid) AS bloqueado_pelos_pids,
       a.wait_event_type, left(a.query,50) AS query
FROM pg_stat_activity a
WHERE cardinality(pg_blocking_pids(a.pid)) > 0;
```

`pg_locks` é a visão crua de todos os bloqueios (mais detalhada, menos amigável):

```sql
SELECT locktype, relation::regclass, mode, granted, pid
FROM pg_locks WHERE relation = 'tshoot.contas'::regclass ORDER BY granted DESC;
```

Detalhes e exemplos reais nos docs [02](02-locks-e-bloqueios.md) e
[03](03-cadeia-de-bloqueios-e-como-matar.md).

## Os logs do PostgreSQL

Muita coisa só aparece no **log**: deadlocks, erros, conexões recusadas, e (se
configurado) queries lentas. No nosso ambiente (PGDG) ficam em:

```bash
/var/lib/pgsql/18/data/log/*.log
```

```sql
-- descubra o arquivo de log ativo (caminho relativo ao data_dir):
SELECT pg_current_logfile();
```

> Dica: para registrar **queries lentas** no log, use
> `log_min_duration_statement = '500ms'` (loga toda query acima de 500 ms). Ótimo
> para caçar lentidão sem ficar olhando ao vivo.

## Estatísticas de tabelas e índices

```sql
-- tabelas com muita leitura sequencial (candidatas a índice) e dead tuples:
SELECT relname, seq_scan, idx_scan, n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables ORDER BY seq_scan DESC;
```

- `seq_scan` alto + `idx_scan` baixo → pode faltar índice.
- `n_dead_tup` alto → bloat; o autovacuum pode estar atrás.

## Ação: cancelar ou matar uma sessão

```sql
SELECT pg_cancel_backend(<pid>);     -- cancela a QUERY atual (educado); mantém a conexão
SELECT pg_terminate_backend(<pid>);  -- derruba a CONEXÃO inteira (força); aborta a transação
```

> Prefira **sempre** `pg_cancel_backend` primeiro. Só use `terminate` quando cancel
> não resolve (ex.: `idle in transaction` — doc [05](05-idle-in-transaction.md)).

## Papel para enxergar tudo sem ser superusuário

```sql
GRANT pg_monitor TO seu_usuario_de_monitoramento;
```

`pg_monitor` deixa o usuário ver todas as sessões e estatísticas (sem dar
superusuário). Ideal para quem só observa.

## Lições

- `pg_stat_activity` responde "o que roda agora e por quê espera".
- `pg_blocking_pids()` responde "quem bloqueia quem" em uma linha.
- Deadlocks e queries lentas moram no **log**.
- `cancel` antes de `terminate`, sempre.
