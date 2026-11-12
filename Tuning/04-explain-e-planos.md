# 04 — EXPLAIN e planos de execução

`EXPLAIN` mostra o **plano** que o PostgreSQL vai usar para rodar uma query. Saber
lê-lo é a diferença entre "chutar índices" e **entender** por que a query é lenta.

## EXPLAIN x EXPLAIN ANALYZE

- **`EXPLAIN`** — mostra o plano **estimado** (não executa). Rápido e seguro.
- **`EXPLAIN (ANALYZE, BUFFERS)`** — **executa** a query e mostra tempos e páginas
  **reais**. É o que você quer para diagnóstico.

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

> ⚠️ `ANALYZE` **executa de verdade**. Com `INSERT/UPDATE/DELETE`, envolva em
> `BEGIN; ... ROLLBACK;` para não alterar dados.

## Como ler um plano

- Leia **de dentro para fora** e **de baixo para cima**: o nó mais indentado roda
  primeiro; os de cima consomem os de baixo.
- Cada nó mostra `(actual time=início..fim rows=N loops=L)`. O tempo de um nó
  **inclui** os filhos.
- Compare `rows` **estimado** com o **real**: diferença grande = estatísticas ruins
  (rode `ANALYZE`) → planos ruins.

## Tipos de acesso a tabela (scans)

| Nó | Significa | Quando é bom/ruim |
|----|-----------|-------------------|
| **Seq Scan** | Lê a tabela inteira | Ok em tabela pequena; **ruim** em tabela grande com filtro seletivo (falta índice) |
| **Index Scan** | Usa índice e vai ao heap buscar as linhas | Bom para poucas linhas |
| **Index Only Scan** | Responde **só com o índice** (não toca o heap) | Ótimo — exige índice covering + VACUUM (doc 05) |
| **Bitmap Heap/Index Scan** | Junta várias entradas do índice e lê o heap em lote | Bom para "médias" quantidades de linhas |

## Tipos de JOIN (e quando o planejador escolhe cada um)

### Nested Loop — testado
Para cada linha do lado **externo** (pequeno), busca no **interno** (geralmente por
índice). Ótimo quando o externo tem **poucas** linhas.

```sql
EXPLAIN (ANALYZE) SELECT c.nome, p.valor
FROM perf.pedidos p JOIN perf.clientes c ON c.id = p.cliente_id
WHERE c.id = 42;
```
```
Nested Loop
  -> Index Scan using clientes_pkey on clientes  (rows=1)
  -> Index Only Scan using idx_ped_cli_inc on pedidos  (rows=98)
Execution Time: 0.140 ms
```
Um cliente só → 1 linha externa → nested loop com índice é imbatível (0,14 ms).

### Hash Join — testado
Monta uma **tabela hash** do lado menor e "sonda" com o maior. Ótimo para juntar
**muitas** linhas.

```sql
EXPLAIN (ANALYZE) SELECT c.nome, count(*)
FROM perf.pedidos p JOIN perf.clientes c ON c.id = p.cliente_id
GROUP BY c.nome;
```
```
Hash Join  (rows=333316)
  Hash Cond: (p.cliente_id = c.id)
  -> Parallel Seq Scan on pedidos  (rows=333333)
  -> Hash -> Seq Scan on clientes  (rows=10000)     <- hash da tabela pequena
Execution Time: 134 ms
```
Todas as linhas → hash join varrendo tudo é o certo (134 ms para 1 M linhas).

> **Merge Join** é o terceiro tipo: junta duas entradas **já ordenadas**. Comum
> quando ambos os lados vêm ordenados por índice.

## O que procurar (sinais de problema)

- **Seq Scan** em tabela grande com filtro seletivo → falta índice (doc 05).
- **`Rows Removed by Filter`** alto → leu muita coisa para jogar fora.
- **`Sort Method: external merge Disk`** → `work_mem` baixo (doc 03).
- **Estimativa vs real** muito diferente → `ANALYZE` / mais estatísticas.
- **`Nested Loop`** com milhares de loops no lado interno sem índice → join ruim.

## auto_explain: logar planos automaticamente

Para capturar o plano das queries lentas **em produção** sem rodar EXPLAIN na mão:

```ini
# via patronictl edit-config
shared_preload_libraries = 'pg_stat_statements,auto_explain'
auto_explain.log_min_duration = '1s'      # loga o plano de toda query > 1s
auto_explain.log_analyze = on
```

## Lições

- `EXPLAIN (ANALYZE, BUFFERS)` é o raio-X de uma query. Leia de dentro para fora.
- Seq Scan em tabela grande + filtro seletivo = candidato a índice.
- Nested Loop para poucas linhas; Hash Join para muitas — o planejador decide pelas
  **estatísticas**.
- `auto_explain` captura planos lentos automaticamente.

## Pegadinhas

- `EXPLAIN ANALYZE` executa a query (cuidado com escrita).
- Estimativas erradas de linhas quase sempre = **estatísticas desatualizadas**.
- Um plano "feio" (Seq Scan) pode ser o **certo** para tabela pequena — não crie
  índice sem necessidade.
