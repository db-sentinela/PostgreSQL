# 02 — Monitoramento PostgreSQL Single Instance

## O que é

O monitoramento de uma instância PostgreSQL no PMM é executado através do **`postgres_exporter`**.
Ele se conecta periodicamente ao banco de dados com um usuário dedicado e extrai métricas cruciais de
saúde operacional:
- Conexões ativas x limites (`max_connections`);
- Transações por segundo (TPS / Commits e Rollbacks);
- Taxa de acerto em cache (*Cache Hit Ratio* dos buffers);
- Checkpoints, escritas de WAL e estatísticas de background writer;
- Locks e deadlocks acumulados.

---

## Por que importa

- **Detecção precoce de saturação:** Alertas disparam antes que o banco atinja o esgotamento de conexões (`FATAL: remaining connection slots are reserved...`) ou o disco encha com WALs acumulados.
- **Segurança e Menor Privilégio:** O usuário do monitoramento **nunca deve ser superusuário**. O PostgreSQL oferece uma role nativa de sistema específica para ferramentas de observabilidade (`pg_monitor`).

---

## Como reproduzir

### 1. Criar o usuário de monitoramento no PostgreSQL

Conecte no PostgreSQL como superusuário (`postgres`) e crie a role `pmm` com permissões restritas de leitura de métricas:

```sql
CREATE USER pmm WITH PASSWORD 'S3nh4_F0rt3';

-- Conceder os privilégios nativos de observabilidade
GRANT pg_monitor TO pmm;
GRANT pg_read_all_settings TO pmm;
GRANT pg_read_all_stats TO pmm;
```

> **Atenção:** Se você estiver em um cluster Patroni ou com replicação por streaming, basta criar o usuário no **nó Leader** (Primary) — ele será replicado instantaneamente para todas as réplicas.

### 2. Registrar o serviço PostgreSQL no `pmm-admin`

No servidor do banco de dados, execute o comando de registro indicando as credenciais e o endpoint local:

```bash
pmm-admin add postgresql \
  --username=pmm \
  --password=S3nh4_F0rt3 \
  --service-name=postgresql-db-01 \
  --query-source=pgstatements \
  192.168.1.15:5432
```

---

## O que você vê (saída real)

### 1. Confirmação do serviço e exporters via `pmm-admin list`:

```
Service type        Service name            Address and port         Service ID
PostgreSQL          postgresql-db-01        192.168.1.15:5432        872f0645-294c-4c0e-bb3d-e3e69ae2c7c2

Agent type                           Status           Metrics Mode        Agent ID                                    Service ID                                  Port
pmm_agent                            Connected                            5360897a-36d0-4642-972f-2429ae7fff07                                                    0 
node_exporter                        Running          push                8d35add7-a1fe-4e10-a171-53b2baa14c23                                                    42003 
postgres_exporter                    Running          push                484afbc8-6490-4889-9d03-71358dc3fc97        872f0645-294c-4c0e-bb3d-e3e69ae2c7c2        42001 
postgresql_pgstatements_agent        Running                              2a03cf5e-e7fb-4af7-8191-67d926875e43        872f0645-294c-4c0e-bb3d-e3e69ae2c7c2        0 
vmagent                              Running          push                52b87874-9b62-48f4-b1bb-a02037502036                                                    42002 
nomad_agent                          Running                              08ac4075-72b0-42d4-bbd0-248aa40a643a                                                    42000
```

- **`postgres_exporter: Running` na porta `42001`**: O exportador está conectado ao PostgreSQL e coletando métricas a cada poucos segundos.

### 2. Validação da métrica `pg_up` no VictoriaMetrics do PMM Server:

Ao consultar a API de métricas do PMM Server (`192.168.1.16:9090`):

```json
{
  "metric": {
    "__name__": "pg_up",
    "service_name": "postgresql-db-01",
    "service_type": "postgresql"
  },
  "value": [1794605409, "1"]
}
```

O valor `"1"` confirma que o banco respondeu com sucesso às sondagens do exporter.

---

## Lições

- **O poder da role `pg_monitor`:** A partir do PostgreSQL 10, não existe justificativa para rodar exporters como superusuário. O `pg_monitor` agrega `pg_read_all_settings` (ler parâmetros do `postgresql.conf`) e `pg_read_all_stats` (ler views estatísticas de qualquer tabela), sem permitir que o exporter leia o conteúdo dos dados dos clientes.
- **Porta do exporter (`42001`):** O `postgres_exporter` roda como um processo filho do `pmm-agent` e escuta apenas em `127.0.0.1`. O `vmagent` local coleta essas métricas em loopback e faz o push seguro para o servidor central.

---

## Pegadinhas

- **Erro `pq: password authentication failed for user "pmm" (28P01)`:** Esse erro no `journalctl -u pmm-agent` significa que:
  1. A role `pmm` não foi criada no banco;
  2. A senha passada no comando `pmm-admin add` divergiu da senha cadastrada no PostgreSQL;
  3. Ou o `pg_hba.conf` não possui regra permitindo conexões com `scram-sha-256` para o usuário `pmm`.
- **Falso positivo de `Waiting` após o cadastro:** Quando você executa `pmm-admin add postgresql`, os agentes podem ficar no status `Waiting` ou `Unknown` por até 10-15 segundos enquanto o Nomad baixa os binários e inicia o processo. Aguarde alguns instantes antes de diagnosticar erro.
- **Conexões do exporter no `pg_stat_activity`:** O `postgres_exporter` mantém conexões ativas na view `pg_stat_activity`. Não as confunda com tráfego da aplicação — elas aparecerão com `application_name = 'postgres_exporter'`.
