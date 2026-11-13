# Segurança no PostgreSQL

Base de conhecimento **didática e prática** sobre segurança ofensiva e defensiva
no PostgreSQL: **autenticação moderna (SCRAM), controle de acesso por rede (`pg_hba`),
princípio do menor privilégio (RBAC), Row-Level Security (RLS), criptografia em
trânsito (TLSv1.3), auditoria detalhada (`pgaudit`), criptografia de dados (`pgcrypto`)
e checklist de hardening**.

Cada documento foi **homologado e testado de verdade** no cluster de laboratório,
seguindo a estrutura: **o que é → por que importa → como reproduzir → o que você vê
(saída real) → lições → pegadinhas**.

> Laboratório: cluster `pgha` (3 nós Fedora 42, PostgreSQL 18.4, Patroni 4.1.5, etcd).
> VIP de acesso: `192.168.1.10`. Objetos de segurança no schema `seguranca`.

---

## Índice

| # | Documento | O que você aprende |
|---|-----------|--------------------|
| 01 | [Autenticação SCRAM e pg_hba](01-autenticacao-scram-e-pg_hba.md) | Por que aposentar `md5`/`trust`; regras de rede no Patroni e ordem de avaliação |
| 02 | [Roles, privilégios e Least Privilege](02-roles-privilegios-e-least-privilege.md) | Grupos de permissão (RO/RW), `DEFAULT PRIVILEGES`, revogar `PUBLIC` e matar o superuser de app |
| 03 | [Row-Level Security (RLS)](03-row-level-security-rls.md) | Isolamento multi-tenant real no banco; bypass de superusuário e `FORCE ROW LEVEL SECURITY` |
| 04 | [TLS/SSL nas conexões](04-tls-ssl-conexoes-e-replicacao.md) | Certificados, TLSv1.3, forçar `hostssl`, inspecionar ciphers na `pg_stat_ssl` |
| 05 | [pgaudit (Auditoria de Comandos)](05-pgaudit-auditoria-de-comandos.md) | Instalação no Fedora 42, `shared_preload_libraries`, auditoria de DDL, roles e escritas |
| 06 | [pgcrypto e proteção de dados sensíveis](06-pgcrypto-dados-sensiveis.md) | Hashing seguro de senhas com bcrypt (`gen_salt`), criptografia simétrica AES-256 (`pgp_sym`) |
| 07 | [Hardening e checklist de segurança](07-hardening-e-checklist-seguranca.md) | Configurações críticas de `postgresql.conf`, permissões de SO, portas e checklist para juniores |

---

## Metodologia: Defesa em Profundidade

Segurança de banco de dados não é uma barreira única, mas uma série de **camadas concêntricas**:

```
[ Camada Externa: Rede ]
     │  VIP / Firewall / pg_hba.conf (Quem pode tentar conectar?)
     ▼
[ Camada de Transporte ]
     │  TLSv1.3 / SSL obrigatório (Alguém pode farejar o tráfego?)
     ▼
[ Camada de Identidade ]
     │  SCRAM-SHA-256 (A senha é forte e o hash resiste a vazamentos?)
     ▼
[ Camada de Autorização ]
     │  RBAC / Menor Privilégio / DEFAULT PRIVILEGES (O que a role pode tocar?)
     ▼
[ Camada de Linha / Dado ]
     │  RLS / pgcrypto (Um tenant vê o dado do outro? O cartão está em claro?)
     ▼
[ Camada de Rastreabilidade ]
        pgaudit / Logs (Quem alterou ou tentou alterar o quê e quando?)
```

---

## Onde procurar (mapa de bolso do DBA)

| Pergunta | Onde olhar |
|----------|------------|
| Quem tem permissão de superuser? | `SELECT rolname FROM pg_roles WHERE rolsuper;` |
| Qual algoritmo de senha está ativo? | `SHOW password_encryption;` |
| Quais regras de rede estão carregadas? | `SELECT * FROM pg_hba_file_rules;` |
| As conexões atuais estão criptografadas? | `SELECT pid, ssl, version, cipher FROM pg_stat_ssl;` |
| Quais tabelas têm RLS ativo? | `SELECT relname, relrowsecurity FROM pg_class WHERE relrowsecurity;` |
| Onde estão os logs de auditoria? | Arquivo de log atual (`SELECT pg_current_logfile();`) |
| Extensões de segurança instaladas | `SELECT extname, extversion FROM pg_extension;` |

---

## Como aplicar configurações em cluster Patroni

Em um ambiente gerenciado por Patroni, **nunca altere `postgresql.conf` ou `pg_hba.conf` diretamente no disco**:

```bash
# Alterar parâmetros globais (ex: SSL ou bibliotecas de auditoria):
patronictl -c /etc/patroni/patroni.yml edit-config pgha -s "postgresql.parameters.ssl=on" --force

# Recarregar nós sem indisponibilidade:
patronictl -c /etc/patroni/patroni.yml reload pgha --force
```

Isso garante que todos os nós (Leader e Replicas) mantenham a mesma política de segurança e que as configurações sobrevivam a switchovers e failovers.
