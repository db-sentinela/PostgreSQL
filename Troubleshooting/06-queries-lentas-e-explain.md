# 06 — Queries lentas e EXPLAIN

## O que é

Duas situações diferentes que confundem iniciantes:

- **Query longa** = está rodando **agora** há muito tempo (você acha no
  `pg_stat_activity`).
- **Query lenta** = é **intrinsecamente** ineficiente (faz varredura de tabela
  inteira); você diagnostica com `EXPLAIN`.

## Parte 1 — Achar e cancelar uma query longa

```sql
-- queries ativas há mais de 2 segundos:
SELECT pid, application_name, state,
       now() - query_start AS rodando_ha,
       left(query, 60) AS query
FROM pg_stat_activity
WHERE state = 'active' AND now() - query_start > interval '2 seconds'
ORDER BY rodando_ha DESC;
```

### Saída real

```
pid        | 2791
quem       | query_lenta
state      | active
rodando_ha | 00:00:03.01
query      | SELECT count(*) FROM generate_series(1,80000000) g WHER
```

Para **interromper** sem derrubar a conexão da pessoa, use o `cancel` (educado):

```sql
SELECT pg_cancel_backend(2791);
```

A sessão alvo recebe:

```
ERROR:  canceling statement due to user request
```

> Para o banco cancelar **sozinho** queries longas, configure
> `statement_timeout` (ex.: `SET statement_timeout = '30s'`). Pode ser por sessão,
> por usuário (`ALTER ROLE`) ou global (`ALTER SYSTEM`).

## Parte 2 — Por que a query é lenta? `EXPLAIN`

`EXPLAIN` mostra o **plano** que o PostgreSQL vai usar. `EXPLAIN (ANALYZE, BUFFERS)`
**executa** de verdade e mostra tempos e páginas lidas.

### Antes: sem índice → Seq Scan (varre tudo)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM tshoot.eventos WHERE payload = 'nao-existe';
```

```
 Gather  (actual time=13.383..15.622 rows=0 loops=1)
   Workers Launched: 2
   ->  Parallel Seq Scan on eventos  (actual time=10.248..10.248 rows=0 loops=3)
         Filter: (payload = 'nao-existe'::text)
         Rows Removed by Filter: 166667        <- leu e descartou 500k linhas!
         Buffers: shared hit=5155
 Execution Time: 15.656 ms
```

**Seq Scan** = "li a tabela inteira". `Rows Removed by Filter` enorme é o sinal
clássico de **falta de índice**.

### Depois: criando o índice → Index Scan

```sql
CREATE INDEX idx_eventos_payload ON tshoot.eventos(payload);
ANALYZE tshoot.eventos;
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM tshoot.eventos WHERE payload = 'nao-existe';
```

```
 Index Scan using idx_eventos_payload on eventos  (actual time=0.042..0.042 rows=0 loops=1)
   Index Cond: (payload = 'nao-existe'::text)
   Buffers: shared read=3
 Execution Time: 0.084 ms
```

**15,656 ms → 0,084 ms**: cerca de **180x mais rápido**. O **Index Scan** foi direto
ao ponto (3 páginas) em vez de varrer 5155.

## Como ler um plano (o básico que importa)

- Leia **de dentro para fora** (o nó mais indentado roda primeiro).
- **Seq Scan** em tabela grande + filtro seletivo → provável falta de índice.
- **Index Scan / Index Only Scan** → está usando índice (bom).
- `actual time` e `Execution Time` → o tempo real. `rows` esperado x real muito
  diferente → estatísticas desatualizadas (rode `ANALYZE`).
- `Buffers: shared hit` (cache) x `read` (disco) → muito `read` pode indicar dado
  frio ou falta de memória.

## Registrar queries lentas automaticamente

Não dá para ficar olhando ao vivo. Deixe o banco **logar** as lentas:

```sql
ALTER SYSTEM SET log_min_duration_statement = '500ms';  -- loga query > 500 ms
SELECT pg_reload_conf();
```

Aparece no log (`/var/lib/pgsql/18/data/log/*.log`) toda query acima do limite — a
forma mais prática de caçar lentidão em produção. (A extensão `pg_stat_statements`
dá um ranking agregado das piores queries; tema de **tuning**.)

## Lições

- "Query longa" (achar no `pg_stat_activity` + `cancel`) ≠ "query lenta" (diagnosticar
  com `EXPLAIN`).
- `EXPLAIN (ANALYZE, BUFFERS)` revela **Seq Scan x Index Scan**; índice certo muda
  ordens de grandeza.
- `statement_timeout` e `log_min_duration_statement` são seus aliados em produção.

## Pegadinhas

- `EXPLAIN ANALYZE` **executa** a query de verdade — cuidado com `UPDATE/DELETE`
  (eles acontecem!). Envolva em transação com `ROLLBACK` se precisar.
- Índice não é de graça: ocupa espaço e deixa a escrita um pouco mais lenta. Crie o
  que o `WHERE`/`JOIN` realmente usa, não "por via das dúvidas".
- Em tabela grande e em produção, use `CREATE INDEX CONCURRENTLY` para não travar
  escritas durante a criação.
