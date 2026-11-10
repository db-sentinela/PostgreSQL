# 05 — Failover automático

## O que é

**Failover** é a promoção **automática** de uma réplica quando o primary **cai
sozinho**. Ninguém aperta botão: o Patroni detecta a falha e age em segundos. É o
coração da alta disponibilidade.

## Por que importa

É o cenário real de desastre: a VM do primary travou, o disco morreu, a rede caiu.
Sem failover, a aplicação fica fora até alguém acordar e consertar. Com failover, a
parada é de **segundos**.

## Como o Patroni detecta e decide

1. O primary renova, de tempos em tempos, um **lock de liderança** no etcd (TTL de
   30s no nosso lab).
2. Se o primary para de renovar (porque caiu), o lock **expira**.
3. As réplicas percebem que não há líder e **disputam** a liderança via etcd.
4. A réplica **mais em dia** (menor lag) ganha, vira primary e avança a timeline.
5. O HAProxy, pela REST API, passa a rotear a escrita para o novo primary.

> Esse "tempo até promover" depende do TTL. TTL menor = failover mais rápido, porém
> mais sensível a soluços de rede. 30s é um equilíbrio comum.

## Como reproduzir (simulando a queda do primary)

```bash
# no primary atual (ex.: db-01), derruba o Patroni (simula a queda):
systemctl stop patroni

# de OUTRO nó, observe a eleição:
patronictl -c /etc/patroni/patroni.yml list
```

## O que você vê (saída real)

Primary db-01 derrubado → **db-03 foi promovido sozinho**:

```
| Member | Host         | Role    | State     | TL |
| db-01  | 192.168.1.15 | Replica | stopped   |    |
| db-02  | 192.168.1.17 | Replica | streaming |  7 |
| db-03  | 192.168.1.18 | Leader  | running   |  7 |
```

E a **escrita continuou** pelo VIP, agora apontando para o novo primary:

```
## escrita pelo VIP:5000 durante o failover
INSERT 0 1
novo primary = 192.168.1.18
```

A aplicação só precisou **reconectar**: o VIP é o mesmo (192.168.1.10), e o HAProxy
já estava mandando para o db-03.

## O que a aplicação sente

- As conexões que estavam **abertas** no primary antigo **caem** (ele sumiu).
- Ao **reconectar** no VIP, a aplicação já chega no novo primary.
- Por isso, apps resilientes usam **retry com reconexão**. O failover reduz o RTO a
  segundos, mas não torna as conexões em andamento imortais.

## Failover x Switchover (lembrete)

| | Switchover | Failover |
|---|---|---|
| Gatilho | Você | Queda do primary |
| Primary | Saudável | Falhou |
| Perda | Zero | Possível (assíncrono) |

## Lições

- Failover = promoção **automática** quando o primary cai. RTO de segundos.
- Quem decide é o **quorum** do etcd (doc [09](09-perda-de-quorum-split-brain.md)):
  sem maioria, não há promoção — é proteção contra split-brain.
- A aplicação precisa **reconectar**; o VIP continua o mesmo.

## Pegadinhas

- Em replicação **assíncrona**, as últimas transações do primary morto podem não ter
  chegado na réplica promovida → RPO > 0. Quer RPO 0? Replicação síncrona.
- O nó que caiu, ao voltar, **não** vira primary de novo sozinho — ele volta como
  **réplica** (doc [06](06-reintegracao-de-nos.md)).
