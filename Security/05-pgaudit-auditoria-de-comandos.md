# 05 — pgaudit (Auditoria de Comandos)

## O que é

O **pgAudit (PostgreSQL Audit Extension)** é a extensão padrão de mercado para auditoria
detalhada de ações no banco de dados.

Enquanto o log nativo do PostgreSQL com `log_statement = 'all'` gera um volume gigantesco
e desestruturado de texto misturando queries operacionais e comandos triviais, o **pgAudit**
produz registros estruturados em conformidade com normas regulatórias governamentais e financeiras
(como PCI-DSS, SOC 2, HIPAA e LGPD).

---

## Por que importa

- **Rastreabilidade Forense:** Se uma tabela for deletada ou um privilégio for concedido indevidamente às 3h da manhã, o pgAudit informa com precisão cirúrgica: quem executou, de qual IP, em qual transação e o comando exato.
- **Diferenciação por classe:** Permite auditar apenas comandos de alto risco (DDL, alterações de roles e escrita em dados críticos), ignorando leituras massivas triviais (`SELECT`) para economizar disco e I/O.

---

## Como reproduzir

### 1. Instalação do pacote no sistema operacional

No Fedora 42 (utilizando o repositório oficial da PGDG):

```bash
dnf install -y pgaudit_18
```

*(Deve ser instalado em todos os nós do cluster: `db-01`, `db-02`, `db-03`).*

### 2. Configurar o Patroni DCS e reiniciar o cluster

O pgAudit precisa ser carregado no início da inicialização do PostgreSQL via `shared_preload_libraries`.

No nó principal:
```bash
patronictl -c /etc/patroni/patroni.yml edit-config pgha \
  -s "postgresql.parameters.shared_preload_libraries=pg_stat_statements,pgaudit" \
  -s "postgresql.parameters.pgaudit.log=ddl, role, write" \
  --force

# Reiniciar o cluster de forma ordenada (rolling restart)
patronictl -c /etc/patroni/patroni.yml restart pgha --force
```

### 3. Habilitar a extensão no banco de dados

Conecte no banco desejado como superusuário e execute:

```sql
CREATE EXTENSION IF NOT EXISTS pgaudit;
```

### 4. Simular ações para gerar auditoria

```sql
CREATE TABLE seguranca.audit_teste (id int, nome text);
ALTER TABLE seguranca.audit_teste ADD COLUMN criado_em timestamp DEFAULT now();
INSERT INTO seguranca.audit_teste VALUES (1, 'Teste Auditoria');
DROP TABLE seguranca.audit_teste;
```

---

## O que você vê (saída real)

Inspecione o log atual do PostgreSQL (`/var/lib/pgsql/18/data/log/*.log`):

```
2026-11-13 17:53:18.138 -03 [3342] LOG:  AUDIT: SESSION,1,1,DDL,CREATE EXTENSION,,,CREATE EXTENSION IF NOT EXISTS pgaudit,<not logged>
2026-11-13 17:53:18.143 -03 [3342] LOG:  AUDIT: SESSION,2,1,DDL,CREATE TABLE,TABLE,seguranca.audit_teste,"CREATE TABLE seguranca.audit_teste (id int, nome text)",<not logged>
2026-11-13 17:53:18.144 -03 [3342] LOG:  AUDIT: SESSION,3,1,DDL,ALTER TABLE,TABLE,seguranca.audit_teste,ALTER TABLE seguranca.audit_teste ADD COLUMN criado_em timestamp DEFAULT now(),<not logged>
2026-11-13 17:53:18.144 -03 [3342] LOG:  AUDIT: SESSION,4,1,WRITE,INSERT,,,"INSERT INTO seguranca.audit_teste VALUES (1, 'Teste Auditoria')",<not logged>
2026-11-13 17:53:18.147 -03 [3342] LOG:  AUDIT: SESSION,5,1,DDL,DROP TABLE,TABLE,seguranca.audit_teste,DROP TABLE seguranca.audit_teste,<not logged>
2026-11-13 17:53:18.147 -03 [3342] LOG:  AUDIT: SESSION,5,1,DDL,DROP TABLE,DEFAULT VALUE,for seguranca.audit_teste.criado_em,DROP TABLE seguranca.audit_teste,<not logged>
```

### Anatomia da linha do pgAudit:
1. `AUDIT: SESSION`: Tipo de auditoria (por sessão).
2. `2,1`: ID sequencial da declaração dentro da sessão e sub-declaração.
3. `DDL`: Classe auditada (`DDL`, `ROLE`, `WRITE`, etc.).
4. `CREATE TABLE`: Tipo do comando executado.
5. `TABLE,seguranca.audit_teste`: Objeto alvo afetado.
6. `CREATE TABLE seguranca.audit_teste...`: Texto exato da instrução executada.

---

## Classes de Auditoria do pgAudit

Você pode configurar `pgaudit.log` combinando as seguintes classes:

| Classe | O que audita | Recomendação em Produção |
|--------|--------------|---------------------------|
| `ddl` | `CREATE`, `ALTER`, `DROP` | **Sempre ligada** |
| `role` | `GRANT`, `REVOKE`, `CREATE/ALTER ROLE` | **Sempre ligada** |
| `write` | `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE` | Usar em bancos ou schemas críticos |
| `read` | `SELECT`, `COPY` | Muito pesado! Usar apenas com auditoria por objeto |
| `function` | Chamadas de funções e blocos `DO` | Moderado |
| `misc` | `DISCARD`, `VACUUM`, `LOCK` | Opcional |

---

## Lições

- O pgAudit é modular: você audita **o que realmente importa** sem inundar o disco de logs desnecessários.
- Para auditar tabelas específicas de dados ultrassensíveis (ex: tabela de pagamentos) sem gerar log de escrita no banco todo, use **Auditoria por Objeto** via role de auditoria (`pgaudit.role`).
- `shared_preload_libraries` exige reinicialização do PostgreSQL; já os parâmetros como `pgaudit.log` podem ser alterados em tempo de execução via `reload`.

---

## Pegadinhas

- **Falta de restart:** Adicionar `pgaudit` ao `patronictl edit-config` e apenas dar `reload` **não carrega a biblioteca**. O log mostrará `pgaudit must be loaded via shared_preload_libraries`. É obrigatório fazer restart.
- **Log flooding com `pgaudit.log = 'read'`:** Nunca ligue `read` globalmente em instâncias com alto tráfego OLTP, a menos que você tenha um cluster de log dedicado (ex: Elasticsearch/Graylog/Loki) capaz de absorver centenas de milhares de linhas por segundo.
- **Parâmetros de queries preparados:** Se você usa queries com bind parameters (`$1`, `$2`), o pgAudit por padrão exibe `<not logged>` para os valores se `pgaudit.log_parameter` estiver desligado. Se precisar auditar os parâmetros das queries, ative `pgaudit.log_parameter = on` (atenção para não vazar dados de cartão de crédito no log!).
