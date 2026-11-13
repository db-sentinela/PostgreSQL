# Observabilidade no PostgreSQL com Percona Monitoring and Management (PMM)

Base de conhecimento **didática e prática** sobre o monitoramento completo de infraestrutura,
instâncias e clusters PostgreSQL utilizando **Percona Monitoring and Management (PMM 3)**:
**arquitetura da stack, instalação do servidor e agentes, monitoramento de instâncias single,
análise aprofundada de consultas com Query Analytics (QAN / `pg_stat_statements`), integração
nativa com a telemetria do Patroni (HA) e resolução de incidentes comuns**.

Cada procedimento foi **homologado e testado de verdade** no laboratório,
seguindo a estrutura: **o que é → por que importa → como reproduzir → o que você vê
(saída real) → lições → pegadinhas**.

> Laboratório:
> - **Servidor PMM:** `192.168.1.16` (`pmm-mon-01`), porta web `8443` (PMM Server 3.10.0 em Docker).
> - **Cluster PostgreSQL:** `192.168.1.15` (`db-01`), `192.168.1.17` (`db-02`), `192.168.1.18` (`db-03`) gerenciados por Patroni 4.1.5 e etcd.

---

## Índice

| # | Documento | O que você aprende |
|---|-----------|--------------------|
| 01 | [Arquitetura e Instalação do PMM](01-arquitetura-e-instalacao-pmm.md) | Componentes do PMM (Grafana, VictoriaMetrics, ClickHouse), deploy via Docker e setup do `pmm-client` |
| 02 | [Monitoramento PostgreSQL Single Instance](02-monitoramento-postgresql-single-instance.md) | Criação da role `pmm` com menor privilégio (`pg_monitor`), exporters e registro no `pmm-admin` |
| 03 | [Query Analytics (QAN) e pg_stat_statements](03-query-analytics-qan-e-pg_stat_statements.md) | Captura de queries lentas, tempo de CPU/IO, planos de execução e diferenças entre `pg_stat_statements` e `pg_stat_monitor` |
| 04 | [Monitoramento de Cluster Patroni (HA)](04-monitoramento-cluster-patroni-alta-disponibilidade.md) | Exposição de `/metrics` na porta 8008, ajuste de bind `0.0.0.0`, cadastro de exporters externos e dashboard de HA |
| 05 | [Dashboards, Métricas e Troubleshooting](05-dashboards-metricas-e-troubleshooting.md) | Navegação em dashboards, pegadinha do seletor `pmm-server-postgresql`, reset de senha de admin e diagnóstico de agentes |

---

## Topologia de Observabilidade

```
      [ PMM Server - 192.168.1.16 ]
      ├── Grafana (Dashboard UI :8443)
      ├── VictoriaMetrics (Série temporal :9090)
      ├── ClickHouse (Query Analytics / QAN :8123)
      └── Nomad & PMM Managed (Orquestração de Agentes)
                    ▲
                    │ Push de Métricas / QAN via TLS
                    │
      [ Nós PostgreSQL: db-01, db-02, db-03 ]
      ├── pmm-agent (Coordenador local de exporters)
      ├── node_exporter (CPU, Memória, Disco, Rede do Linux)
      ├── postgres_exporter (Métricas internas do PG :5432)
      ├── postgresql_pgstatements_agent (Estatísticas de queries)
      └── external-exporter (Telemetria nativa do Patroni :8008)
```

---

## Mapa de Portas e Serviços de Monitoramento

| Porta | Protocolo | Serviço | Descrição |
|-------|-----------|---------|-----------|
| `8443` | HTTPS | PMM Server / Nginx | Acesso à interface web do Grafana e APIs |
| `8080` | HTTP | PMM Server / Nginx | Acesso HTTP sem TLS (quando habilitado) |
| `42000` | TCP | Nomad Agent | Agente de tarefas do PMM no nó monitorado |
| `42001` | TCP | postgres_exporter | Exporter de métricas do banco de dados |
| `42002` | TCP | vmagent | Agente de envio de métricas para o VictoriaMetrics |
| `42003` | TCP | node_exporter | Exporter de métricas de hardware/SO |
| `8008` | HTTP | Patroni REST API | Endpoint `/metrics` com métricas nativas de HA |
| `5432` | TCP | PostgreSQL | Porta de conexão do banco monitorado |
