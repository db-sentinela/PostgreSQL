# 01 — Autenticação SCRAM e pg_hba

## O que é

A **autenticação** é o porteiro do PostgreSQL: ela decide se quem está tentando entrar
é realmente quem diz ser. O controle é feito em duas etapas:
1. **`pg_hba.conf` (Host-Based Authentication):** Uma lista de regras que diz: *"De qual IP/rede, para qual banco, qual usuário pode tentar autenticar e usando qual método"*.
2. **Método de Autenticação:**
   - **`trust`**: Deixa entrar sem pedir senha (perigoso para redes).
   - **`md5`**: Legado e vulnerável a ataques de colisão e vazamentos de hash.
   - **`scram-sha-256`**: O padrão moderno (RFC 7677 / 5802). Utiliza salt criptográfico e derivações de chave (PBKDF2) para garantir que a senha nunca trafegue em claro e o hash do banco não permita ataques de força bruta rápidos com rainbow tables.

---

## Por que importa

- **Vazamento de backup:** Se um invasor roubar um dump do banco que contenha senhas em `md5`, ele consegue quebrar a maioria das senhas fracas em minutos com rainbow tables. No `scram-sha-256`, o custo computacional de quebra é ordens de grandeza maior.
- **Segurança de rede:** Uma única regra `trust` mal posicionada permite que qualquer pessoa na sua rede interna assuma a identidade de `postgres` (superusuário) sem credencial alguma.

---

## Como reproduzir

### 1. Verificar se o cluster está configurado para SCRAM

No PostgreSQL, a variável `password_encryption` define o algoritmo usado ao criar novos usuários ou redefinir senhas:

```sql
SHOW password_encryption;
```

### 2. Criar um usuário com senha forte

```sql
CREATE ROLE sec_analista WITH LOGIN PASSWORD 'SenhaForte#2026';
```

Inspecione o formato do hash gerado no catálogo interno:

```sql
SELECT rolname, substring(rolpassword from 1 for 30) || '...' as hash_scram 
FROM pg_authid 
WHERE rolname = 'sec_analista';
```

### 3. Entender a ordem das regras do `pg_hba.conf`

Consulte a view do sistema que expõe o arquivo `pg_hba.conf` carregado:

```sql
SELECT line_number, type, database, user_name, address, auth_method 
FROM pg_hba_file_rules 
ORDER BY line_number;
```

### 4. Testar a autenticação via IP da rede

**Conexão com senha correta:**
```bash
PGPASSWORD='SenhaForte#2026' psql -h 192.168.1.15 -U sec_analista -d postgres -c "SELECT current_user;"
```

**Conexão com senha errada:**
```bash
PGPASSWORD='SenhaIncorreta' psql -h 192.168.1.15 -U sec_analista -d postgres -c "SELECT current_user;"
```

---

## O que você vê (saída real)

### Hash armazenado com SCRAM-SHA-256:

```
   rolname    |           hash_scram           
--------------+---------------------------------
 sec_analista | SCRAM-SHA-256$4096:Rby1jb8xS...
(1 row)
```

Repare no prefixo `SCRAM-SHA-256$4096:`. O número `4096` representa as iterações do PBKDF2 com salt aleatório por usuário.

### Regras do `pg_hba` carregadas no cluster:

```
 line_number | type  |   database    |  user_name   |   address   |  auth_method  
-------------+-------+---------------+--------------+-------------+---------------
         117 | local | {all}         | {all}        |             | trust
         119 | host  | {all}         | {all}        | 127.0.0.1   | trust
         121 | host  | {all}         | {all}        | ::1         | trust
         129 | host  | {all}         | {all}        | 127.0.0.1   | scram-sha-256
         130 | host  | {all}         | {all}        | 192.168.1.0 | scram-sha-256
         131 | host  | {replication} | {replicator} | 192.168.1.0 | scram-sha-256
```

### Tentativa de login bem-sucedida:

```
 current_user 
--------------
 sec_analista
(1 row)
```

### Tentativa de login com senha incorreta (rejeição imediata):

```
psql: error: connection to server at "192.168.1.15", port 5432 failed: FATAL:  password authentication failed for user "sec_analista"
```

---

## Lições

- **`password_encryption = scram-sha-256`** deve ser o padrão absoluto em qualquer ambiente.
- Mudar o parâmetro `password_encryption` **não converte senhas antigas** automaticamente. É preciso rodar `ALTER ROLE <usuario> WITH PASSWORD '<senha>'` para cada usuário migrar seu hash de `md5` para `scram`.
- **A primeira regra correspondente no `pg_hba.conf` vence.** Se houver uma regra genérica acima de uma regra restritiva, a restritiva nunca será avaliada.

---

## Pegadinhas

- **A armadilha do `127.0.0.1 trust`**: Se a linha 119 tiver `host all all 127.0.0.1 trust`, qualquer comando disparado na própria máquina local conseguirá conectar com **qualquer usuário** sem digitar senha (ou mesmo passando uma senha errada!). Sempre restrinja o loopback para `scram-sha-256` ou `peer` (sockets locais unix).
- **Sobrescrita do `pg_hba` no Patroni**: Nunca edite o arquivo `/var/lib/pgsql/18/data/pg_hba.conf` diretamente no disco. O Patroni gerencia o `pg_hba` através da seção `postgresql.pg_hba` no DCS (`patronictl edit-config`). Se editar na mão, o Patroni sobrescreverá suas alterações no próximo reload ou restart!
- **Clientes legados:** Drivers JDBC muito antigos (anteriores a 2018) ou versões arcaicas de bibliotecas de conexão podem não suportar SCRAM-SHA-256. Atualize o driver da aplicação antes de forçar SCRAM.
