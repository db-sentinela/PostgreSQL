# 06 — pgcrypto e Proteção de Dados Sensíveis

## O que é

O **`pgcrypto`** é uma extensão oficial que adiciona funções criptográficas diretamente no PostgreSQL.
Ele resolve dois problemas fundamentais de proteção de dados no banco:

1. **Hashing Unidirecional (Senhas):** Transformar senhas em hashes matemáticos irreversíveis utilizando algoritmos com resistência comprovada a força bruta (como **Bcrypt**).
2. **Criptografia Bidirecional (Dados Sensíveis):** Encriptar campos de alta criticidade (números de cartão de crédito, documentos pessoais, tokens bancários) utilizando cifras simétricas robustas (**AES-256**), permitindo decifrá-los apenas quando a chave correta for fornecida.

---

## Por que importa

- **Vazamento de banco / Dump roubado:** Se um arquivo `.sql` ou backup de banco for vazado ou exposto acidentalmente, qualquer invasor terá acesso imediato a dados de clientes. Com criptografia por coluna, o dado vazado é lixo ilegível sem a chave.
- **Armazenamento de senhas de aplicação:** Nunca armazene senhas de usuários da sua aplicação em texto puro ou com `md5`/`sha1`. O `pgcrypto` fornece a implementação de referência do `bcrypt` (`gen_salt('bf')`), que inclui salt automático e fator de trabalho adaptável.

---

## Como reproduzir

### 1. Habilitar a extensão

O módulo `pgcrypto` faz parte do pacote `postgresql-contrib` e já está disponível nativamente:

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

### 2. Criar tabela com campos protegidos

```sql
CREATE TABLE seguranca.credenciais (
    id serial PRIMARY KEY,
    usuario text UNIQUE NOT NULL,
    senha_hash text NOT NULL,
    cartao_encriptado bytea NOT NULL
);
```

> **Atenção:** Colunas criptografadas com `pgp_sym_encrypt` produzem sequências de bytes binários, portanto o tipo da coluna deve ser **`bytea`** (não `text` ou `varchar`).

### 3. Inserir dados com Bcrypt (hash) e AES-256 (criptografia)

```sql
INSERT INTO seguranca.credenciais (usuario, senha_hash, cartao_encriptado)
VALUES (
    'roberto_financeiro',
    -- Hashing com bcrypt e salt de 8 rounds:
    crypt('SegredoMestre#2026', gen_salt('bf', 8)),
    -- Criptografia simétrica com chave mestra e cifra AES-256:
    pgp_sym_encrypt('4532-8888-9999-1234', 'ChaveMestraAES2026', 'compress-algo=1, cipher-algo=aes256')
);
```

### 4. Consultar validando a senha e decifrando o cartão

```sql
SELECT 
    usuario,
    -- Validação: compara o hash armazenado com a senha fornecida:
    (senha_hash = crypt('SegredoMestre#2026', senha_hash)) AS senha_valida,
    (senha_hash = crypt('SenhaErrada', senha_hash)) AS senha_invalida,
    -- Decriptografia com a chave correta:
    pgp_sym_decrypt(cartao_encriptado, 'ChaveMestraAES2026') AS cartao_decriptado
FROM seguranca.credenciais;
```

---

## O que você vê (saída real)

```
      usuario       | senha_valida | senha_invalida |  cartao_decriptado  
--------------------+--------------+----------------+---------------------
 roberto_financeiro | t            | f              | 4532-8888-9999-1234
(1 row)
```

Se inspecionar a coluna no banco sem decifrar:

```sql
SELECT cartao_encriptado FROM seguranca.credenciais;
```

```
                               cartao_encriptado                               
-------------------------------------------------------------------------------
 \xc30d04070302fa9bc4b36070085bd2393a01533ad3b664d50c76ae5f560e9803b9b9409...
(1 row)
```

Mesmo tendo acesso físico ou superusuário ao banco, o valor real do cartão permanece inviolável sem a chave simétrica.

---

## Hashing x Criptografia: Qual usar?

| Cenário | Técnica | Função `pgcrypto` | Reversível? |
|---------|---------|-------------------|-------------|
| Senhas de usuários do sistema | Hash com Salt (Bcrypt) | `crypt(senha, gen_salt('bf'))` | **Não** (irreversível por design) |
| Número de cartão de crédito | Cifra Simétrica (AES-256) | `pgp_sym_encrypt(dado, chave)` | **Sim** (com a chave correta) |
| Validação de integridade de arquivo | Digest / Checksum | `digest(dado, 'sha256')` | **Não** |
| Troca segura entre partes | Cifra Assimétrica (PGP/GPG) | `pgp_pub_encrypt(...)` | **Sim** (com a chave privada) |

---

## Lições

- **Bcrypt desacelera ataques:** O segundo argumento de `gen_salt('bf', <custo>)` controla o custo computacional. No valor padrão (8 ou 10), calcular cada hash leva alguns milissegundos — imperceptível para um login humano, mas devastador para um invasor tentando bilhões de senhas por segundo.
- **O banco pode validar senhas sem jamais conhecê-las:** A função `crypt('tentativa', hash_do_banco)` extrai o salt do próprio hash e gera o mesmo resultado se a senha for idêntica.
- **Criptografia a nível de aplicação vs a nível de banco:** Criptografar colunas no banco protege contra roubo de disco/backup. Se a chave estiver no backend da aplicação, protege até contra invasão do servidor de banco de dados.

---

## Pegadinhas

- **Nunca passe chaves de criptografia em claro no log:** Se o parâmetro `log_statement = 'all'` estiver ativo ou você rodar a query sem bind parameters, sua chave mestra e o dado em claro aparecerão no log do PostgreSQL! Use queries preparadas com parâmetros (`$1`) ou funções de banco protegidas.
- **Custo de CPU com `gen_salt('bf', N)` muito alto:** Se colocar um custo como `14` ou `16`, cada inserção de senha pode levar 2 a 5 segundos de CPU 100%! Para servidores de produção, um custo entre `8` e `10` é o ponto de equilíbrio perfeito entre segurança e performance.
- **Índices em colunas encriptadas:** Você **não pode** fazer busca por igualdade (`WHERE cartao_encriptado = ...`) ou criar índices normais em dados criptografados com PGP, pois o algoritmo adiciona um vetor de inicialização (IV) aleatório a cada encriptação (a mesma string criptografada duas vezes gera bytes completamente diferentes). Se precisar buscar pelo dado, gere uma coluna auxiliar com hash determinístico (`digest(cartao, 'sha256')`) e crie o índice nela.
