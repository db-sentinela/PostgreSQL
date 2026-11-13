# 02 — Roles, Privilégios e Menor Privilégio (Least Privilege)

## O que é

No PostgreSQL, usuários e grupos são a mesma entidade: uma **Role**.
- Uma role com o atributo `LOGIN` funciona como um usuário de conexão.
- Uma role com `NOLOGIN` funciona como um **grupo de permissões** (RBAC - Role-Based Access Control).

O **Princípio do Menor Privilégio (Least Privilege)** determina que cada aplicação ou usuário deve ter estritamente as permissões necessárias para cumprir seu papel — e absolutamente nada além disso.

---

## Por que importa

- **Risco de Injeção de SQL (SQLi):** Se sua aplicação conecta como `postgres` (superusuário) e sofre um ataque de SQL Injection, o invasor pode executar `COPY ... PROGRAM` para rodar comandos como root/postgres no sistema operacional, apagar bancos inteiros (`DROP DATABASE`) ou ler arquivos confidenciais do servidor.
- **Acidentes de desenvolvimento:** Um script de migração mal executado por um estagiário ou aplicação não deve ter permissão para dar `DROP TABLE` ou `TRUNCATE` em tabelas de produção.

---

## Como reproduzir

### 1. Criar o schema e a tabela de negócios

```sql
CREATE SCHEMA seguranca;

CREATE TABLE seguranca.clientes (
    id serial PRIMARY KEY,
    nome text NOT NULL,
    email text UNIQUE,
    saldo numeric(10,2) DEFAULT 0.00
);

INSERT INTO seguranca.clientes (nome, email, saldo) VALUES 
('Mariana Silva', 'mariana@empresa.com', 1500.50),
('Carlos Lima', 'carlos@empresa.com', 3200.00);
```

### 2. Criar roles de grupo (NOLOGIN) para Leitura e Escrita

```sql
-- Role de Somente Leitura
CREATE ROLE rol_readonly NOLOGIN;
GRANT USAGE ON SCHEMA seguranca TO rol_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA seguranca TO rol_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA seguranca GRANT SELECT ON TABLES TO rol_readonly;

-- Role de Leitura e Escrita (DML)
CREATE ROLE rol_readwrite NOLOGIN;
GRANT USAGE ON SCHEMA seguranca TO rol_readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA seguranca TO rol_readwrite;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA seguranca TO rol_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA seguranca GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO rol_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA seguranca GRANT USAGE, SELECT ON SEQUENCES TO rol_readwrite;
```

### 3. Criar os usuários da aplicação atribuindo os grupos

```sql
CREATE ROLE user_app_ro WITH LOGIN PASSWORD 'AppRo#2026' IN ROLE rol_readonly;
CREATE ROLE user_app_rw WITH LOGIN PASSWORD 'AppRw#2026' IN ROLE rol_readwrite;
```

---

## O que você vê (saída real)

### 1. Usuário Read-Only consultando dados (Sucesso):

```bash
PGPASSWORD='AppRo#2026' psql -h 127.0.0.1 -U user_app_ro -d postgres -c "SELECT id, nome, saldo FROM seguranca.clientes;"
```

Saída:
```
 id |     nome      |  saldo  
----+---------------+---------
  1 | Mariana Silva | 1500.50
  2 | Carlos Lima   | 3200.00
(2 rows)
```

### 2. Usuário Read-Only tentando inserir registro (Bloqueio):

```bash
PGPASSWORD='AppRo#2026' psql -h 127.0.0.1 -U user_app_ro -d postgres -c "INSERT INTO seguranca.clientes (nome, email) VALUES ('Hacker', 'hack@test.com');"
```

Saída de erro real:
```
ERROR:  permission denied for table clientes
```

### 3. Usuário Read-Write tentando derrubar a tabela (Bloqueio DDL):

```bash
PGPASSWORD='AppRw#2026' psql -h 127.0.0.1 -U user_app_rw -d postgres -c "DROP TABLE seguranca.clientes;"
```

Saída de erro real:
```
ERROR:  must be owner of table clientes
```

O usuário tem permissão para `SELECT`, `INSERT`, `UPDATE` e `DELETE`, mas **não é dono da tabela**, portanto não pode destruí-la.

---

## Lições

- **Nunca use superusuário na aplicação:** A role `postgres` deve ser reservada para manutenções administrativas e automações do Patroni.
- **Use o padrão Grupo + Membro:** Crie roles abstratas de permissão (`rol_readonly`, `rol_readwrite`, `rol_analytics`) e conceda essas roles aos usuários reais (`user_app_rw IN ROLE rol_readwrite`).
- **`ALTER DEFAULT PRIVILEGES` é vital:** Sem ele, toda vez que uma nova tabela for criada no schema, ninguém terá permissão sobre ela até que alguém rode um novo `GRANT`.
- **Sequences importam:** Se uma tabela usa `serial` ou `GENERATED AS IDENTITY`, o usuário precisa de `USAGE, SELECT` na sequence correspondente, caso contrário o `INSERT` falhará.

---

## Pegadinhas

- **O perigo do schema `public`:** Historicamente no PostgreSQL, todos os usuários herdam permissão para criar tabelas no schema `public`. Em ambientes seguros, revogue isso explicitamente:
  ```sql
  REVOKE CREATE ON SCHEMA public FROM PUBLIC;
  ```
- **DEFAULT PRIVILEGES pertence ao criador:** Se o usuário `postgres` roda `ALTER DEFAULT PRIVILEGES`, isso só valerá para objetos criados pelo próprio `postgres`. Se outro usuário criar objetos no schema, as permissões padrão não serão herdadas a menos que `FOR ROLE <criador>` seja especificado.
- **Herança de Roles (`INHERIT`):** Por padrão, roles criadas no PostgreSQL possuem o atributo `INHERIT`, o que significa que herdam automaticamente os privilégios dos grupos a que pertencem. Se criar uma role com `NOINHERIT`, o usuário precisará rodar `SET ROLE rol_readwrite;` explicitamente para assumir os privilégios.
