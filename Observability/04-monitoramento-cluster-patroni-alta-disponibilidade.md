# 04 — Monitoramento de Cluster Patroni (Alta Disponibilidade)

## O que é

O **Patroni** é o orquestrador de alta disponibilidade e consenso (via DCS / etcd) do cluster PostgreSQL.
Ele expõe nativamente na sua API REST (porta `8008`) um endpoint de telemetria no formato Prometheus:
**`http://<ip-do-no>:8008/metrics`**.

Esse endpoint entrega métricas fundamentais de HA em tempo real:
- **`patroni_primary`**: Vale `1` se o nó for o Leader atual; `0` se for réplica.
- **`patroni_replica`**: Vale `1` se o nó for uma réplica ativa.
- **`patroni_postgres_running`**: Indica se o processo do PostgreSQL está vivo e operacional.
- **`patroni_postgres_timeline`**: Timeline atual da replicação.
- **`patroni_dcs_last_seen`**: Timestamp da última comunicação com o etcd.
- **`patroni_is_paused`**: Indica se a manutenção automática (pause) está ativa.
- **`patroni_xlog_location` / `patroni_xlog_replayed_location`**: LSN de gravação e replay de WAL.

---

## Por que importa

- **Visualização imediata de Failover e Switchover:** Se o Leader cair e o Patroni eleger uma nova réplica, a métrica `patroni_primary` chaveia imediatamente no painel do PMM, mostrando a mudança de papéis.
- **Prevenção de Split-Brain e Perda de Quorum:** O dashboard monitora o tempo de resposta do DCS (etcd), alertando se um nó perder comunicação antes que ele seja forçado a reiniciar.
- **Acompanhamento de Lag de Replicação:** Permite acompanhar em megabytes a distância exata de WAL entre o nó primário e as réplicas.

---

## Como reproduzir

### 1. Configurar a escuta da API REST do Patroni para aceitar conexões locais

Por padrão, a configuração inicial do Patroni vincula a API apenas ao IP físico externo do nó (ex: `192.168.1.15:8008`).
Para que o agente local do PMM (`vmagent`) consiga raspar as métricas em loopback (`127.0.0.1`), ajuste a diretiva `restapi.listen` no arquivo `/etc/patroni/patroni.yml` de cada nó:

```yaml
restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.1.15:8008
```

> **Atenção:** Mantenha o `connect_address` apontando para o IP de rede da máquina (`192.168.1.15:8008`) para que o HAProxy e os outros membros do cluster saibam onde encontrar o nó.

Reinicie o serviço do Patroni para aplicar:

```bash
systemctl restart patroni
```

Valide se o endpoint responde tanto no loopback quanto na rede:

```bash
curl -s http://127.0.0.1:8008/metrics | head -n 10
```

### 2. Cadastrar o coletor externo no PMM

Execute o comando `pmm-admin add external` em cada servidor do cluster, associando ao cluster `pgha`:

**No nó db-01:**
```bash
pmm-admin add external \
  --service-name=patroni-db-01 \
  --listen-port=8008 \
  --metrics-path=/metrics \
  --cluster=pgha
```

**No nó db-02:**
```bash
pmm-admin add external \
  --service-name=patroni-db-02 \
  --listen-port=8008 \
  --metrics-path=/metrics \
  --cluster=pgha
```

**No nó db-03:**
```bash
pmm-admin add external \
  --service-name=patroni-db-03 \
  --listen-port=8008 \
  --metrics-path=/metrics \
  --cluster=pgha
```

---

## O que você vê (saída real)

### 1. Serviço e exportador externo ativos no `pmm-admin list`:

```
Service type             Service name            Address and port         Service ID
PostgreSQL               postgresql-db-01        192.168.1.15:5432        872f0645-294c-4c0e-bb3d-e3e69ae2c7c2
External:external        patroni-db-01                                    79ac0d09-89ad-4079-b4db-5a1752704252

Agent type                           Status           Metrics Mode        Agent ID                                    Service ID                                  Port
node_exporter                        Running          push                8d35add7-a1fe-4e10-a171-53b2baa14c23                                                    42003
postgres_exporter                    Running          push                484afbc8-6490-4889-9d03-71358dc3fc97        872f0645-294c-4c0e-bb3d-e3e69ae2c7c2        42001
external-exporter                    Running          push                6532079d-3da1-48f0-b03b-903297202492        79ac0d09-89ad-4079-b4db-5a1752704252        8008
```

- **`external-exporter: Running` na porta `8008`**: O PMM está raspando o endpoint `/metrics` do Patroni e empurrando os dados para o VictoriaMetrics.

### 2. Métricas de Leader e Replicas ingeridas no VictoriaMetrics:

Consulta ao servidor PMM (`192.168.1.16:9090`):

```json
[
  {
    "metric": {
      "__name__": "patroni_primary",
      "cluster": "pgha",
      "name": "db-01",
      "service_name": "patroni-db-01"
    },
    "value": [1794605877, "1"]   <-- Leader
  },
  {
    "metric": {
      "__name__": "patroni_primary",
      "cluster": "pgha",
      "name": "db-02",
      "service_name": "patroni-db-02"
    },
    "value": [1794605877, "0"]   <-- Replica
  },
  {
    "metric": {
      "__name__": "patroni_primary",
      "cluster": "pgha",
      "name": "db-03",
      "service_name": "patroni-db-03"
    },
    "value": [1794605877, "0"]   <-- Replica
  }
]
```

---

## Lições

- **Dois exportadores por nó:** Em um ambiente com Patroni, o monitoramento é dual:
  1. `postgres_exporter` (porta 5432): métricas do motor relacional (tabelas, locks, queries);
  2. `external-exporter` (porta 8008): métricas do orquestrador de HA (consenso, leader/réplicas).
- **Cluster label (`--cluster=pgha`):** Ao cadastrar com `--cluster=pgha`, o PMM unifica os nós no mesmo seletor de cluster nos dashboards do Grafana.

---

## Pegadinhas

- **A armadilha da colisão de portas no `patroni.yml`:**
  O arquivo `/etc/patroni/patroni.yml` contém **duas seções** que utilizam a palavra-chave `listen:`:
  ```yaml
  restapi:
    listen: 0.0.0.0:8008        # <-- API REST do Patroni
    connect_address: 192.168.1.15:8008

  postgresql:
    listen: 192.168.1.15:5432   # <-- Porta do PostgreSQL!
    connect_address: 192.168.1.15:5432
  ```
  Se você usar um comando `sed` ou script que substitua genericamente `listen: ...` por `0.0.0.0:8008`, o PostgreSQL tentará subir na porta 8008, colidirá com a API do Patroni e falhará com erro fatal:
  `LOG: could not bind IPv4 address "0.0.0.0": Address already in use (port 8008)`.
  **Certifique-se sempre de alterar apenas o `listen` da seção `restapi`!**
- **Erro `dial tcp 127.0.0.1:8008: connect: connection refused`:** Ocorre quando o `restapi.listen` está restrito apenas ao IP da rede física (`192.168.1.15:8008`). Como o `pmm-admin add external` valida a conexão local em `127.0.0.1`, o comando falha até que a escuta seja alterada para `0.0.0.0:8008`.
