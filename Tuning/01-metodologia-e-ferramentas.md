# 01 — Metodologia e ferramentas

## A mentalidade certa

Tuning **não** é decorar valores mágicos de parâmetros. É um **método**:

1. **Meça** onde está o gargalo (não suponha).
2. **Mude uma coisa** de cada vez.
3. **Meça de novo** e compare.
4. Repita.

> 80% dos ganhos de performance vêm de **queries e índices**, não de parâmetros do
> `postgresql.conf`. Comece pela query mais cara, não pelo `shared_buffers`.

## A pirâmide: onde os ganhos moram

```
        ┌───────────────────────────┐
        │  Query + índice certo      │  <- MAIOR impacto (docs 02, 04, 05)
        ├───────────────────────────┤
        │  Modelo de dados / schema  │
        ├───────────────────────────┤
        │  Parâmetros (memória, WAL) │  <- ajuste fino (docs 03, 07)
        ├───────────────────────────┤
        │  Hardware / SO             │  <- último recurso
        └───────────────────────────┘
```

## As ferramentas essenciais

| Ferramenta | Responde | Doc |
|------------|----------|-----|
| **`pg_stat_statements`** | "Quais queries consomem mais tempo no total?" | [02](02-pg_stat_statements.md) |
| **`EXPLAIN (ANALYZE, BUFFERS)`** | "Por que **esta** query é lenta?" | [04](04-explain-e-planos.md) |
| **`pg_stat_activity`** | "O que está rodando/esperando agora?" | `../Troubleshooting/` |
| **`pg_stat_user_tables` / `_indexes`** | "Falta índice? Tem índice inútil? Tem bloat?" | [05](05-indices.md), [06](06-vacuum-autovacuum-bloat.md) |
| **`pg_stat_io`** (PG 16+) | "Para onde vai o meu I/O?" | [07](07-checkpoints-wal-e-io.md) |
| **`auto_explain`** | "Logar o plano das queries lentas automaticamente" | [04](04-explain-e-planos.md) |

## O fluxo típico de uma investigação de lentidão

```sql
-- 1) Quais queries pesam mais (tempo total acumulado)?
SELECT substring(query,1,60), calls,
       round(total_exec_time::numeric,0) AS total_ms,
       round(mean_exec_time::numeric,2)  AS mean_ms
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 10;

-- 2) Pegue a pior e entenda o plano:
EXPLAIN (ANALYZE, BUFFERS) <a query campeã>;

-- 3) Procure Seq Scan em tabela grande, Sort/Hash em disco, estimativas erradas.
-- 4) Aja: índice, reescrita da query, work_mem, estatísticas (ANALYZE).
```

## "total_time" x "mean_time" — não confunda

- **`total_exec_time`** (tempo total) = `mean × calls`. Uma query de 1 ms chamada
  1 milhão de vezes **pesa mais** que uma de 2 s chamada 3 vezes. Comece por aqui.
- **`mean_exec_time`** (tempo médio) = quão lenta ela é **por execução**. Boa para
  achar a query individualmente mais pesada.

> Otimizar a query de **maior tempo total** costuma dar o maior retorno — é onde o
> banco realmente gasta o dia.

## Estatísticas: o combustível do planejador

O PostgreSQL escolhe planos com base em **estatísticas** das tabelas (coletadas pelo
`ANALYZE` / autovacuum). Estatística velha = plano ruim. Se um `EXPLAIN ANALYZE`
mostra estimativa de linhas **muito** diferente do real, rode:

```sql
ANALYZE minha_tabela;
-- para colunas com distribuição complexa, aumente o detalhe:
ALTER TABLE t ALTER COLUMN c SET STATISTICS 1000;  ANALYZE t;
```

## Lições

- Tuning é **método** (medir → mudar 1 coisa → medir), não decoreba.
- O maior impacto está em **query + índice**, não em parâmetros.
- `pg_stat_statements` e `EXPLAIN` são suas duas ferramentas principais.
- Priorize por **tempo total**; mantenha **estatísticas** atualizadas.

## Pegadinhas

- Mudar vários parâmetros de uma vez: você não saberá qual ajudou (ou piorou).
- "Copiar config de um blog": o que serve para um banco de 1 TB pode destruir um de
  4 GB. Dimensione para **o seu** hardware e carga.
- Otimizar a query de maior **tempo médio** ignorando que ela roda 2x/dia: foco errado.
