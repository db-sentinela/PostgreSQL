# 01 — Arquitetura e Instalação do PMM

## O que é

O **Percona Monitoring and Management (PMM)** é uma plataforma open source completa de
observabilidade para bancos de dados open source (PostgreSQL, MySQL, MariaDB, MongoDB).

Na versão 3, o PMM consolida as melhores tecnologias de telemetria moderna em um container unificado:
- **Grafana 12:** Camada de visualização rica com dezenas de dashboards pré-construídos.
- **VictoriaMetrics:** Banco de séries temporais de altíssima compactação e performance para métricas de infraestrutura e do PostgreSQL.
- **ClickHouse:** Banco colunar analítico ultrarrápido para armazenar dados de **Query Analytics (QAN)**.
- **pmm-agent / vmagent:** Agentes instalados nos servidores de banco de dados que coletam e empurram (*push*) dados com segurança criptografada para o servidor.

---

## Por que importa

- **Monitoramento 3 em 1:** Elimina a necessidade de manter três ferramentas separadas (Prometheus/Grafana para métricas, ferramentas de APM para queries e scripts bash para hardware).
- **Sem gargalo de rede (Arquitetura Push):** Em vez do servidor ficar sondando (*pull*) centenas de instâncias abrindo conexões, os agentes locais comprimem e enviam as métricas de forma eficiente para o VictoriaMetrics.

---

## Como reproduzir

### 1. Instalar o PMM Server (na VM de Monitoramento: `192.168.1.16`)

Crie o volume de dados para persistência e inicie o container oficial:

```bash
# Criar volume persistente
docker volume create pmm-data

# Executar o container na porta 8443 (HTTPS)
docker run -d \
  --name pmm-server \
  --restart always \
  -p 8443:8443 \
  -v pmm-data:/srv \
  -e ENABLE_BACKUP_MANAGEMENT=1 \
  percona/pmm-server:3
```

### 2. Instalar o `pmm-client` nos nós do PostgreSQL (`db-01`, `db-02`, `db-03`)

Nos servidores Fedora/RHEL/CentOS:

```bash
# Instalar o repositório oficial da Percona
dnf install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
percona-release enable pmm2-client release

# Instalar o cliente
dnf install -y pmm2-client
```

### 3. Registrar o nó cliente no Servidor PMM

Execute em cada nó do cluster indicando o IP do PMM Server e a senha administrativa:

```bash
pmm-admin config \
  --server-insecure-tls \
  --server-url=https://admin:Admin#2026@192.168.1.16:8443
```

---

## O que você vê (saída real)

### 1. Container do PMM Server saudável:

```
NAMES        STATUS                   PORTS
pmm-server   Up 25 minutes (healthy)  8080/tcp, 0.0.0.0:8443->8443/tcp, [::]:8443->8443/tcp
```

### 2. Validação da conexão do nó via `pmm-admin status`:

```
Agent ID : 5360897a-36d0-4642-972f-2429ae7fff07
Node ID  : 2504bec7-8514-4c8f-8f01-aa86ae49f6de
Node name: postgres-db-01

PMM Server:
	URL    : https://192.168.1.16:8443/
	Version: 3.10.0

PMM Client:
	Connected        : true
	Time drift       : -576.515µs
	Latency          : 1.021409ms
	Connection uptime: 100
	pmm-admin version: 3.10.0
	pmm-agent version: 3.10.0
```

`Connected: true` com latência de ~1ms confirma a comunicação TLS operacional entre a VM do banco e o servidor de observabilidade.

---

## Lições

- O `pmm-admin config` apenas conecta o **nó operacional** (sistema operacional e hardware) ao servidor. O monitoramento de cada banco de dados é adicionado como um serviço secundário (`pmm-admin add postgresql`).
- O `node_exporter` é iniciado automaticamente pelo `pmm-agent`, garantindo métricas de CPU, memória RAM, I/O de disco e rede imediatamente após o registro do nó.
- O volume Docker `pmm-data:/srv` guarda tanto a base do ClickHouse quanto do VictoriaMetrics e os dashboards do Grafana. Se o container for recriado, os dados históricos não são perdidos.

---

## Pegadinhas

- **Certificados autoassinados:** Como o PMM Server vem por padrão com certificado TLS autoassinado, **sempre** inclua a flag `--server-insecure-tls` nos comandos do `pmm-admin`, caso contrário o handshake TLS falhará com `x509: certificate signed by unknown authority`.
- **Divergência de relógio (*Time drift*):** Se o relógio do servidor PostgreSQL estiver dessincronizado com o PMM Server em mais de alguns segundos, as métricas serão descartadas pelo VictoriaMetrics por estarem "no passado" ou "no futuro". Garanta o serviço `chronyd` ou `systemd-timesyncd` rodando em todas as VMs.
- **Portas locais efêmeras:** O `pmm-agent` aloca portas locais (ex: `42000` a `42005`) para comunicação entre exporters e agentes. Se você tiver firewall local restritivo no nó (iptables/firewalld), certifique-se de liberar o tráfego loopback (`lo`).
