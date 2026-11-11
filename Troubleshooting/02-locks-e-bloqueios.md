# 02 — Locks e bloqueios

## O que é

Um **lock** (bloqueio) é como o PostgreSQL impede que duas transações baguncem o
mesmo dado ao mesmo tempo. Quando duas transações querem alterar **a mesma linha**,
a segunda **espera** a primeira terminar. Isso é normal e correto — vira **problema**
quando a primeira demora (ou nunca termina).

## Por que importa

"O sistema travou" quase sempre é **uma query esperando um lock**. Saber achar
**quem segura o lock** é a habilidade nº 1 do troubleshooting.

## Objetos de teste

```sql
CREATE SCHEMA tshoot;
CREATE TABLE tshoot.contas(id int PRIMARY KEY, dono text, saldo numeric);
INSERT INTO tshoot.contas VALUES (1,'Ana',1000),(2,'Bruno',1000),(3,'Carla',1000);
```

## Como reproduzir o bloqueio

Precisamos de **duas sessões** (dois terminais `psql`):

**Sessão A** (segura o lock da linha 1 e não solta):
```sql
BEGIN;
UPDATE tshoot.contas SET saldo = saldo - 100 WHERE id = 1;
-- (não faz COMMIT ainda)
```

**Sessão B** (tenta alterar a MESMA linha → fica esperando):
```sql
BEGIN;
UPDATE tshoot.contas SET saldo = saldo - 50 WHERE id = 1;   -- <-- trava aqui
```

**Sessão C** (um observador) investiga:
```sql
SELECT a.pid AS pid_bloqueado, a.application_name AS quem,
       a.wait_event_type, a.wait_event,
       pg_blocking_pids(a.pid) AS bloqueado_pelos_pids,
       a.query AS query_bloqueada
FROM pg_stat_activity a
WHERE cardinality(pg_blocking_pids(a.pid)) > 0;
```

## O que você vê (saída real)

Quem está **bloqueado**:

```
pid_bloqueado        | 1681
quem                 | sessao_B
wait_event_type      | Lock
wait_event           | transactionid
bloqueado_pelos_pids | {1674}
query_bloqueada      | UPDATE tshoot.contas SET saldo = saldo - 50 WHERE id = 1;
```

Quem está **bloqueando**:

```
pid_bloqueador      | 1674
quem                | sessao_A
state               | active
transacao_aberta_ha | 00:00:06.05
query               | SELECT pg_sleep(12);
```

E a visão crua `pg_locks` na tabela:

```
 locktype |       mode       | granted | pid
----------+------------------+---------+------
 relation | RowExclusiveLock | t       | 1674   <- A
 relation | RowExclusiveLock | t       | 1681   <- B (lock de tabela já concedido)
 tuple    | ExclusiveLock    | t       | 1681   <- B segura o lock da tupla, mas...
```

Repare: `wait_event = transactionid`. A sessão B está esperando a **transação** da
sessão A **terminar** (é assim que o PostgreSQL implementa o lock de linha: a linha
"aponta" para a transação que a alterou, e quem quer a mesma linha espera essa
transação). Assim que A faz `COMMIT`/`ROLLBACK`, **B segue sozinha**:

```
## log da Sessão B — o UPDATE só completou DEPOIS de A liberar
SET
BEGIN
UPDATE 1
COMMIT
```

## Como ler os níveis de lock (rápido)

- `RowExclusiveLock` (na *relation*) — lock **leve** de tabela que todo
  `INSERT/UPDATE/DELETE` pega. Vários coexistem; **não** é ele que trava.
- O que trava no exemplo é o lock de **linha** (via `transactionid`): duas
  transações disputando a **mesma linha**.
- Locks **pesados** de tabela (ex.: `AccessExclusiveLock`, pego por `ALTER TABLE`,
  `DROP`, `VACUUM FULL`) bloqueiam **todo mundo**, inclusive leitura. Cuidado com DDL
  em horário de pico.

## Lições

- Bloqueio = uma transação esperando outra soltar o lock da **mesma** linha/recurso.
- `pg_blocking_pids(pid)` entrega o culpado em uma linha.
- `wait_event_type = Lock` no `pg_stat_activity` é o sinal de "estou esperando lock".
- O lock some quando a transação dona faz `COMMIT` ou `ROLLBACK`.

## Pegadinhas

- A causa raiz quase nunca é "o banco"; é uma **transação aberta demais** (app que
  esqueceu de commitar, ou uma query lenta dentro da transação).
- DDL (`ALTER TABLE`) pega lock pesado e pode travar **leituras** também. Rode em
  janela de manutenção ou com `lock_timeout`.
- Não confunda **bloqueio** (um espera o outro, resolve sozinho) com **deadlock**
  (os dois se esperam em ciclo, o banco mata um) — doc [04](04-deadlocks.md).
