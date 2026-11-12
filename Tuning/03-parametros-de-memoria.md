# 03 — Parâmetros de memória

Os parâmetros de memória são o ajuste fino mais importante do `postgresql.conf`.
Entenda **o que cada um faz** antes de mexer.

## Os quatro que importam

| Parâmetro | Controla | Ponto de partida (regra prática) |
|-----------|----------|----------------------------------|
| **`shared_buffers`** | Cache de dados do PostgreSQL (compartilhado) | **25%** da RAM |
| **`effective_cache_size`** | **Estimativa** de cache total (SO + PG) — só informa o planejador | **50–75%** da RAM |
| **`work_mem`** | Memória **por operação** de sort/hash **por query** | 16–64 MB (cuidado — ver abaixo) |
| **`maintenance_work_mem`** | Memória para VACUUM, CREATE INDEX, etc. | 256 MB – 1 GB |

No nosso lab (8 GB RAM): `shared_buffers=512MB`, `effective_cache_size=1536MB`,
`work_mem=4MB`, `maintenance_work_mem=128MB`.

## shared_buffers

O cache de páginas do próprio PostgreSQL. ~25% da RAM é um bom começo. **Mais não é
sempre melhor**: o SO também faz cache, e valores altos demais podem até piorar.
Mudar exige **restart**.

## effective_cache_size

Não aloca nada! É só uma **dica** para o planejador de quanta memória de cache
(PostgreSQL + SO) existe no total. Valor alto faz o planejador **confiar mais em
índices** (acha que as páginas estarão em cache). ~50–75% da RAM.

## work_mem — o mais traiçoeiro

`work_mem` é a memória que **cada operação** de ordenação (`ORDER BY`, `DISTINCT`,
merge join) ou hash pode usar. Se a operação **não cabe** em `work_mem`, ela
**transborda para o disco** (temp files) — e fica lenta.

### Teste real: o mesmo sort, em memória x em disco

```sql
SET work_mem = '4MB';
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM tshoot.eventos ORDER BY md5(payload);
```
```
Sort Method: external merge  Disk: 45080kB     <-- transbordou 45 MB para o DISCO
```

```sql
SET work_mem = '256MB';
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM tshoot.eventos ORDER BY md5(payload);
```
```
Sort Method: quicksort  Memory: 63070kB        <-- coube na MEMÓRIA
```

A linha **`Sort Method`** é o indicador: `external merge Disk` = transbordou (ruim);
`quicksort Memory` = ficou em memória (bom). Procure por isso nos seus `EXPLAIN`.

> ⚠️ **A pegadinha mortal do work_mem:** ele é **por operação, por conexão** — não é
> global! Uma query complexa pode usar `work_mem` **várias vezes** (vários sorts/
> hashes). Com `work_mem=256MB`, `max_connections=200` e queries com 3 operações
> cada, o pico teórico é `256MB × 200 × 3 ≈ 150 GB`. É assim que se estoura a RAM e
> o servidor começa a usar swap (ou o OOM killer mata o Postgres).

### Como usar work_mem com segurança

- Mantenha o **global** modesto (ex.: 16 MB).
- **Suba por sessão** só para a query pesada que precisa:
  ```sql
  SET work_mem = '256MB';  -- só nesta conexão/relatório
  SELECT ... ORDER BY ...;
  RESET work_mem;
  ```
- Ou por usuário/role: `ALTER ROLE relatorios SET work_mem = '256MB';`

## maintenance_work_mem

Usado por operações de manutenção: `VACUUM`, `CREATE INDEX`, `ALTER TABLE`. Valor
maior = índices criados mais rápido e VACUUM mais eficiente. Pode ser generoso
(256 MB–1 GB) porque **poucas** rodam ao mesmo tempo.

## Como dimensionar (ponto de partida para 8 GB RAM)

```ini
shared_buffers = 2GB                 # 25%
effective_cache_size = 6GB           # 75%
work_mem = 16MB                      # global modesto; suba por sessão quando precisar
maintenance_work_mem = 512MB
```

> Ferramentas como **PGTune** dão um ponto de partida por hardware/carga. Use como
> **ponto de partida**, depois meça.

## Lições

- `shared_buffers` ~25% RAM; `effective_cache_size` ~50–75% (só informa o planejador).
- `work_mem` é **por operação, por conexão** — conservador no global, generoso por
  sessão. `Sort Method: external merge Disk` no EXPLAIN = suba o `work_mem`.
- `maintenance_work_mem` pode ser alto; acelera VACUUM e criação de índice.

## Pegadinhas

- Setar `work_mem` global muito alto → risco de **esgotar a RAM** sob concorrência.
- `shared_buffers` gigante nem sempre ajuda (o SO já faz cache); teste.
- Esquecer que mudar `shared_buffers` exige **restart** (os outros aceitam `reload`).
