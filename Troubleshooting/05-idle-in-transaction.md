# 05 — Idle in transaction (o assassino silencioso)

## O que é

`idle in transaction` é uma sessão que fez `BEGIN`, executou algo (e **pegou locks**)
e **parou** — sem `COMMIT` nem `ROLLBACK`. Fica ali, parada, **segurando os locks**.
É uma das causas mais comuns de "o sistema travou do nada".

## Por que importa

Diferente de uma query lenta (que ao menos está trabalhando), a sessão
`idle in transaction` **não faz nada** — mas **segura locks** e **impede o autovacuum**
de limpar linhas mortas. Costuma vir de **bug de aplicação**: um `BEGIN` sem o
`COMMIT` correspondente, ou um pool de conexões mal configurado.

## Como reconhecer

No `pg_stat_activity`, o `state` é literalmente **`idle in transaction`**:

```sql
SELECT pid, application_name, state,
       now() - state_change AS parada_ha,
       now() - xact_start  AS transacao_aberta_ha,
       left(query,50) AS ultima_query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;
```

### Saída real

```
pid          | 2627
quem         | sessao_idle
state        | idle in transaction
parada_ha    | 00:00:02.03
tx_aberta_ha | 00:00:02.04
ultima_query | UPDATE tshoot.contas SET saldo=saldo-500 WHERE id=1
```

Note `wait_event_type = Client`: ela está esperando o **cliente** mandar o próximo
comando — que nunca vem.

## O perigo na prática

Enquanto ela segura o lock, qualquer um que queira a mesma linha **trava**:

```
   sessao    | wait_event_type | bloqueado_por
-------------+-----------------+---------------
 sessao_B    | Lock            | {2627}        <- vítima presa na sessão idle
 sessao_idle | Client          | {}            <- a culpada, parada
```

## A pegadinha nº 1: `cancel` NÃO resolve

Como a sessão `idle in transaction` **não tem query ativa**, não há o que cancelar.
Veja o teste real — `pg_cancel_backend` retorna `t`, mas **nada muda**:

```
## pg_cancel_backend(2627) -> retorna t, porém...
   sessao    |        state        | bloqueado_por
-------------+---------------------+---------------
 sessao_B    | active              | {2627}         <- B CONTINUA bloqueado
 sessao_idle | idle in transaction | {}             <- a idle continua lá
```

Só `pg_terminate_backend` (que **derruba a conexão** e aborta a transação) libera:

```
## pg_terminate_backend(2627)
 pg_terminate_backend = t

## depois: B destrava e some
(0 rows)
```

## A defesa: timeouts automáticos

Não dependa de achar essas sessões na mão. Configure o banco para **matar sozinho**
transações ociosas:

```sql
-- encerra sessões paradas EM TRANSAÇÃO há mais de 5 min:
ALTER SYSTEM SET idle_in_transaction_session_timeout = '5min';

-- (relacionado) limita quanto uma transação pode durar no total:
ALTER SYSTEM SET transaction_timeout = '15min';   -- PostgreSQL 17+

SELECT pg_reload_conf();
```

- `idle_in_transaction_session_timeout` → mata quem fica **ocioso dentro de uma
  transação** tempo demais. **Ligue isso em produção.**
- `statement_timeout` → mata **queries** que demoram demais (ver doc
  [06](06-queries-lentas-e-explain.md)).

## Lições

- `idle in transaction` = `BEGIN` sem fim → segura locks sem fazer nada.
- `pg_cancel_backend` **não** resolve (não há query ativa); use `pg_terminate_backend`.
- A prevenção definitiva é `idle_in_transaction_session_timeout`.
- Causa raiz: quase sempre **bug de aplicação** ou pool mal configurado — conserte lá.

## Pegadinhas

- `idle` (sem "in transaction") é **inofensivo** — é só uma conexão parada, sem
  transação aberta. O vilão é o `idle in transaction`.
- `idle in transaction (aborted)`: a transação deu erro e ninguém deu `ROLLBACK`.
  Também segura recursos; mesma defesa.
- Essas sessões também **travam o autovacuum** (o banco não pode limpar linhas que a
  transação antiga ainda "enxerga") → bloat e lentidão com o tempo.
