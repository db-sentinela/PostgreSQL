# 10 — Cheat sheet de comandos

A cola do dia a dia. No nosso lab o `patronictl` está num venv; crie um atalho:

```bash
alias pt='/opt/patroni-venv/bin/patronictl -c /etc/patroni/patroni.yml'
```

(Nos exemplos abaixo, `pt` = esse comando completo. Cluster = `pgha`.)

## Ver o estado

```bash
pt list                 # raio-X do cluster (rode sempre!)
pt history              # histórico de trocas de líder / timelines
pt show-config          # configuração efetiva (DCS)
pt topology             # topologia (útil em cascata)
```

## Trocar o primary

```bash
# planejado, sem perda (você escolhe quando):
pt switchover pgha --leader <primary-atual> --candidate <novo> --force

# forçar promoção de um nó específico (emergência):
pt failover pgha --candidate <no> --force
```

## Manutenção

```bash
pt pause pgha --wait      # congela a automação (antes de mexer num nó)
pt resume pgha --wait     # volta a automação (NÃO esqueça disso)
```

## Reiniciar / recriar nós

```bash
pt restart pgha <no>      # reinicia o PostgreSQL de um nó, sob controle do Patroni
pt reinit pgha <no>       # RECRIA a réplica do zero (apaga e copia do primary)
```

## Checagem direta de papel (REST API, porta 8008)

```bash
curl -s http://<ip>:8008/primary   # 200 só no primary
curl -s http://<ip>:8008/replica   # 200 só nas réplicas
curl -s http://<ip>:8008/health    # JSON com o estado
```

## etcd (a fonte da verdade)

```bash
EP=192.168.1.15:2379,192.168.1.17:2379,192.168.1.18:2379
etcdctl --endpoints=$EP endpoint health         # saúde / quorum
etcdctl --endpoints=$EP member list -w table    # membros
etcdctl --endpoints=$EP endpoint status -w table # quem é o leader do etcd
```

## Conexões da aplicação (sempre pelo VIP)

```bash
# ESCRITA -> vai sempre ao primary:
psql "host=192.168.1.10 port=5000 user=<u> dbname=<db>"
# LEITURA -> balanceada entre as réplicas:
psql "host=192.168.1.10 port=5001 user=<u> dbname=<db>"
```

## Diagnóstico rápido no PostgreSQL

```sql
-- sou primary ou réplica?
select pg_is_in_recovery();          -- f = primary, t = réplica

-- quem está replicando de mim (rodar no primary):
select application_name, state, sync_state, replay_lag from pg_stat_replication;

-- (na réplica) atraso em relação ao primary:
select now() - pg_last_xact_replay_timestamp() as atraso;
```

## Logs (quando algo dá errado)

```bash
journalctl -u patroni -f            # log do Patroni ao vivo
journalctl -u etcd -n 50            # log do etcd
tail -f /var/lib/pgsql/18/data/log/*.log   # log do PostgreSQL
systemctl status haproxy keepalived etcd patroni
```

## Regra de ouro

> **Antes e depois de qualquer ação, rode `pt list`.** Confirme: um `Leader`, todas
> as réplicas em `streaming`, mesma **timeline**, **lag** baixo. Se algo fugir disso,
> pare e investigue antes de continuar.
