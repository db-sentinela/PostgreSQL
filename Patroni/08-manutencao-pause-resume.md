# 08 — Manutenção sem failover acidental (pause / resume)

## O que é

O **modo de manutenção** (`pause`) diz ao Patroni: "fique quieto, **não** promova
nem reinicie nada automaticamente, eu vou mexer manualmente". Ao terminar, você dá
`resume` e o Patroni volta a gerenciar.

## Por que importa

Sem o pause, se você **parar o PostgreSQL** de um nó para manutenção, o Patroni
pensa que ele caiu e **dispara um failover** — exatamente o que você **não** quer no
meio de uma manutenção planejada. O pause evita esse "failover acidental".

Casos típicos:

- Reiniciar o PostgreSQL para aplicar um parâmetro que exige restart.
- Fazer upgrade de pacote no nó.
- Mexer no SO / reiniciar a VM com controle.

## Como reproduzir

```bash
# entra em manutenção (espera todos os nós reconhecerem):
patronictl -c /etc/patroni/patroni.yml pause pgha --wait

# ... faça sua manutenção manual aqui ...

# sai da manutenção:
patronictl -c /etc/patroni/patroni.yml resume pgha --wait
```

## O que você vê (saída real)

```
## entra em manutenção
'pause' request sent, waiting until it is recognized by all nodes
Success: cluster management is paused

## list  (repare na última linha)
| db-01 | 192.168.1.15 | Replica | streaming | 7 |
| db-02 | 192.168.1.17 | Replica | streaming | 7 |
| db-03 | 192.168.1.18 | Leader  | running   | 7 |
 Maintenance mode: on

## sai da manutenção
'resume' request sent, waiting until it is recognized by all nodes
Success: cluster management is resumed
```

A linha **`Maintenance mode: on`** no `patronictl list` é o seu lembrete visual de
que a automação está **congelada**. Enquanto estava pausado, mexer num nó **não**
provocou promoção de ninguém.

## O que muda com o pause ligado

| Ação | Patroni normal | Em `pause` |
|------|----------------|------------|
| PostgreSQL de um nó para | Dispara failover | **Não** faz nada |
| Nó fica para trás | Pode reiniciar/reagir | **Não** interfere |
| Você promove manualmente | — | Permitido e sob seu controle |

## Lições

- `pause` **antes** de qualquer manutenção que pare o PostgreSQL. `resume` ao fim.
- Confirme no `patronictl list` que apareceu `Maintenance mode: on`.
- Pause é **cluster-wide**: vale para todos os nós ao mesmo tempo.

## Pegadinhas

- **Esquecer o `resume`.** Com o cluster pausado, um failover **real** (queda de
  verdade do primary) **não** acontece — você fica sem HA sem perceber. Sempre
  confirme que voltou (`Maintenance mode` some do list).
- Usar `--wait` garante que todos os nós reconheceram o pause antes de você começar a
  mexer. Sem ele, você pode agir antes de o pause valer em todo mundo.
