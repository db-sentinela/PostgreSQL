# 03 — Replicação e consistência

Aqui você entende **como os dados chegam nas réplicas** e como medir se estão em dia.

## A ideia em 30 segundos

1. Você faz um `INSERT` no **primary**.
2. O primary grava a mudança no **WAL** (o diário).
3. O primary **envia** esse WAL pela rede para as réplicas.
4. Cada réplica **aplica** a mudança e fica idêntica ao primary.

Tudo isso em **milissegundos**. As réplicas ficam em **somente-leitura** justamente
para nunca divergirem do primary.

## O pipeline do WAL (as 4 etapas que o LSN mostra)

Quando o WAL viaja do primary para a réplica, ele passa por 4 marcos. O comando
`pg_stat_replication` (rodado **no primary**) mostra cada um:

| Etapa | Coluna | Significa |
|-------|--------|-----------|
| 1. Enviado | `sent_lsn` | O primary mandou pela rede |
| 2. Escrito | `write_lsn` | A réplica recebeu e escreveu no SO |
| 3. Persistido | `flush_lsn` | A réplica gravou em disco (seguro) |
| 4. Aplicado | `replay_lsn` | A réplica já **aplicou** (visível em consultas) |

**LSN** (Log Sequence Number) é a posição no WAL — pense num "número de página"
sempre crescente do diário.

## Teste real: escrever no primary e ver nas réplicas

```bash
# 1) escreve no PRIMARY (sempre pelo VIP:5000):
psql "host=192.168.1.10 port=5000 user=postgres dbname=postgres" -c \
  "create table ha_demo(id serial primary key, msg text, criado_em timestamptz default now());"
psql "host=192.168.1.10 port=5000 user=postgres dbname=postgres" -c \
  "insert into ha_demo(msg) values ('estagiario-1'),('estagiario-2'),('estagiario-3');"

# 2) lê DIRETO em cada réplica (porta 5432 do nó):
psql "host=192.168.1.17 port=5432 user=postgres dbname=postgres" -c "select count(*) from ha_demo;"
psql "host=192.168.1.18 port=5432 user=postgres dbname=postgres" -c "select count(*) from ha_demo;"
```

### O que você vê (saída real)

```
primary diz: recovery=f  linhas=3  lsn=0/602E128
  replica 192.168.1.17: recovery=t linhas=3 ultima=estagiario-3
  replica 192.168.1.18: recovery=t linhas=3 ultima=estagiario-3
```

As 3 linhas apareceram **instantaneamente** nas duas réplicas. `recovery=t` confirma
que elas são standby (somente-leitura); `recovery=f` confirma que o primary é quem escreve.

## Medindo o atraso (lag) — saída real

```
 application_name |   state   | sent_lsn  | write_lsn | flush_lsn | replay_lsn |    write_lag    |   replay_lag
------------------+-----------+-----------+-----------+-----------+------------+-----------------+-----------------
 db-02            | streaming | 0/602E160 | 0/602E160 | 0/602E160 | 0/602E160  | 00:00:00.002081 | 00:00:00.007683
 db-03            | streaming | 0/602E160 | 0/602E160 | 0/602E160 | 0/602E160  | 00:00:00.001092 | 00:00:00.007577
```

Leitura: os 4 LSNs estão **iguais** (nada atrasado) e os lags são de **milissegundos**
(`write_lag` ~2 ms, `replay_lag` ~7 ms). Isso é um cluster saudável.

> Em produção, monitore `replay_lag`. Se começar a crescer (segundos, minutos), a
> réplica não está acompanhando — rede, disco lento ou carga alta.

## Prova: réplica é somente-leitura

Tentar escrever **direto numa réplica** tem que falhar:

```bash
psql "host=192.168.1.17 port=5432 user=postgres dbname=postgres" -c \
  "insert into ha_demo(msg) values('nao-deveria-funcionar');"
```

### Saída real

```
ERROR:  cannot execute INSERT in a read-only transaction
```

É por isso que a aplicação **nunca** deve apontar para o IP de um nó fixo: ela deve
usar o **VIP + HAProxy**, que sempre acha o primary (doc
[04](04-switchover-planejado.md) mostra o roteamento mudando junto com o primary).

## Replication slots

```
 slot_name | slot_type | active
-----------+-----------+--------
 db_02     | physical  | t
 db_03     | physical  | t
```

Os **slots** garantem que o primary **não descarte** WAL que uma réplica ainda não
consumiu. Protegem contra a réplica "ficar para trás e perder o fio". O Patroni
cria e gerencia isso para você (`use_slots: true`).

## Lições

- Replicação física = o primary manda o **WAL**, a réplica **reaplica**. Cópia exata.
- `pg_stat_replication` é o painel do atraso. LSNs iguais + lag em ms = saudável.
- Réplica é **somente-leitura**. Escrita só no primary, e sempre via VIP.

## Pegadinhas

- Assíncrono (nosso padrão) significa que, num failover, as **últimas** transações
  podem não ter chegado na réplica. Se o negócio não tolera isso, use replicação
  síncrona (ver doc [01](01-conceitos-fundamentais.md)).
- Lag alto nem sempre é a réplica: muitas vezes é **disco** ou **rede** do nó standby.
