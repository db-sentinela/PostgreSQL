# 07 — Hardening e Checklist de Segurança

## O que é

**Hardening** (endurecimento) é o processo de proteger um sistema reduzindo sua
superfície de vulnerabilidade. No PostgreSQL, isso envolve fechar portas desnecessárias,
restringir permissões no sistema operacional, aplicar parâmetros defensivos no `postgresql.conf`,
auditar conexões e eliminar privilégios excessivos.

Este documento consolida as diretrizes do **CIS PostgreSQL Benchmark** em um guia
direto e acionável para o dia a dia.

---

## Por que importa

- **A maioria dos ataques explora erros de configuração:** Mais de 80% das invasões a bancos de dados não exploram vulnerabilidades de dia zero (*zero-day*), mas sim credenciais fracas, permissões de arquivo abertas (`777`), superusuários expostos e ausência de logs.
- **Rápida detecção de intrusão:** Sem `log_connections` e um prefixo de log adequado, você não saberá quem invadiu seu banco, quando a invasão ocorreu ou de qual máquina ela partiu.

---

## Como auditar o ambiente

### 1. Auditar permissões de arquivos no Sistema Operacional

O diretório de dados (`PGDATA`) e as chaves SSL **devem** ser acessíveis única e exclusivamente pelo usuário `postgres`:

```bash
# O diretório de dados deve ter permissão 0700 (drwx------):
ls -ld /var/lib/pgsql/18/data

# A chave privada SSL deve ter permissão 0600 (-rw-------):
ls -l /var/lib/pgsql/18/data/server.key
```

### 2. Auditar Superusuários no Banco

Identifique quem possui superpoderes na instância. Em produção, **apenas o usuário `postgres`** deve ter `rolsuper`:

```sql
SELECT rolname, rolsuper, rolinherit, rolcreaterole, rolcreatedb 
FROM pg_roles 
WHERE rolsuper;
```

### 3. Auditar parâmetros essenciais de Hardening

Verifique se a instância possui os parâmetros de segurança ativos:

```sql
SELECT name, setting, source 
FROM pg_settings 
WHERE name IN (
    'password_encryption', 
    'ssl', 
    'log_connections', 
    'log_disconnections', 
    'shared_preload_libraries'
)
ORDER BY name;
```

### 4. Ajustar Parâmetros Recomendados via Patroni DCS

Para ativar o registro de logins e deslogins em todo o cluster:

```bash
patronictl -c /etc/patroni/patroni.yml edit-config pgha \
  -s "postgresql.parameters.log_connections=on" \
  -s "postgresql.parameters.log_disconnections=on" \
  -s "postgresql.parameters.log_line_prefix=%m [%p] %q%u@%d %a " \
  --force

patronictl -c /etc/patroni/patroni.yml reload pgha --force
```

- **`%m`**: Timestamp com milissegundos.
- **`%p`**: PID do processo.
- **`%u@%d`**: Usuário e banco de dados.
- **`%a`**: Nome da aplicação conectada.

---

## O que você vê (saída real)

### 1. Permissões de disco validadas:

```
drwx------. 20 postgres postgres 4096 Nov 13 17:57 /var/lib/pgsql/18/data
```

`drwx------` confirma que apenas o usuário `postgres` tem permissão de leitura, escrita e execução. Nenhum outro usuário do sistema operacional consegue ler arquivos do banco.

### 2. Apenas um superusuário no catálogo:

```
 rolname  | rolsuper | rolinherit | rolcreaterole | rolcreatedb 
----------+----------+------------+---------------+-------------
 postgres | t        | t          | t             | t
(1 row)
```

Nenhum usuário de aplicação ou estagiário possui atributos de superusuário indevidos.

### 3. Parâmetros de segurança auditados:

```
           name           |          setting           |       source       
--------------------------+----------------------------+--------------------
 log_connections          | on                         | configuration file
 log_disconnections       | on                         | configuration file
 password_encryption      | scram-sha-256              | configuration file
 shared_preload_libraries | pg_stat_statements,pgaudit | configuration file
 ssl                      | on                         | configuration file
(5 rows)
```

---

## Checklist Prático de Segurança para o DBA Júnior

Imprima e confira este checklist antes de colocar qualquer banco em homologação ou produção:

| # | Item de Verificação | Estado Esperado | Como checar |
|---|---------------------|-----------------|-------------|
| 1 | **Criptografia de Senhas** | `scram-sha-256` | `SHOW password_encryption;` |
| 2 | **Criptografia em Trânsito** | `ssl = on` | `SHOW ssl;` |
| 3 | **Regras de Rede (`pg_hba`)** | Sem regras `trust`; IPs estritos | `SELECT * FROM pg_hba_file_rules;` |
| 4 | **Superusuários** | Somente `postgres` | `SELECT * FROM pg_roles WHERE rolsuper;` |
| 5 | **Acesso ao Schema Public** | `CREATE` revogado do `PUBLIC` | `\dn+ public` no psql |
| 6 | **Auditoria de Comandos** | `pgaudit` ativo e carregado | `SHOW pgaudit.log;` |
| 7 | **Log de Conexões** | `log_connections = on` | `SHOW log_connections;` |
| 8 | **Permissões no S.O.** | `0700` no data_dir / `0600` na key | `ls -ld $PGDATA` |
| 9 | **Portas Externas** | Firewall liberando apenas VIP/HAProxy | `ss -tulpn` |
| 10 | **Prevenção de Senha Vazia** | Nenhuma role sem senha | `SELECT rolname FROM pg_authid WHERE rolpassword IS NULL;` |

---

## Lições

- Hardening não é um evento único, mas um **processo contínuo de auditoria**.
- Mantenha sempre um `log_line_prefix` informativo: debugar incidentes sem saber qual usuário, banco e aplicação gerou a linha de log é um dos maiores desperdícios de tempo de um DBA.
- A segurança do banco depende diretamente da **segurança do sistema operacional subjacente**: um PostgreSQL perfeito em um servidor Linux com portas SSH abertas com senha fraca continua vulnerável.

---

## Pegadinhas

- **`log_connections = on` em ambientes de conexões efêmeras:** Se sua aplicação abre e fecha conexões a cada requisição HTTP sem usar um pooler (como PgBouncer), ativar `log_connections` e `log_disconnections` gerará gigabytes de log por dia. **Sempre use Connection Pooling**.
- **Backup esquecido com permissão aberta:** O DBA faz hardening primoroso no `$PGDATA`, mas roda um `pg_dump` para `/tmp/backup.sql` com permissão `0666` ou `0777`. Qualquer usuário local da máquina poderá ler o dump completo.
- **Logs legíveis por outros usuários:** Verifique a permissão do diretório de logs (`/var/lib/pgsql/18/data/log/`). O parâmetro `log_file_mode` deve estar configurado para `0600` (padrão) para impedir que outros usuários do SO leiam os logs do banco.
