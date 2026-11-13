# 04 — TLS/SSL nas Conexões e Replicação

## O que é

O **TLS (Transport Layer Security)**, frequentemente chamado de SSL, é o protocolo
que provê **criptografia em trânsito** entre as aplicações clientes e o servidor PostgreSQL,
bem como entre os nós do cluster Patroni durante a replicação de streaming.

Sem TLS, o tráfego de rede (consultas, resultados, dados sensíveis de clientes e até mesmo
senhas transmitidas por métodos vulneráveis) circula em texto plano, passível de interceptação
(*sniffing*) por qualquer dispositivo no mesmo segmento de rede.

---

## Por que importa

- **Ataques Man-In-The-Middle (MITM):** Em redes corporativas ou ambientes de nuvem compartilhados, um atacante com acesso a um switch, roteador ou host comprometido pode espionar o tráfego do banco.
- **Conformidade (LGPD, GDPR, PCI-DSS):** Praticamente todas as regulamentações de proteção de dados exigem explicitamente criptografia ponta a ponta para dados em trânsito.

---

## Como reproduzir

### 1. Gerar certificados e chaves nos nós do cluster

No ambiente Linux de cada VM (`db-01`, `db-02`, `db-03`), gere o par de chaves e o certificado autoassinado (ou utilize os certificados da sua PKI interna):

```bash
openssl req -new -x509 -days 365 -nodes -text \
  -out /var/lib/pgsql/18/data/server.crt \
  -keyout /var/lib/pgsql/18/data/server.key \
  -subj "/CN=postgres-cluster"

chown postgres:postgres /var/lib/pgsql/18/data/server.crt /var/lib/pgsql/18/data/server.key
chmod 0600 /var/lib/pgsql/18/data/server.key
chmod 0644 /var/lib/pgsql/18/data/server.crt
```

> **Atenção:** Se a chave privada (`server.key`) tiver permissões permissivas (ex: `0644`), o PostgreSQL **recusa-se a iniciar** por segurança!

### 2. Ativar o SSL via Patroni DCS

Em um cluster Patroni, aplique a alteração no DCS para que todos os membros habilitem o SSL de forma homogênea:

```bash
patronictl -c /etc/patroni/patroni.yml edit-config pgha -s "postgresql.parameters.ssl=on" --force
patronictl -c /etc/patroni/patroni.yml reload pgha --force
```

### 3. Conectar exigindo criptografia

No cliente `psql` ou na string de conexão da sua aplicação:

```bash
PGPASSWORD='SenhaForte#2026' psql "host=192.168.1.15 user=sec_analista dbname=postgres sslmode=require"
```

Inspecione a view `pg_stat_ssl` da própria sessão:

```sql
SELECT pid, ssl, version, cipher, bits 
FROM pg_stat_ssl 
WHERE pid = pg_backend_pid();
```

---

## O que você vê (saída real)

### Conexão estabelecida com TLSv1.3 e cifra moderna:

```
 pid  | ssl | version |         cipher         | bits 
------+-----+---------+------------------------+------
 3924 | t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256
(1 row)
```

- **`ssl: t`**: Conexão 100% criptografada.
- **`version: TLSv1.3`**: O protocolo mais moderno e seguro disponível.
- **`cipher: TLS_AES_256_GCM_SHA384`**: Cifra autenticada (AEAD) de alta performance e segurança militar com chave de 256 bits.

---

## Modos de SSL no Cliente (`sslmode`)

A configuração do cliente é tão importante quanto a do servidor:

| `sslmode` | Comportamento | Segurança |
|-----------|---------------|-----------|
| `disable` | Nunca usa SSL (texto puro) | Inseguro |
| `allow` | Tenta sem SSL; se o servidor exigir, usa SSL | Inseguro |
| `prefer` | *(Padrão em drivers legados)* Tenta SSL; se falhar, conecta sem SSL | Vulnerável a downgrade |
| `require` | Exige SSL, mas não valida o certificado da máquina | Protege contra sniffing, não contra MITM |
| `verify-ca` | Exige SSL e valida se o certificado foi assinado pela CA confiável | Seguro |
| `verify-full` | Exige SSL, valida a CA **e** confere se o CN/SAN bate com o hostname | **Padrão ouro em produção** |

---

## Lições

- **Forçar SSL no `pg_hba`:** No `pg_hba`, declare as regras de rede como `hostssl` em vez de `host`. Conexões sem SSL serão sumariamente rejeitadas com `FATAL: no pg_hba.conf entry for host ... SSL off`.
- **`pg_stat_ssl` é o seu raio-X:** Use essa view para descobrir se alguma aplicação antiga ainda está conectando sem criptografia (`WHERE ssl = false`).
- **O overhead de TLS moderno é desprezível:** Com TLSv1.3 e instruções AES-NI nativas nas CPUs modernas, o impacto em CPU para dados criptografados é inferior a 1-2%.

---

## Pegadinhas

- **Permissões de arquivo (`0600`):** Se alguém rodar `chmod 777` ou `chmod 644` na chave privada `server.key`, o PostgreSQL não subirá com o erro `FATAL: private key file "server.key" has group or world access`.
- **Falsa sensação de segurança com `sslmode=prefer`:** Se um invasor derrubar a negociação de SSL na rede (ataque de downgrade), clientes com `prefer` rebaixam silenciosamente a conexão para texto puro sem emitir erro! Use sempre `sslmode=require` ou `verify-full`.
- **Replicação com SSL:** Para que a replicação de streaming entre os nós do Patroni use SSL, o parâmetro `pg_hba` deve conter `hostssl replication replicator ...` e os nós devem possuir os certificados configurados.
