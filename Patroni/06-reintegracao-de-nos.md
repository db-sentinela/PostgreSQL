# 06 — Reintegração de nós (como um nó que caiu volta)

## O que é

Depois de um failover, o **antigo primary** (ou qualquer nó que estava fora) precisa
**voltar ao cluster**. Ele não volta como primary — volta como **réplica**, seguindo
o novo líder. O Patroni faz isso automaticamente quando o serviço sobe.

## O detalhe importante: streaming simples x pg_rewind

Há **dois jeitos** de o nó voltar a acompanhar o novo primary:

1. **Streaming simples** — se o nó que voltou **não tem nada a mais** que o novo
   primary (não divergiu), ele apenas pede o WAL que faltou e segue em frente.
2. **pg_rewind** — se o antigo primary chegou a gravar algo que o novo primary
   **não tem** (divergência de timeline), não dá para simplesmente "continuar": seria
   inconsistente. O `pg_rewind` **desfaz** as mudanças divergentes do nó que voltou e
   o realinha com o novo primary. Só então ele volta a replicar.

> Por isso habilitamos `use_pg_rewind: true` no patroni.yml. Sem isso, um nó que
> divergiu só voltaria recriando a réplica do zero (lento).

## Como reproduzir

```bash
# depois de derrubar e o cluster ter promovido outro primary, religue o nó:
systemctl start patroni
patronictl -c /etc/patroni/patroni.yml list
```

## O que você vê (saída real)

O db-01 (que tínhamos derrubado) voltou como **réplica**, na mesma timeline:

```
| Member | Host         | Role    | State     | TL |
| db-01  | 192.168.1.15 | Replica | streaming |  7 |
| db-02  | 192.168.1.17 | Replica | streaming |  7 |
| db-03  | 192.168.1.18 | Leader  | running   |  7 |
```

E uma checagem de **consistência**: a linha que foi escrita **enquanto o db-01
estava fora** ("durante-failover") apareceu nele depois do reingresso:

```
## db-01 tem a linha escrita durante o failover?
achou=1  total=4
```

### Neste teste, precisou de pg_rewind?

**Não.** Como todas as escritas durante a queda foram para o **novo** primary (via
VIP), o db-01 **não tinha nada divergente** — ele só buscou o WAL que faltava e
seguiu via **streaming simples**. O `pg_rewind` entra em cena apenas quando o nó que
volta tem escritas próprias que o novo primary nunca viu (ex.: um split de rede em
que o nó antigo recebeu writes locais).

## Recriando uma réplica do zero (quando a reintegração não resolve)

Se um nó ficou corrompido ou muito atrasado e não consegue voltar, você força a
**recriação completa** (novo `pg_basebackup` a partir do primary):

```bash
patronictl -c /etc/patroni/patroni.yml reinit pgha db-03
```

Isso apaga os dados locais da réplica e copia tudo de novo do primary. Use quando a
reintegração normal falhar.

## Lições

- Um nó que caiu **volta como réplica**, nunca reassume como primary sozinho.
- `pg_rewind` realinha nós **divergentes**; sem divergência, é só streaming.
- `reinit` é o "reset de fábrica" de uma réplica — recria do zero a partir do primary.

## Pegadinhas

- Se o nó voltar e ficar em `start failed`, veja `journalctl -u patroni` — muitas
  vezes é divergência que o pg_rewind resolve (precisa de `use_pg_rewind: true` e de
  `data-checksums` ou `wal_log_hints: on`, que já configuramos).
- `reinit` **apaga** os dados da réplica alvo. Nunca rode isso contra o primary.
