# PostgreSQL HA com Patroni — Fedora 42 (homologação em lab)

Documento único e reproduzível para implantar **alta disponibilidade** do
PostgreSQL 18 em 3 nós, com **Patroni + etcd + HAProxy + Keepalived (VIP)**.
Escrito a partir de uma execução real em laboratório (Oracle VirtualBox), com as
correções que apareceram no caminho registradas na seção
[Problemas encontrados e soluções](#15-problemas-encontrados-e-soluções).

> **Validado em:** Fedora Linux 42 Server · PostgreSQL **18.4** (PGDG) ·
> etcd **3.5.13** · Patroni 4.x · 3 VMs com 8 GB RAM.

---

## 1. Topologia e parâmetros

| Nó             | Hostname        | IP            | Papel inicial |
|----------------|-----------------|---------------|---------------|
| 1              | postgres-db-01  | 192.168.1.15  | Primary       |
| 2              | postgres-db-02  | 192.168.1.17  | Replica       |
| 3              | postgres-db-03  | 192.168.1.18  | Replica       |

| Item                         | Valor                                  |
|------------------------------|----------------------------------------|
| Cluster (scope)              | `pgha`                                 |
| VIP (endpoint de escrita)    | `192.168.1.10`                         |
| Interface de rede            | `enp0s3`                               |
| PostgreSQL                   | `5432` · binários em `/usr/pgsql-18/bin` |
| Patroni REST API             | `8008`                                 |
| etcd (client / peer)         | `2379` / `2380` · token `etcd-pgha-token` |
| HAProxy escrita / leitura / stats | `5000` / `5001` / `7000`          |
| Keepalived VRID              | `51`                                   |
| Replicação                   | assíncrona (síncrona na seção 11)      |
| Anti split-brain             | watchdog `softdog` + quorum etcd (2/3) |

**Usuários do PostgreSQL (senhas de LAB — 🔒 troque em produção):**

| Usuário       | Função        | Senha (lab)    |
|---------------|---------------|----------------|
| `postgres`    | superusuário  | `PgSuper#2026` |
| `replicator`  | replicação    | `Repl#2026`    |
| `rewind_user` | pg_rewind     | `Rewind#2026`  |
| `admin`       | admin de apps | `Admin#2026`   |

```
            APLICACOES
                |
                v
     VIP 192.168.1.10 (Keepalived)
                |
      HAProxy :5000 (write) / :5001 (read)
                |
   +------------+------------+
   v            v            v
 db-01        db-02        db-03     (PostgreSQL 18.4 + Patroni)
 etcd m1      etcd m2      etcd m3   (quorum 2 de 3)
```

> Todos os comandos rodam como **root**. Onde disser "**nos 3 nós**", repita em
> db-01, db-02 e db-03.

---

## 2. Pré-requisitos (nos 3 nós)

**2.1 `/etc/hosts`** — use um marcador para a inserção ser idempotente (sem isso,
o `grep` pode casar com a linha `127.0.0.1 <hostname>` e pular a inclusão):

```bash
grep -q '# patroni-cluster' /etc/hosts || cat >> /etc/hosts <<'EOF'
# patroni-cluster
192.168.1.15  postgres-db-01
192.168.1.17  postgres-db-02
192.168.1.18  postgres-db-03
EOF
```

**2.2 Relógio sincronizado** (crítico para etcd/Patroni):

```bash
dnf install -y chrony
systemctl enable --now chronyd
timedatectl set-ntp true
```

**2.3 SELinux → permissive (simplificação de lab):**

```bash
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```
> 🔒 **Produção:** mantenha `enforcing` e aplique contextos/políticas adequadas.

**2.4 Firewall** (firewalld ativo por padrão no Fedora Server):

```bash
for p in 5432 8008 2379 2380 5000 5001 7000; do firewall-cmd --permanent --add-port=$p/tcp; done
firewall-cmd --permanent --add-rich-rule='rule protocol value="vrrp" accept'
firewall-cmd --reload
```

---

## 3. PostgreSQL 18 via repositório PGDG (nos 3 nós)

O Fedora 42 entrega apenas **PostgreSQL 16** no repo nativo. Para o **18.4** use o
PGDG. O RPM automático `pgdg-fedora-repo` **não existe** para o Fedora 42, então
declaramos o repositório manualmente (o caminho de pacotes 18/fedora-42 existe):

```bash
# chave GPG oficial do PGDG para Fedora
rpm --import https://download.postgresql.org/pub/repos/yum/keys/PGDG-RPM-GPG-KEY-Fedora

cat > /etc/yum.repos.d/pgdg18.repo <<'EOF'
[pgdg18]
name=PostgreSQL 18 for Fedora 42 - x86_64
baseurl=https://download.postgresql.org/pub/repos/yum/18/fedora/fedora-42-x86_64/
enabled=1
gpgcheck=1
gpgkey=https://download.postgresql.org/pub/repos/yum/keys/PGDG-RPM-GPG-KEY-Fedora
EOF

# se tinha o PG16 nativo instalado, remova para não confundir binários:
dnf remove -y postgresql-server postgresql postgresql-contrib 2>/dev/null || true

dnf install -y postgresql18-server postgresql18-contrib
/usr/pgsql-18/bin/postgres --version      # -> PostgreSQL 18.4
```

> ⚠️ **Não** rode `initdb` nem o `postgresql-18-setup`. Quem inicializa e sobe o
> PostgreSQL é o **Patroni**.

---

## 4. etcd (nos 3 nós)

O etcd **está no repo do Fedora** (`etcd 3.5.13`); não precisa de binário externo.
A unit do Fedora lê variáveis `ETCD_*` do **env-file `/etc/etcd/etcd.conf`**
(não usa `--config-file`).

```bash
dnf install -y etcd
```

**Config por nó** — grave `/etc/etcd/etcd.conf` substituindo o IP do próprio nó.
Exemplo do **db-01** (nos outros, troque `db-01`/`192.168.1.15`):

```ini
# /etc/etcd/etcd.conf  (db-01)
ETCD_NAME=db-01
ETCD_DATA_DIR=/var/lib/etcd
ETCD_LISTEN_PEER_URLS=http://192.168.1.15:2380
ETCD_LISTEN_CLIENT_URLS=http://192.168.1.15:2379,http://127.0.0.1:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=http://192.168.1.15:2380
ETCD_ADVERTISE_CLIENT_URLS=http://192.168.1.15:2379
ETCD_INITIAL_CLUSTER=db-01=http://192.168.1.15:2380,db-02=http://192.168.1.17:2380,db-03=http://192.168.1.18:2380
ETCD_INITIAL_CLUSTER_TOKEN=etcd-pgha-token
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_ENABLE_V2=false
```

**Subir o cluster.** Use `--no-block`: com `INITIAL_CLUSTER_STATE=new` o primeiro
nó fica esperando os peers, e um `start` bloqueante travaria o laço:

```bash
systemctl daemon-reload
systemctl enable etcd
systemctl start --no-block etcd        # nos 3 nós
```

**Validar** (de qualquer nó):

```bash
EP=192.168.1.15:2379,192.168.1.17:2379,192.168.1.18:2379
etcdctl --endpoints=$EP endpoint health
etcdctl --endpoints=$EP member list -w table
```
Esperado: 3 membros `started`, todos `healthy`, um `IS LEADER=true`.

---

## 5. Patroni via venv (nos 3 nós)

O Patroni **não está** empacotado no Fedora 42. Instale num virtualenv (evita o
bloqueio PEP 668 de "externally-managed" e mantém versões isoladas). O
`psycopg[binary]` tem wheel pronto e dispensa compilar:

```bash
dnf install -y python3-pip gcc python3-devel
python3 -m venv /opt/patroni-venv
/opt/patroni-venv/bin/pip install --upgrade pip wheel
/opt/patroni-venv/bin/pip install "patroni[etcd3]" "psycopg[binary]"
/opt/patroni-venv/bin/patroni --version
```

**Preparar o PostgreSQL para ser gerenciado só pelo Patroni:**

```bash
systemctl disable --now postgresql-18 2>/dev/null || true
systemctl mask postgresql-18                       # evita start manual acidental
install -d -o postgres -g postgres -m 0700 /var/lib/pgsql/18/data   # vazio!
```

**Config do Patroni** — grave `/etc/patroni/patroni.yml` por nó. Exemplo do
**db-01** (nos outros troque `name`, e os IPs de `restapi` e `postgresql`):

```yaml
# /etc/patroni/patroni.yml  (db-01)
scope: pgha
namespace: /service/
name: db-01

restapi:
  listen: 192.168.1.15:8008
  connect_address: 192.168.1.15:8008

etcd3:
  hosts:
    - 192.168.1.15:2379
    - 192.168.1.17:2379
    - 192.168.1.18:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    synchronous_mode: false
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        wal_log_hints: "on"
        max_connections: 200
        max_wal_senders: 10
        max_replication_slots: 10
        wal_keep_size: 256MB
        shared_buffers: 512MB
        effective_cache_size: 1536MB
        maintenance_work_mem: 128MB
        hot_standby_feedback: "on"
        password_encryption: scram-sha-256
  initdb:
    - encoding: UTF8
    - data-checksums
  pg_hba:
    - local   all             all                                   peer
    - host    all             all                127.0.0.1/32       scram-sha-256
    - host    all             all                192.168.1.0/24     scram-sha-256
    - host    replication     replicator         192.168.1.0/24     scram-sha-256
  users:
    admin:
      password: Admin#2026
      options: [createrole, createdb]

postgresql:
  # ATENCAO: o formato é "host1,host2:porta" (porta só no fim, uma vez),
  # NÃO "host:porta,host:porta".
  listen: 192.168.1.15,127.0.0.1:5432
  connect_address: 192.168.1.15:5432
  # use_unix_socket evita o Patroni conectar via TCP em localhost->::1 (IPv6),
  # onde o PG não escuta. Sem isso o bootstrap falha nesta distro.
  use_unix_socket: true
  data_dir: /var/lib/pgsql/18/data
  bin_dir: /usr/pgsql-18/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: Repl#2026
    superuser:
      username: postgres
      password: PgSuper#2026
    rewind:
      username: rewind_user
      password: Rewind#2026
  parameters:
    password_encryption: scram-sha-256

watchdog:
  mode: automatic          # troque p/ required após habilitar o softdog (seção 8)
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

```bash
chown postgres:postgres /etc/patroni/patroni.yml
chmod 0600 /etc/patroni/patroni.yml     # contém senhas
```

**Serviço systemd** — `/etc/systemd/system/patroni.service` (igual nos 3):

```ini
[Unit]
Description=Patroni PostgreSQL HA
After=network-online.target etcd.service
Wants=network-online.target

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/opt/patroni-venv/bin/patroni /etc/patroni/patroni.yml
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
TimeoutSec=30
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Subir — primeiro o db-01** (ele faz o `initdb` e vira primary); depois os outros
(eles clonam do primary):

```bash
systemctl daemon-reload
# db-01:
systemctl enable --now patroni
journalctl -u patroni -f           # aguarde "... promoted" / "is the leader"
# db-02 e db-03:
systemctl enable --now patroni
```

**Validar:**

```bash
/opt/patroni-venv/bin/patronictl -c /etc/patroni/patroni.yml list
```

---

## 6. HAProxy (nos 3 nós)

```bash
dnf install -y haproxy
```

`/etc/haproxy/haproxy.cfg` (igual nos 3) — roteia escrita ao primary (`/primary`)
e balanceia leitura nas replicas (`/replica`), checando a REST API do Patroni:

```
global
    maxconn 1000
defaults
    mode    tcp
    retries 2
    timeout client  30m
    timeout server  30m
    timeout connect 4s
    timeout check   5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /
    stats refresh 5s

listen postgres_write
    bind *:5000
    option httpchk GET /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server db-01 192.168.1.15:5432 maxconn 100 check port 8008
    server db-02 192.168.1.17:5432 maxconn 100 check port 8008
    server db-03 192.168.1.18:5432 maxconn 100 check port 8008

listen postgres_read
    bind *:5001
    balance roundrobin
    option httpchk GET /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server db-01 192.168.1.15:5432 maxconn 100 check port 8008
    server db-02 192.168.1.17:5432 maxconn 100 check port 8008
    server db-03 192.168.1.18:5432 maxconn 100 check port 8008
```

```bash
systemctl enable --now haproxy
# painel: http://<ip-do-no>:7000/
```

---

## 7. Keepalived / VIP

```bash
dnf install -y keepalived psmisc
```

Script de verificação `/etc/keepalived/chk_haproxy.sh` (nos 3):

```bash
#!/usr/bin/env bash
/usr/bin/pidof haproxy >/dev/null 2>&1
```
```bash
chmod +x /etc/keepalived/chk_haproxy.sh
```

`/etc/keepalived/keepalived.conf` — **db-01 = MASTER** (priority 110); db-02 e
db-03 = `BACKUP` com priority 105 e 100. Exemplo db-01:

```
global_defs {
    router_id db-01
    enable_script_security
    script_user root
}
vrrp_script chk_haproxy {
    script "/etc/keepalived/chk_haproxy.sh"
    interval 2
    weight 2
    fall 2
    rise 2
}
vrrp_instance VI_PG {
    state MASTER
    interface enp0s3
    virtual_router_id 51
    priority 110
    advert_int 1
    authentication { auth_type PASS; auth_pass pgha2026 }
    virtual_ipaddress { 192.168.1.10/24 }
    track_script { chk_haproxy }
}
```

```bash
systemctl enable --now keepalived
ip a | grep 192.168.1.10        # VIP aparece em UM nó
```

---

## 8. Watchdog — anti split-brain (nos 3 nós)

```bash
modprobe softdog
echo softdog > /etc/modules-load.d/softdog.conf
cat > /etc/udev/rules.d/60-watchdog.rules <<'EOF'
KERNEL=="watchdog", OWNER="postgres", MODE="0600"
EOF
udevadm control --reload-rules && udevadm trigger
```
Depois, em `patroni.yml`, troque `mode: automatic` por `mode: required` e
`systemctl restart patroni`.
> ⚠️ Com `softdog`, se o Patroni travar, **a VM reinicia** (fencing esperado).

---

## 9. Replicação síncrona (opcional)

```bash
/opt/patroni-venv/bin/patronictl -c /etc/patroni/patroni.yml edit-config
# defina:  synchronous_mode: true   (e opcional synchronous_node_count: 1)
```

---

## 10. Testes de homologação

Conecte **pelo VIP**:

```bash
# escrita (sempre no primary) -> pg_is_in_recovery() = f
psql "host=192.168.1.10 port=5000 user=postgres password=PgSuper#2026 dbname=postgres" \
  -c "select inet_server_addr(), pg_is_in_recovery();"
# leitura (replicas) -> t
psql "host=192.168.1.10 port=5001 user=postgres password=PgSuper#2026 dbname=postgres" \
  -c "select inet_server_addr(), pg_is_in_recovery();"
```

- **Switchover:** `patronictl switchover pgha` → novo Leader assume, sem perda.
- **Failover:** desligue a VM do primary → em segundos uma replica vira Leader e a
  escrita no VIP continua; ao religar, o nó volta como replica (pg_rewind).
- **Perda de quorum (2 nós caídos):** etcd sem quorum, Patroni **não promove**
  (seguro, evita split-brain).
- **Queda do nó com o VIP:** Keepalived move o VIP para outro nó.

### Evidências da homologação (execução real neste lab)

| Teste | Resultado |
|-------|-----------|
| Cluster inicial | db-01 Leader; db-02/db-03 `streaming`, lag 0 (TL 1) |
| WRITE no VIP:5000 | roteou para o primary `192.168.1.15`, `pg_is_in_recovery()=f` |
| READ no VIP:5001 | roteou para a replica `192.168.1.17`, `pg_is_in_recovery()=t` |
| Switchover db-01→db-02 | `Successfully switched over to "db-02"`; WRITE passou a ir ao `.17` (TL 2) |
| Failover (stop no líder db-02) | db-01 promovido **automaticamente** (TL 3); WRITE voltou ao `.15` |
| Reintegração do db-02 | voltou como `Replica streaming`, lag 0 |

---

## 11. Operação do dia a dia

```bash
P="/opt/patroni-venv/bin/patronictl -c /etc/patroni/patroni.yml"
$P list                       # estado do cluster
$P switchover pgha            # troca de primary controlada
$P restart pgha db-02         # restart de um nó
$P reinit pgha db-03          # recria uma replica do zero
$P pause ; $P resume          # janela de manutenção (pausa o HA)
```

---

## 12. Troubleshooting rápido

| Sintoma | Ação |
|---------|------|
| Patroni não sobe o 1º nó | `bin_dir` errado (confirme `/usr/pgsql-18/bin`) ou `data_dir` não vazio. |
| Replica não entra em `streaming` | senha do `replicator` divergente ou `pg_hba` bloqueando; veja `journalctl -u patroni`. |
| etcd `unhealthy` | relógio dessincronizado, firewall em 2379/2380, ou env-file incorreto. |
| VIP não aparece | nome de `interface` errado no keepalived; VRRP bloqueado no firewall. |
| HAProxy marca tudo `DOWN` | REST API 8008 inacessível entre os nós. |

---
