# 04 — Switchover planejado

## O que é

**Switchover** é você **trocar o primary de propósito**, com tudo funcionando. É uma
troca **ordenada e sem perda de dados**: o Patroni garante que a réplica escolhida
esteja 100% em dia antes de promovê-la.

## Por que importa

Você vai precisar disso o tempo todo em produção:

- Atualizar o SO ou reiniciar a VM do primary.
- Aplicar um patch no PostgreSQL do primary.
- Rebalancear qual nó é o primary depois de uma manutenção.

Fazer manualmente (parar o banco, promover outro, reconfigurar) é lento e arriscado.
O switchover faz tudo isso em **um comando**.

## Como reproduzir

```bash
# troca o primary de db-01 para db-02:
patronictl -c /etc/patroni/patroni.yml switchover pgha --leader db-01 --candidate db-02 --force
```

- `--leader db-01` → quem é o primary **agora** (confirmação de segurança).
- `--candidate db-02` → quem **você quer** que vire primary.
- `--force` → não perguntar interativamente (bom para scripts).

## O que você vê (saída real)

```
## SWITCHOVER planejado: db-01 -> db-02
2026-11-10 19:32:31  Successfully switched over to "db-02"
primary agora (VIP:5000) = 192.168.1.17   (antes era 192.168.1.15)

## voltando: db-02 -> db-01
2026-11-10 19:32:44  Successfully switched over to "db-01"
primary agora (VIP:5000) = 192.168.1.15
```

E o cluster logo depois, já estável de novo:

```
| Member | Host         | Role    | State     | TL |
| db-01  | 192.168.1.15 | Leader  | running   |  6 |
| db-02  | 192.168.1.17 | Replica | streaming |  6 |
| db-03  | 192.168.1.18 | Replica | streaming |  6 |
```

Repare em **dois detalhes**:

1. **A escrita pelo VIP seguiu o novo primary sozinha** (192.168.1.17 e depois
   .15). Você não mudou nada na aplicação — o HAProxy detectou a troca pela REST API
   e passou a rotear para o novo primary.
2. **A Timeline (TL) subiu** a cada promoção. Isso é normal e esperado: toda troca
   de líder avança a timeline. Todos os nós ficam na mesma TL.

## O que o Patroni faz por baixo dos panos

1. Verifica se o candidato (`db-02`) está **em dia** (sem lag). Se não estiver, recusa.
2. Faz o primary atual **parar de aceitar escrita** e garante que o candidato recebeu
   todo o WAL.
3. **Promove** o candidato a primary.
4. Reconfigura o antigo primary para virar **réplica** do novo.
5. Atualiza o etcd. O HAProxy, lendo a REST API, muda o roteamento.

Por isso é **sem perda**: nada é promovido antes de estar sincronizado.

## Lições

- Switchover = troca **planejada**, **zero perda**, **um comando**.
- A aplicação não precisa saber de nada: o VIP + HAProxy seguem o primary.
- Cada troca **incrementa a timeline** — é assim que o cluster registra a história.

## Pegadinhas

- Passar o `--leader` errado (quem não é o primary atual) faz o comando recusar — isso
  é proteção, não bug.
- Switchover exige o candidato **saudável e em dia**. Se a réplica estiver com lag
  alto, sincronize antes.
- Não confunda com **failover** (doc [05](05-failover-automatico.md)): switchover é
  quando está **tudo bem** e **você** decide.
