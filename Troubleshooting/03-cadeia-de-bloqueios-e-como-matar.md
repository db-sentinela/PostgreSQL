# 03 — Cadeia de bloqueios e como resolver

## O que é

Bloqueios podem se **encadear**: A segura um lock, B espera A, e C espera B. Vira uma
**fila** (A → B → C). Isso é comum em produção e assusta porque "tudo parou" — mas a
causa é **um único** culpado no topo da fila.

## Por que importa

Se você matar o nó **errado** da fila, não resolve. A regra é: **ache e resolva o
topo da cadeia** (quem não está esperando ninguém). Mate o topo e a fila inteira
anda.

## Como reproduzir

Três sessões, travando em ordem:

- **A**: `BEGIN; UPDATE linha 1;` (segura a linha 1)
- **B**: `BEGIN; UPDATE linha 2; UPDATE linha 1;` → segura a 2, **espera** a 1 (em A)
- **C**: `BEGIN; UPDATE linha 2;` → **espera** a 2 (em B)

Observador:
```sql
SELECT a.application_name AS sessao, a.pid, a.wait_event_type, a.state,
       pg_blocking_pids(a.pid) AS bloqueado_pelos_pids
FROM pg_stat_activity a
WHERE a.application_name LIKE 'sessao_%' ORDER BY a.application_name;
```

## O que você vê (saída real)

```
  sessao  | pid  | wait_event_type | state  | bloqueado_pelos_pids
----------+------+-----------------+--------+----------------------
 sessao_A | 2230 | Timeout         | active | {}          <- TOPO: não espera ninguém
 sessao_B | 2237 | Lock            | active | {2230}      <- espera A
 sessao_C | 2244 | Lock            | active | {2237}      <- espera B
```

Leitura imediata: a fila é **C → B → A**. O **topo** é `sessao_A` (pid 2230),
porque `bloqueado_pelos_pids` dele está **vazio** (`{}`).

## Resolvendo: mate o topo

Duas funções para agir sobre uma sessão:

| Função | O que faz | Quando usar |
|--------|-----------|-------------|
| `pg_cancel_backend(pid)` | Cancela só a **query atual**; mantém a conexão | Primeira opção (educada) |
| `pg_terminate_backend(pid)` | Derruba a **conexão inteira**; aborta a transação | Quando cancel não resolve |

```sql
-- descubra o topo (quem não está bloqueado por ninguém) e encerre-o:
SELECT pg_terminate_backend(2230);
```

### O que acontece (saída real)

```
## matar o TOPO da cadeia: pg_terminate_backend(2230)
 matou_A = t

## DEPOIS de matar A (a cadeia se desfaz)
(0 rows)
```

Ao encerrar A, **a fila inteira destravou**: B conseguiu a linha 1 e terminou; C
conseguiu a linha 2 e terminou. Resolver o topo resolve tudo.

## Cancel x Terminate — a regra

1. **Tente `pg_cancel_backend` primeiro.** Ele é "educado": cancela a query, mas a
   conexão e a transação da pessoa continuam (ela pode dar `ROLLBACK`).
2. **Use `pg_terminate_backend` quando cancel não basta** — por exemplo, quando o
   bloqueador está **`idle in transaction`** (não tem query ativa para cancelar;
   ver doc [05](05-idle-in-transaction.md)).

## Como achar o topo de qualquer cadeia (query pronta)

```sql
-- lista só os bloqueadores que NÃO estão bloqueados (os "topos"):
SELECT DISTINCT b AS pid_topo
FROM pg_stat_activity a,
     LATERAL unnest(pg_blocking_pids(a.pid)) AS b
WHERE cardinality(pg_blocking_pids(a.pid)) > 0
  AND cardinality(pg_blocking_pids(b)) = 0;
```

## Lições

- Bloqueios se encadeiam; `pg_blocking_pids` revela a fila inteira.
- O **topo** é quem tem `bloqueado_pelos_pids` vazio. Resolva ele primeiro.
- `cancel` (educado) antes de `terminate` (força).

## Pegadinhas

- Matar o **meio** da fila (B) não adianta: ele está preso em A. Vá no topo.
- Em produção, antes de `terminate`, cheque **o que** a sessão do topo é: pode ser um
  job legítimo preso, não o vilão. Encerrar aborta o trabalho dela.
