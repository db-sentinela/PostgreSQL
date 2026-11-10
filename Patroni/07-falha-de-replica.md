# 07 — Falha de uma réplica

## O que é

Uma das **réplicas** cai (não o primary). Este é o cenário **menos grave** — e é
importante entender por quê.

## Por que importa

Mostra que o cluster **continua escrevendo** mesmo com um nó standby fora. O que você
perde é **redundância** (menos cópias prontas para assumir), não **disponibilidade**
de escrita.

## Como reproduzir

```bash
# derruba uma réplica (ex.: db-02):
systemctl stop patroni     # no db-02

# observa do primary:
patronictl -c /etc/patroni/patroni.yml list
```

## O que você vê (saída real)

Com o db-02 fora, o primary (db-03 neste teste) **seguiu normal** e o db-01
continuou replicando:

```
| Member | Host         | Role    | State     | TL |
| db-01  | 192.168.1.15 | Replica | streaming |  7 |
| db-02  | 192.168.1.17 | Replica | stopped   |    |
| db-03  | 192.168.1.18 | Leader  | running   |  7 |
```

A **escrita continuou** sem qualquer impacto:

```
## escrita pelo VIP continua?
INSERT 0 1
ok primary = 192.168.1.18
```

E o primary passou a enxergar **apenas uma** réplica conectada (a que sobrou):

```
## réplicas conectadas agora
db-01 | streaming
```

Ao religar o db-02, ele voltou a `streaming` com lag 0 automaticamente.

## Por que o primary não se importa (modo assíncrono)

No nosso cluster a replicação é **assíncrona**: o primary **não espera** a réplica
para confirmar o commit. Então perder uma réplica não atrasa nem bloqueia a escrita.

> ⚠️ **Atenção se você usar replicação síncrona:** se exigir que uma réplica
> confirme cada commit e essa réplica cair, a escrita pode **travar** até outra
> assumir o papel síncrono (ou até você afrouxar a regra). É um trade-off: mais
> segurança (RPO 0) em troca de sensibilidade à queda de réplicas. Planeje o
> `synchronous_node_count` com isso em mente.

## Lições

- Perder uma réplica (assíncrono) = **primary intacto**, só cai a redundância.
- O risco real é **acumular** falhas: com uma réplica fora, você está mais perto de
  perder o quorum se outro nó cair (doc [09](09-perda-de-quorum-split-brain.md)).
- Réplicas voltam sozinhas ao subir o Patroni.

## Pegadinhas

- "Está tudo bem, é só uma réplica" — sim, **mas conserte logo**. Redundância
  reduzida é dívida: o próximo nó a cair pode custar o quorum.
- Em modo síncrono, a queda de uma réplica tem impacto bem diferente. Saiba qual modo
  o seu cluster usa (`sync_state` no `pg_stat_replication`).
