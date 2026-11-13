# 03 — Query Analytics (QAN) e pg_stat_statements

## O que é

O **Query Analytics (QAN)** é a ferramenta de APM (Application Performance Monitoring)
integrada ao PMM para análise profunda de performance de consultas SQL.

Em vez de exibir apenas gráficos de "CPU a 90%" ou "I/O alto", o QAN responde à pergunta que todo
DBA e desenvolvedor precisa saber: **qual query específica está consumindo 80% dos recursos do banco?**

O QAN funciona coletando dados normalizados (queries parametrizadas) da view nativa
**`pg_stat_statements`** e enviando-os ao banco colunar **ClickHouse** do PMM Server.

---

## Por que importa

- **Tuning orientado a impacto real:** Otimizar uma query que roda 1 vez ao dia e demora 2 segundos tem impacto nulo. Otimizar uma query que roda 50.000 vezes por minuto e demora 10ms libera metade da CPU do servidor. O QAN ordena as consultas pelo **tempo total de execução** (*Load / Total Time*).
- **Sem overhead de log:** Enquanto auditar queries via log (`log_statement = all` ou `log_min_duration_statement = 0`) estrangula o disco do servidor, o `pg_stat_statements` acumula estatísticas em memória com overhead inferior a 1-2%.

---

## Como reproduzir

### 1. Habilitar `pg_stat_statements` no cluster

A biblioteca deve ser carregada no início do PostgreSQL via `shared_preload_libraries`.

Em cluster Patroni:
```bash
patronictl -c /etc/patroni/patroni.yml edit-config pgha \
  -s "postgresql.parameters.shared_preload_libraries=pg_stat_statements" \
  --force

# Requer reinicialização para carregar bibliotecas compartilhadas
patronictl -c /etc/patroni/patroni.yml restart pgha --force
```

### 2. Criar a extensão no banco de dados

Conecte no banco como superusuário:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

### 3. Registrar o serviço no PMM especificando a fonte de consultas

Ao cadastrar o banco de dados no `pmm-admin`, utilize explicitamente a flag **`--query-source=pgstatements`**:

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

### 1. Agente de consultas em execução via `pmm-admin list`:

```
Agent type                           Status           Metrics Mode        Agent ID                                    Service ID                                  Port
postgresql_pgstatements_agent        Running                              2a03cf5e-e7fb-4af7-8191-67d926875e43        872f0645-294c-4c0e-bb3d-e3e69ae2c7c2        0 
```

O status **`Running`** confirma que o agente está lendo periodicamente a view `pg_stat_statements` e enviando os blocos analíticos para o ClickHouse do PMM.

### 2. No painel do PMM Web (`https://192.168.1.16:8443`):

No menu lateral esquerdo, clique em **Query Analytics (QAN)**:
- Você verá um gráfico de dispersão com o perfil de carga por banco e por aplicação;
- Uma tabela com as top queries ordenadas por **Load (% do tempo total do banco)**, **Calls/min**, **Latência média (Avg Latency)** e percentil 95 (P95);
- Ao clicar em qualquer query, o PMM exibe o SQL normalizado, o plano de execução (`EXPLAIN`) e o tempo gasto em CPU vs I/O de disco.

---

## `pg_stat_statements` x `pg_stat_monitor`

O PMM suporta dois mecanismos para capturar métricas de queries:

| Característica | `pg_stat_statements` | `pg_stat_monitor` |
|----------------|----------------------|-------------------|
| **Origem** | Oficial do PostgreSQL Global Development Group (PGDG) | Desenvolvido pela Percona |
| **Disponibilidade** | Nativo em 100% das instalações PostgreSQL | Requer pacotes específicos da Percona |
| **Compatibilidade** | Universal (PostgreSQL 10 até 18+) | Limitado às distribuições Percona Server for PostgreSQL |
| **Granularidade temporal** | Cumulativa (contadores globais desde o reset) | Em baldes temporais (*time buckets*) |
| **Recomendação no Lab** | **Padrão Obrigatório (`--query-source=pgstatements`)** | Não usar em pacotes PGDG upstream |

---

## Lições

- **Normalização de queries:** O `pg_stat_statements` converte `SELECT * FROM clientes WHERE id = 10` e `SELECT * FROM clientes WHERE id = 99` na mesma assinatura parametrizada: `SELECT * FROM clientes WHERE id = $1`. Isso permite agrupar todas as execuções da mesma query em uma única métrica consolidada.
- **Armazenamento no ClickHouse:** O ClickHouse comprime logs de queries em índices colunares. Isso permite pesquisar históricos de queries de semanas atrás em frações de segundo sem onerar o servidor PMM.

---

## Pegadinhas

- **A armadilha do `pg_stat_monitor` em distros upstream:** Se você rodar `pmm-admin add postgresql` com `--query-source=pgstatmonitor` (ou sem especificar, caso ele tente autodetectar) em um PostgreSQL padrão do PGDG, o agente **`postgresql_pgstatmonitor_agent` ficará eternamente no status `Waiting`** e o QAN nunca exibirá nenhuma query.
- **Como corrigir caso fique travado em `Waiting`:**
  ```bash
  # 1. Remover o serviço com configuração incorreta
  pmm-admin remove postgresql postgresql-db-03

  # 2. Re-adicionar especificando pgstatements
  pmm-admin add postgresql \
    --username=pmm \
    --password=S3nh4_F0rt3 \
    --query-source=pgstatements \
    --service-name=postgresql-db-03 \
    127.0.0.1:5432
  ```
- **Reset acidental de estatísticas:** A view `pg_stat_statements` mantém acumuladores contínuos. Se alguém rodar `SELECT pg_stat_statements_reset();`, os contadores voltam a zero e os gráficos do QAN sofrerão uma descontinuidade temporária.
