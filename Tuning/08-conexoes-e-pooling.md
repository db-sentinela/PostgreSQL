# 08 — Conexões e connection pooling

Um tema que derruba bancos em produção: **conexões demais**. Entender isso evita
uma das piores dores de cabeça.

## Cada conexão custa caro

No PostgreSQL, **cada conexão é um processo** do sistema operacional (não uma
thread). Cada processo consome memória (alguns MB de início) e disputa CPU. Então:

- **Centenas** de conexões ociosas = memória e CPU desperdiçados.
- Conexões **ativas** demais (mais que ~2–4× o número de vCPUs) = disputa; o banco
  fica **mais lento** fazendo tudo ao mesmo tempo do que faria em fila.

> Regra de ouro: o número de conexões **ativas** ideal costuma ser próximo de
> `vCPUs × 2` a `vCPUs × 4`. No nosso lab (4 vCPU), algo como **8–16 ativas**, não
> 200. (`max_connections = 200` é o **teto**, não uma meta.)

## O problema do "cada app abre 100 conexões"

Aplicações web/microsserviços costumam abrir pools grandes. Dez instâncias com pool
de 50 = 500 conexões no banco, a maioria **ociosa**, mas cada uma segurando recursos.
Quando sobra pouca, chega o temido:

```
FATAL: sorry, too many clients already
```

## A solução: connection pooler (PgBouncer)

Um **pooler** fica **entre** a aplicação e o PostgreSQL. As apps abrem **muitas**
conexões **leves** no pooler; o pooler mantém **poucas** conexões reais ao banco e as
**reaproveita**.

```
[ 500 apps ] ──► [ PgBouncer: 500 conexões leves ] ──► [ 20 conexões reais ao Postgres ]
```

### Modos do PgBouncer

| Modo | Reaproveita a conexão real... | Uso |
|------|-------------------------------|-----|
| **session** | ...no fim da **sessão** do cliente | Compatível com tudo, menos economia |
| **transaction** | ...no fim de cada **transação** | **O mais usado** — ótima economia |
| **statement** | ...a cada **comando** | Agressivo; restrições de uso |

O modo **transaction** é o campeão: entre uma transação e outra, a conexão real
volta ao pool e serve outro cliente. Dá para servir **centenas** de apps com
**dezenas** de conexões reais.

> Cuidado no modo transaction: recursos "de sessão" (prepared statements no servidor,
> `SET` de sessão, advisory locks de sessão, `LISTEN/NOTIFY`) podem não funcionar como
> esperado, porque a conexão real muda de dono a cada transação. Configure a app de
> acordo.

## Exemplo mínimo de PgBouncer (`pgbouncer.ini`)

```ini
[databases]
lojadb = host=192.168.1.10 port=5432 dbname=lojadb   ; via VIP do cluster

[pgbouncer]
listen_port = 6432
auth_type = scram-sha-256
pool_mode = transaction
max_client_conn = 1000      ; conexões leves que a app abre
default_pool_size = 20      ; conexões reais por banco/usuário
```

A aplicação então conecta na **porta 6432 do PgBouncer** em vez de direto no 5432.

## Dimensionando o pool

- `default_pool_size` ≈ `vCPUs × 2` a `× 4` (as conexões **reais** de trabalho).
- `max_client_conn` = quantas conexões leves você aceita das apps (pode ser alto).
- Some os pools de todos os bancos/usuários e confira que o total de conexões reais
  cabe no `max_connections` do PostgreSQL com folga.

## Em um cluster Patroni

- O PgBouncer aponta para o **VIP** (doc `../Patroni/`), então segue o primary no
  failover.
- Para leituras, você pode ter um PgBouncer apontando para a porta de réplicas do
  HAProxy (5001).

## Lições

- Cada conexão é um **processo**; conexões ociosas demais desperdiçam recursos.
- Conexões **ativas** ideais ≈ `vCPUs × 2–4`, não o `max_connections`.
- Use um **pooler** (PgBouncer, modo **transaction**) para servir muitas apps com
  poucas conexões reais.

## Pegadinhas

- Subir `max_connections` para "resolver" o too many clients só **empurra** o
  problema e gasta mais memória. A solução é **pooling**, não mais conexões.
- Modo transaction quebra prepared statements de sessão e `SET` persistente — ajuste
  a aplicação/driver.
- Pool_size grande demais no PgBouncer anula o benefício (volta a sobrecarregar o
  banco).
