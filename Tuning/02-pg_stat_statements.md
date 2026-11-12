# 02 — pg_stat_statements (a ferramenta nº 1 de tuning)

## O que é

`pg_stat_statements` é uma extensão que **acumula estatísticas de cada query** que o
banco executa: quantas vezes rodou, tempo total, tempo médio, linhas, uso de cache.
É assim que você descobre, com dados, **onde o banco gasta o tempo** — em vez de
adivinhar.

## Por que importa

Sem ela, "o banco está lento" é um chute. Com ela, você aponta o dedo para **a**
query responsável por 60% do tempo e otimiza o que realmente importa.

## Habilitar (exige restart — feito via Patroni)

É uma biblioteca de `shared_preload_libraries`, então precisa de **restart**. Em um
cluster Patroni, faça pelo `edit-config` (vale para o cluster):

```bash
patronictl -c /etc/patroni/patroni.yml edit-config pgha \
  -s postgresql.parameters.shared_preload_libraries=pg_stat_statements --force
# aplica com restart controlado (réplicas primeiro, depois o líder):
patronictl -c /etc/patroni/patroni.yml restart pgha db-03 --force
patronictl -c /etc/patroni/patroni.yml restart pgha db-01 --force
```

Depois, crie a extensão em cada banco onde quer consultá-la:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;   -- versão 1.12 no PG 18
```

> A extensão é **cluster-wide** na coleta: ela vê queries de **todos** os bancos.
> Você só precisa do `CREATE EXTENSION` no banco de onde vai **consultar**.

## Usar: achar as piores queries

```sql
-- zere para medir uma janela específica (ex.: antes de um teste de carga):
SELECT pg_stat_statements_reset();

-- ... deixe a aplicação rodar ...

-- TOP por tempo TOTAL (o que mais pesa no dia a dia):
SELECT substring(query,1,60) AS query, calls,
       round(total_exec_time::numeric,0) AS total_ms,
       round(mean_exec_time::numeric,3)  AS mean_ms, rows
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat_statements%'
ORDER BY total_exec_time DESC LIMIT 10;
```

## O que você vê (saída real — carga `pgbench`, 784 TPS)

```
query    | UPDATE pgbench_branches SET bbalance = bbalance + $1 WHERE ...
calls    | 11772
total_ms | 27297        <-- 27 segundos no total!
mean_ms  | 2.319        <-- e 2,3 ms por chamada (alto)
rows     | 11772

query    | UPDATE pgbench_tellers SET tbalance = tbalance + $1 WHERE ...
calls    | 11772 | total_ms | 4177 | mean_ms | 0.355
query    | UPDATE pgbench_accounts SET abalance = abalance + $1 WHERE ...
calls    | 11772 | total_ms | 1417 | mean_ms | 0.120
query    | SELECT abalance FROM pgbench_accounts WHERE aid = $1
calls    | 11772 | total_ms |  142 | mean_ms | 0.012
```

Leitura imediata: o `UPDATE pgbench_branches` é o **vilão** — domina o tempo total
**e** o médio. Por quê? No pgbench há **poucas** linhas de "branch", e todas as
transações atualizam as mesmas → **contenção de lock** naquelas linhas quentes.
(É o tipo de achado que só `pg_stat_statements` te dá: o problema não é "o banco",
é uma linha disputadíssima.)

## Colunas que valem ouro

- **`calls`** — quantas vezes rodou.
- **`total_exec_time`** — tempo total acumulado. **Ordene por aqui primeiro.**
- **`mean_exec_time`** — tempo médio por execução.
- **`rows`** — linhas processadas no total.
- **`shared_blks_hit` / `shared_blks_read`** — cache vs disco (muito `read` =
  dados frios ou falta de memória).
- A query aparece **normalizada** (valores viram `$1`, `$2`) — chamadas iguais com
  parâmetros diferentes somam na mesma linha.

## Padrões para reconhecer

| Sintoma no pg_stat_statements | Provável causa |
|-------------------------------|----------------|
| `mean_ms` alto, poucas `calls` | Query pesada isolada → `EXPLAIN` (doc 04) |
| `mean_ms` baixo, `calls` enorme | "Query N+1" / chamada em excesso pela app |
| `total_ms` domina tudo | A query que merece sua atenção primeiro |
| `shared_blks_read` alto | Falta índice ou memória; dados frios |

## Lições

- `pg_stat_statements` mostra, com dados, **onde o tempo vai**. Ligue em produção.
- Ordene por **tempo total** para achar o que mais pesa.
- Queries aparecem **normalizadas**; `reset()` delimita uma janela de medição.
- O resultado te leva direto ao próximo passo: `EXPLAIN` da query campeã (doc 04).

## Pegadinhas

- Tem um custo pequeno de overhead, mas o benefício compensa **muito** — mantenha ligada.
- `pg_stat_statements.max` (padrão 5000) limita quantas queries distintas ela guarda;
  em bancos com muitas queries únicas, aumente.
- Ela mede **tempo**, não I/O de disco do SO — combine com `pg_stat_io` (doc 07)
  para o quadro completo.
