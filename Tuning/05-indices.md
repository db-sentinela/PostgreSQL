# 05 — Índices

Índice é a alavanca de performance mais poderosa — e a mais mal usada. Este doc
mostra os tipos que importam, **com testes reais** numa tabela de 1 milhão de linhas
(`perf.pedidos`: `cliente_id`, `status`, `valor`, `criado`).

## O básico: índice tira você do Seq Scan

```sql
EXPLAIN (ANALYZE) SELECT * FROM perf.pedidos WHERE cliente_id=42 AND status='pago';
```

**Sem índice** → varre a tabela inteira:
```
Parallel Seq Scan on pedidos   Execution Time: 26.786 ms
```

**Com índice** → vai direto:
```
Bitmap Index Scan on idx_ped_cli_status  (Index Cond: cliente_id=42 AND status='pago')
Execution Time: 0.417 ms        <-- ~64x mais rápido
```

## Índice composto (multi-coluna) — testado

```sql
CREATE INDEX idx_ped_cli_status ON perf.pedidos (cliente_id, status);
```

Serve para filtros que usam **as duas** colunas (como acima). **A ordem das colunas
importa**: `(cliente_id, status)` atende buscas por `cliente_id` sozinho e por
`cliente_id + status`, mas **não** por `status` sozinho.

> Regra: ponha primeiro a coluna usada em **igualdade** e/ou mais **seletiva**; por
> último as de range (`>`, `<`, `BETWEEN`).

## Índice covering (INCLUDE) → Index Only Scan — testado

Se o índice contém **todas** as colunas que a query pede, o PostgreSQL responde **só
com o índice**, sem tocar na tabela (heap). É o **Index Only Scan**.

```sql
CREATE INDEX idx_ped_cli_inc ON perf.pedidos (cliente_id) INCLUDE (valor);
VACUUM perf.pedidos;   -- necessário! (ver pegadinha)
EXPLAIN (ANALYZE) SELECT cliente_id, valor FROM perf.pedidos WHERE cliente_id=42;
```
```
Index Only Scan using idx_ped_cli_inc on pedidos
  Heap Fetches: 0            <-- NÃO tocou a tabela!
Execution Time: 0.073 ms
```

> ⚠️ **Pegadinha testada:** antes do `VACUUM`, a **mesma** query **não** fez Index
> Only Scan (fez Bitmap Heap Scan, tocando o heap). O Index Only Scan depende do
> **mapa de visibilidade**, que só é marcado pelo `VACUUM`. Tabela recém-escrita →
> VACUUM antes de esperar index-only. `Heap Fetches: 0` confirma que funcionou.

## Índice parcial (WHERE) — testado

Indexa **só as linhas que importam**. Ótimo para valores raros.

```sql
-- só indexa os pedidos 'pendente' (raros):
CREATE INDEX idx_ped_pendente ON perf.pedidos (cliente_id) WHERE status='pendente';
```

Tamanhos reais dos índices da mesma tabela:
```
idx_ped_cli_inc    | 30 MB      (covering, inclui valor)
pedidos_pkey       | 21 MB
idx_ped_cli_status | 7480 kB
idx_ped_criado     | 7040 kB
idx_ped_pendente   | 1352 kB    <-- PARCIAL: minúsculo (só as linhas 'pendente')
```

O índice parcial é **22x menor** que o covering — menos disco, menos manutenção,
e ainda assim resolve a busca por `status='pendente'`:
```
Bitmap Index Scan on idx_ped_pendente   Execution Time: 0.179 ms
```

## Achar índices INÚTEIS (que só pesam) — testado

Índice não é de graça: ocupa disco e **deixa a escrita mais lenta** (todo
INSERT/UPDATE atualiza os índices). Índice que **ninguém usa** é puro prejuízo.

```sql
SELECT indexrelname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS tamanho
FROM pg_stat_user_indexes
WHERE relname='pedidos'
ORDER BY idx_scan;
```
```
idx_ped_criado   | 0    <-- NUNCA foi usado: candidato a DROP
idx_ped_pendente | 0
idx_ped_cli_status | 1
```

`idx_scan = 0` (por um bom período em produção) = índice candidato a **remover**.

> Cuidado: `idx_scan` acumula desde o último `pg_stat_reset()`. Avalie ao longo de
> dias/semanas, e não remova índices de **constraint** (PK/unique).

## Tipos de índice (além do B-tree)

| Tipo | Para que serve |
|------|----------------|
| **B-tree** (padrão) | Igualdade e range (`=`, `<`, `>`, `BETWEEN`, `ORDER BY`) |
| **Hash** | Só igualdade (`=`) |
| **GIN** | Busca em `jsonb`, arrays, full-text |
| **GiST / SP-GiST** | Dados geométricos, ranges, vizinhança |
| **BRIN** | Tabelas enormes **naturalmente ordenadas** (ex.: por data) — índice minúsculo |

## Lições

- Índice certo troca **Seq Scan** por acesso direto (viramos 26 ms em 0,4 ms).
- **Composto**: ordem das colunas importa (igualdade/seletiva primeiro).
- **Covering (INCLUDE)** → Index Only Scan, mas precisa de **VACUUM** (mapa de
  visibilidade).
- **Parcial**: índice minúsculo para valores raros.
- `idx_scan = 0` → índice inútil, **remova** (cada índice pesa na escrita).

## Pegadinhas

- Criar índice em tabela grande em produção **trava escritas** — use
  `CREATE INDEX CONCURRENTLY`.
- Índice demais = escrita lenta e mais bloat. Menos (e certos) é mais.
- `status = 'pago'` (80% das linhas) **não** se beneficia de índice — o planejador
  prefere Seq Scan. Índice só ajuda o que é **seletivo**.
