# 04 — Deadlocks

## O que é

**Deadlock** ("abraço mortal") é quando duas transações se esperam **em ciclo**:
A espera algo que B segura, **e** B espera algo que A segura. Nenhuma das duas pode
avançar sozinha — ficariam presas para sempre. O PostgreSQL **detecta** isso e
**mata uma** delas para o sistema continuar.

> Diferença crucial para um bloqueio comum (doc [02](02-locks-e-bloqueios.md)):
> no bloqueio, um espera o outro **terminar** (resolve sozinho). No deadlock, os
> dois se esperam **mutuamente** (nunca resolveria — por isso o banco intervém).

## Por que importa

Deadlock é **erro de aplicação**: significa que o código adquire locks em **ordem
inconsistente**. Você precisa (1) reconhecer o erro, (2) achar as queries culpadas no
log e (3) corrigir a ordem no código.

## Como reproduzir (o clássico "ordem oposta")

**Sessão A:**
```sql
BEGIN;
UPDATE tshoot.contas SET saldo=saldo-100 WHERE id=1;   -- trava a linha 1
-- (espera um instante)
UPDATE tshoot.contas SET saldo=saldo+100 WHERE id=2;   -- quer a linha 2
COMMIT;
```

**Sessão B (ao mesmo tempo, ordem invertida):**
```sql
BEGIN;
UPDATE tshoot.contas SET saldo=saldo-100 WHERE id=2;   -- trava a linha 2
-- (espera um instante)
UPDATE tshoot.contas SET saldo=saldo+100 WHERE id=1;   -- quer a linha 1  -> DEADLOCK
COMMIT;
```

A trava a linha 1 e quer a 2; B trava a linha 2 e quer a 1. Ciclo fechado.

## O que você vê (saída real)

A **vítima** (sessão B) recebe o erro e sofre `ROLLBACK` automático:

```
ERROR:  deadlock detected
DETAIL:  Process 2400 waits for ShareLock on transaction 795; blocked by process 2393.
         Process 2393 waits for ShareLock on transaction 796; blocked by process 2400.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,23) in relation "contas"
ROLLBACK
```

A **vencedora** (sessão A) segue normal e comita. No fim, as contas ficam
consistentes (só a transação vencedora valeu):

```
 id | dono  | saldo
----+-------+-------
  1 | Ana   |   900     <- A: -100
  2 | Bruno |  1100     <- A: +100
  3 | Carla |  1000
```

## O ouro está no log do servidor

O erro do cliente diz "deadlock detected", mas é o **log** que mostra **as queries
exatas** de cada lado — é por ali que você descobre o trecho de código a corrigir:

```
[2400] ERROR:  deadlock detected
[2400] DETAIL:  Process 2400 waits for ShareLock on transaction 795; blocked by process 2393.
        Process 2393 waits for ShareLock on transaction 796; blocked by process 2400.
        Process 2400: UPDATE tshoot.contas SET saldo=saldo+100 WHERE id=1;
        Process 2393: UPDATE tshoot.contas SET saldo=saldo+100 WHERE id=2;
[2400] CONTEXT:  while updating tuple (0,23) in relation "contas"
[2400] STATEMENT:  UPDATE tshoot.contas SET saldo=saldo+100 WHERE id=1;
```

Local do log: `/var/lib/pgsql/18/data/log/*.log` (ou `SELECT pg_current_logfile();`).

## Como o PostgreSQL detecta

- Toda transação que espera um lock por mais que `deadlock_timeout` (**padrão 1s**)
  dispara uma **checagem de ciclo**.
- Se houver ciclo, o banco escolhe uma **vítima** (normalmente a que fechou o ciclo),
  mata a transação dela com `ERROR: deadlock detected` e libera as outras.
- Ou seja: o deadlock **não** derruba o banco — ele se auto-resolve, abortando **uma**
  transação. O custo é essa transação ter que ser **repetida** pela aplicação.

## Como prevenir

1. **Sempre adquira locks na mesma ordem.** Se todo mundo altera as linhas em ordem
   crescente de `id`, nunca há ciclo. (No exemplo, se B também fosse 1→2, não daria
   deadlock — viraria um bloqueio comum.)
2. **Transações curtas.** Quanto menos tempo segurando locks, menor a janela de ciclo.
3. **`SELECT ... FOR UPDATE` com `ORDER BY`** ao travar várias linhas de uma vez.
4. **Repita a transação** na aplicação ao receber o erro de deadlock (código SQLSTATE
   **`40P01`**). Deadlock esporádico acontece; o app deve saber tentar de novo.

## Lições

- Deadlock = espera **em ciclo**; o banco mata **uma** vítima (SQLSTATE `40P01`).
- O cliente vê `deadlock detected`; o **log** mostra as **queries** culpadas.
- Causa raiz = ordem inconsistente de locks. Prevenção = **ordem única** + transações
  curtas + retry.

## Pegadinhas

- Deadlock **não** é "banco travou" — ele se resolve sozinho em ~1s matando uma
  transação. Se "tudo parou" e ninguém morreu, é **bloqueio** comum, não deadlock.
- Aumentar `deadlock_timeout` não evita deadlock; só atrasa a detecção.
- Tratar o erro só "logando" na aplicação perde a transação. O certo é **repetir**.
