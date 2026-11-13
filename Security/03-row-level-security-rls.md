# 03 — Row-Level Security (RLS)

## O que é

O **Row-Level Security (RLS)** é um mecanismo nativo do PostgreSQL que filtra linhas de tabelas
automaticamente com base em políticas de segurança (`CREATE POLICY`), **antes** que os dados
cheguem à consulta da aplicação.

Enquanto permissões tradicionais (`GRANT/REVOKE`) controlam o acesso à tabela inteira ou a colunas,
o RLS controla **quais linhas individuais** um usuário tem o direito de enxergar ou modificar.

---

## Por que importa

- **Aplicações Multi-tenant (SaaS):** Em sistemas onde centenas de clientes compartilham o mesmo banco de dados, o maior pesadelo de um desenvolvedor é esquecer um `WHERE tenant_id = 123` numa query e expor faturas ou dados sensíveis de um cliente para outro.
- **Segurança no nível do banco (Defesa em Profundidade):** Mesmo se a aplicação tiver um bug crítico de lógica ou uma injeção de SQL for bem-sucedida, o PostgreSQL garante que o usuário nunca conseguirá ler linhas de outro cliente.

---

## Como reproduzir

### 1. Criar tabela multi-tenant e dados de exemplo

```sql
CREATE TABLE seguranca.faturas (
    id serial PRIMARY KEY,
    tenant_id int NOT NULL,
    descricao text NOT NULL,
    valor numeric(10,2) NOT NULL
);

INSERT INTO seguranca.faturas (tenant_id, descricao, valor) VALUES
(101, 'Servidor Cloud - Alpha', 450.00),
(101, 'Suporte Mensal - Alpha', 200.00),
(202, 'Licenciamento Beta', 1200.00);

GRANT SELECT, INSERT, UPDATE, DELETE ON seguranca.faturas TO rol_readwrite;
```

### 2. Ativar o RLS na tabela

```sql
ALTER TABLE seguranca.faturas ENABLE ROW LEVEL SECURITY;
```

> **Atenção:** Apenas ativar o RLS sem criar políticas bloqueia a leitura/escrita de todos os usuários comuns (política restritiva por padrão).

### 3. Criar a política de isolamento por Tenant

Usamos a função `current_setting('app.current_tenant', true)` para ler uma variável de contexto de sessão que o backend da sua aplicação define ao conectar:

```sql
CREATE POLICY tenant_isolation_policy ON seguranca.faturas
    FOR ALL
    TO PUBLIC
    USING (tenant_id = nullif(current_setting('app.current_tenant', true), '')::int)
    WITH CHECK (tenant_id = nullif(current_setting('app.current_tenant', true), '')::int);
```

- **`USING`**: Determina quais linhas existentes o usuário pode ler (`SELECT`), atualizar (`UPDATE`) ou deletar (`DELETE`).
- **`WITH CHECK`**: Determina se novas linhas inseridas (`INSERT`) ou alteradas (`UPDATE`) atendem à regra de segurança.

---

## O que você vê (saída real)

### 1. Superusuário (`postgres`) consultando a tabela:

O superusuário **ignora o RLS por padrão**:

```sql
SELECT * FROM seguranca.faturas;
```

```
 id | tenant_id |       descricao        |  valor  
----+-----------+------------------------+---------
  1 |       101 | Servidor Cloud - Alpha |  450.00
  2 |       101 | Suporte Mensal - Alpha |  200.00
  3 |       202 | Licenciamento Beta     | 1200.00
(3 rows)
```

### 2. Usuário comum (`user_app_rw`) sem definir o tenant:

Se o desenvolvedor esqueceu de setar a variável de sessão, o banco retorna **zero linhas** de forma segura:

```bash
PGPASSWORD='AppRw#2026' psql -h 127.0.0.1 -U user_app_rw -d postgres -c "SELECT * FROM seguranca.faturas;"
```

```
 id | tenant_id | descricao | valor 
----+-----------+-----------+-------
(0 rows)
```

### 3. Usuário comum setando o Tenant 101 (Empresa Alpha):

```bash
PGPASSWORD='AppRw#2026' psql -h 127.0.0.1 -U user_app_rw -d postgres -c "SET app.current_tenant = '101'; SELECT * FROM seguranca.faturas;"
```

```
SET
 id | tenant_id |       descricao        | valor  
----+-----------+------------------------+--------
  1 |       101 | Servidor Cloud - Alpha | 450.00
  2 |       101 | Suporte Mensal - Alpha | 200.00
(2 rows)
```

### 4. Usuário comum setando o Tenant 202 (Empresa Beta):

```bash
PGPASSWORD='AppRw#2026' psql -h 127.0.0.1 -U user_app_rw -d postgres -c "SET app.current_tenant = '202'; SELECT * FROM seguranca.faturas;"
```

```
SET
 id | tenant_id |     descricao      |  valor  
----+-----------+--------------------+---------
  3 |       202 | Licenciamento Beta | 1200.00
(1 row)
```

### 5. Tentativa de invasão / injeção entre tenants:

O usuário autenticado como tenant 101 tenta forjar um insert no tenant 202:

```bash
PGPASSWORD='AppRw#2026' psql -h 127.0.0.1 -U user_app_rw -d postgres -c "SET app.current_tenant = '101'; INSERT INTO seguranca.faturas (tenant_id, descricao, valor) VALUES (202, 'Fraude', 5000.00);"
```

Saída de erro real:
```
SET
ERROR:  new row violates row-level security policy for table "faturas"
```

A cláusula `WITH CHECK` rejeitou imediatamente a gravação.

---

## Lições

- O RLS transfere a responsabilidade de isolamento de dados do código da aplicação (que pode falhar) para o **motor de banco de dados** (que nunca falha).
- `USING` filtra leituras e exclusões; `WITH CHECK` valida inserções e alterações.
- O segundo parâmetro de `current_setting('param', true)` é o `missing_ok`: se for `true`, retorna `NULL` se a variável não estiver definida em vez de estourar erro fatal na sessão.

---

## Pegadinhas

- **Superusuários e donos da tabela:** Por padrão, o `postgres` e o dono da tabela (owner) **bypassam o RLS**. Se a sua aplicação conecta como dona da tabela, o RLS não terá efeito algum!
- **Como forçar o dono a respeitar o RLS:** Use `ALTER TABLE <tabela> FORCE ROW LEVEL SECURITY;`. Mesmo assim, o superusuário (`postgres`) continuará ignorando o RLS.
- **Impacto em Performance:** Políticas de RLS complexas (como subqueries pesadas no `USING`) são injetadas em toda consulta na tabela. Para consultas de alta volumetria, certifique-se de que a coluna filtrada (ex: `tenant_id`) **possui índice B-Tree**.
