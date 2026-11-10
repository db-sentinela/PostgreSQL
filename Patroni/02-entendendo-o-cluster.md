# 02 — Entendendo o `patronictl list`

O comando que você vai rodar mais vezes na vida é este. Saber lê-lo é 80% do
trabalho de operar um cluster Patroni.

```bash
patronictl -c /etc/patroni/patroni.yml list
```

> Dica: crie um atalho. No nosso lab o Patroni está num venv, então o caminho é
> `/opt/patroni-venv/bin/patronictl -c /etc/patroni/patroni.yml`. Exporte
> `export PATRONICTL="/opt/patroni-venv/bin/patronictl -c /etc/patroni/patroni.yml"`.

## Saída real do nosso cluster saudável

```
+ Cluster: pgha (7706327264164550049) --------+----+-------------+-----+------------+-----+
| Member | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+--------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| db-01  | 192.168.1.15 | Leader  | running   |  9 |             |     |            |     |
| db-02  | 192.168.1.17 | Replica | streaming |  9 |   0/6038568 |   0 |  0/6038568 |   0 |
| db-03  | 192.168.1.18 | Replica | streaming |  9 |   0/6038778 |   0 |  0/6038778 |   0 |
+--------+--------------+---------+-----------+----+-------------+-----+------------+-----+
```

## Coluna por coluna

- **Cluster: pgha (7706...)** — o nome do cluster (`scope`) e o **system identifier**
  do banco. Esse número é a "identidade" do cluster; todas as réplicas o compartilham.
- **Member** — o nome de cada nó (`name` no patroni.yml).
- **Host** — o IP do nó.
- **Role** — o papel atual:
  - `Leader` = o primary (aceita escrita).
  - `Replica` = standby (somente leitura).
  - Pode aparecer `Sync Standby` (réplica síncrona) ou `Standby Leader` (em cluster em cascata).
- **State** — a situação operacional:
  - `running` = primary no ar.
  - `streaming` = réplica recebendo WAL em tempo real (**o que você quer ver**).
  - `in archive recovery` = réplica buscando WAL do arquivo, não do stream (atenção).
  - `stopped` / `start failed` = nó parado ou com problema.
- **TL (Timeline)** — a "linha do tempo" do banco. **Toda vez que há uma promoção**
  (failover ou switchover), a timeline **incrementa**. Todos os nós saudáveis devem
  estar na **mesma TL**. (No nosso lab está em 9 porque já fizemos vários testes de troca.)
- **Receive LSN / Replay LSN** — a posição no WAL que a réplica **recebeu** e a que
  ela já **aplicou**. LSN é como um "número de página" do diário WAL.
- **Lag** — o **atraso** da réplica em relação ao primary, em bytes/MB. `0` = em dia.

## O que é "saudável"

✅ Um `Leader` em `running` + réplicas em `streaming`, **todas na mesma TL** e com
**Lag 0** (ou poucos bytes). Foi exatamente o que vimos acima.

🚩 Sinais de alerta: réplica `stopped`, TL diferente entre nós, Lag crescendo,
ou nenhum `Leader` aparecendo.

## Outros comandos de inspeção

```bash
# Histórico de trocas de líder (quando e por quê a timeline mudou):
patronictl ... history

# Configuração efetiva do cluster (DCS):
patronictl ... show-config

# Desenho da topologia (útil em cascata):
patronictl ... topology
```

Exemplo real do `history` (cada linha é uma promoção):

```
+----+-----------+------------------------------+----------------------------------+------------+
| TL |       LSN | Reason                       | Timestamp                        | New Leader |
+----+-----------+------------------------------+----------------------------------+------------+
|  1 |  84222904 | no recovery target specified | 2026-11-09T21:10:45...            | db-02      |
|  2 |  84223552 | no recovery target specified | 2026-11-09T21:11:19...            | db-01      |
|  3 | 100663456 | no recovery target specified | 2026-11-10T19:26:10...            | db-01      |
+----+-----------+------------------------------+----------------------------------+------------+
```

## A REST API do Patroni (o que o HAProxy usa)

Cada nó expõe uma API HTTP na porta **8008**. É ela que diz "eu sou primary" ou
"eu sou réplica" — e é assim que o HAProxy decide o roteamento:

```bash
curl -s http://192.168.1.15:8008/primary   # responde 200 SÓ no primary
curl -s http://192.168.1.15:8008/replica   # responde 200 SÓ nas réplicas
curl -s http://192.168.1.15:8008/health    # estado geral em JSON
```

## Lições

- `patronictl list` é o seu "raio-X" do cluster. Rode antes e depois de qualquer ação.
- **Mesma timeline + Lag 0 + um Leader** = tudo certo.
- A REST API (porta 8008) é a "fonte da verdade" do papel de cada nó.

## Pegadinhas

- Esquecer o `-c /caminho/patroni.yml` → o comando não acha o cluster.
- Confundir `Receive LSN` (recebeu) com `Replay LSN` (aplicou): uma réplica pode ter
  recebido o WAL mas ainda estar aplicando (replay lag).
