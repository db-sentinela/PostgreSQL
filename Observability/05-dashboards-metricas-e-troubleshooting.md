# 05 — Dashboards, Métricas e Troubleshooting no PMM

## O que é

O **Percona Monitoring and Management (PMM)** entrega um catálogo de dashboards prontos no Grafana
para diferentes níveis de abstração operacional:
- **`PostgreSQL Overview`:** Visão executiva macro (conexões, TPS, CPU, I/O e saturação de cache);
- **`PostgreSQL Summary`:** Métricas operacionais de tabelas, índices, tuplas vivas x mortas (*dead tuples*) e bloat;
- **`PostgreSQL Patroni Details`:** Painel de controle de alta disponibilidade do cluster Patroni;
- **`Query Analytics (QAN)`:** Análise analítica de gargalos e planos de execução de consultas SQL.

Este documento ensina como navegar corretamente nesses dashboards e como solucionar os problemas
mais comuns enfrentados por equipes de infraestrutura e DBAs.

---

## Por que importa

- **Evitar falsos alarmes:** Um painel exibindo `No data` ou `N/A` frequentemente não significa que o banco caiu, mas sim que um filtro no topo do Grafana está apontando para o serviço errado.
- **Rapidez no restabelecimento:** Saber onde olhar os logs locais do `pmm-agent` e do container do PMM reduz o tempo de diagnóstico de problemas de monitoramento de horas para minutos.

---

## Como navegar nos Dashboards

### 1. Seletores de Contexto (Barra Superior)

Todo dashboard no PMM possui filtros em cascata no topo da página:

```
[ Environment: All ] ──► [ Cluster: pgha ] ──► [ Service Name: patroni-db-01 ] ──► [ Scope: pgha ]
```

- **`Environment`**: Agrupa ambientes (ex: `production`, `staging`, `homelab`).
- **`Cluster`**: Nome do cluster lógico (definido pela flag `--cluster=pgha`).
- **`Service Name`**: Nome específico da instância monitorada:
  - Para métricas de banco de dados (tabelas, queries): selecione `postgresql-db-01`, `postgresql-db-02` ou `postgresql-db-03`.
  - Para métricas do cluster Patroni (HA): selecione `patroni-db-01`, `patroni-db-02` ou `patroni-db-03`.

---

## O que você vê (Casos Reais do Laboratório)

### Caso 1: O "falso vazio" no dashboard `PostgreSQL Patroni Details`

**O que aparecia na tela:**
```
Patroni Version: No data
Patroni DCS Last Seen: N/A
Patroni Leader: N/A
Patroni Replica: N/A
Patroni Nodes State: Data does not have a time field
```

**Causa Raiz:**
No topo do dashboard, o seletor **`Service Name`** estava marcado como **`pmm-server-postgresql`**.
Como o `pmm-server-postgresql` é o banco local interno do container do PMM e não utiliza Patroni, todos os painéis ficaram sem dados.

**Solução:**
Alterar o seletor **`Service Name`** para `patroni-db-01` e o **`Scope Name`** para `pgha`. Imediatamente os painéis carregam com os dados reais do cluster:
```
Patroni Version: 4.1.5
Patroni DCS Last Seen: Agora (0s atrás)
Patroni Leader: 1 (db-01)
Patroni Replica: 0 (db-01 é Leader)
Patroni Nodes State: 3 nós ativos
```

---

## Guia de Resolução de Problemas (Troubleshooting)

### 1. Reset da Senha de Administrador do PMM

Se a senha do usuário `admin` do Grafana for esquecida ou perdida, você pode redefini-la diretamente no container via SSH na VM do PMM Server (`192.168.1.16`):

```bash
docker exec -t pmm-server change-admin-password 'NovaSenha#2026'
```

Saída esperada:
```
Admin password changed successfully ✔
```

### 2. Bloqueio Temporário por Tentativas Incorretas (*Brute Force Protection*)

Se muitas tentativas com senha incorreta forem feitas, o Grafana bloqueia temporariamente o usuário:

```
error="[password-auth.failed] too many consecutive incorrect login attempts for user - login for user temporarily blocked"
```

**Como destravar:**
Basta reiniciar o processo do Grafana dentro do container para limpar o contador de bloqueio em memória:

```bash
docker exec pmm-server supervisorctl restart grafana
```

### 3. Exporters no status `Waiting` ou `Unknown`

Ao rodar `pmm-admin list`, algum agente aparece como `Waiting`:

```
Agent type                           Status     Agent ID
postgresql_pgstatements_agent        Waiting    2a03cf5e...
```

**Passos para diagnóstico:**
1. Verifique o log do agente no servidor do banco:
   ```bash
   journalctl -u pmm-agent --no-pager -n 50
   ```
2. Se houver erro de senha (`password authentication failed for user "pmm"`):
   - Crie a role no PostgreSQL com a senha correta (`GRANT pg_monitor TO pmm;`).
3. Se o erro for de extensão ausente (`pg_stat_monitor`):
   - Remova o serviço e recadastre utilizando `--query-source=pgstatements`.
4. Reinicie o agente para forçar o restabelecimento imediato:
   ```bash
   systemctl restart pmm-agent
   ```

---

## Lições

- O PMM é resiliente: se a rede cair ou o servidor de monitoramento for reiniciado, o `vmagent` local nos nós armazena as métricas em buffer em disco temporário e as envia assim que a conexão voltar.
- `pmm-admin list` é a ferramenta definitiva de diagnóstico no nó do banco de dados: se todos os agentes estiverem no estado **`Running`**, os dados estão sendo enviados com sucesso.

---

## Pegadinhas

- **Filtro de tempo do Grafana no canto superior direito:** Se o seletor de período estiver configurado para "Last 12 hours" em um cluster que acabou de ser iniciado há 10 minutos, os gráficos de linha parecerão quase vazios. Ajuste para **"Last 15 minutes"** ou **"Last 30 minutes"** para enxergar a atividade em tempo real.
- **Cache agressivo do navegador:** Após alterar a senha no container ou recadastrar um serviço, o navegador pode manter tokens JWT ou cookies expirados. Se a página continuar dando erro 401 ou os seletores não atualizarem, abra uma aba anônima ou limpe os cookies do endereço `192.168.1.16:8443`.
