# 09 — Perda de quorum e split-brain (o cenário mais importante)

Se você só for ler **um** documento sobre HA, leia este. Ele explica **por que
existem 3 nós** e por que, às vezes, o cluster **prefere ficar indisponível** a
arriscar os seus dados.

## O pesadelo: split-brain

**Split-brain** ("cérebro dividido") é quando **dois nós acham que são o primary ao
mesmo tempo** e os dois aceitam escrita. A aplicação grava o pedido X num; outra
conexão grava o pedido Y no outro. Agora existem **duas verdades diferentes** e
**nenhuma forma segura** de juntar. É o pior desastre de dados possível num cluster.

## A defesa: quorum (regra da maioria)

Para **nunca** haver dois primaries, o Patroni usa o etcd com a regra do **quorum**:
só pode existir um primary se a **maioria** dos membros do etcd concordar e conseguir
registrar isso.

Com **3 nós**, a conta é:

| Nós vivos | Tem maioria (≥2)? | Pode ter primary? |
|-----------|-------------------|-------------------|
| 3 de 3 | Sim | Sim ✅ |
| 2 de 3 | Sim | Sim ✅ (failover normal) |
| **1 de 3** | **Não** | **Não** ❌ (vira só-leitura) |

É por isso que se usa **número ímpar** de nós (3, 5, 7): sempre há um critério claro
de maioria, sem empate.

## Por que isso protege

Imagine que a rede se parta e um nó fique **isolado** dos outros dois. Esse nó
sozinho **não tem maioria**, então ele **não pode** se manter como primary — ele se
rebaixa a somente-leitura. Os outros dois, que **têm** maioria, elegem um primary
entre eles. Resultado: **nunca** dois primaries. Sem split-brain.

## Como reproduzir (⚠️ derruba o cluster de propósito)

```bash
# derruba etcd + patroni em DOIS nós, deixando só um vivo:
systemctl stop patroni etcd    # no db-01
systemctl stop patroni etcd    # no db-02

# espere o TTL do lock (~35s) e observe no nó que sobrou (db-03):
```

## O que você vê (saída real)

O etcd do nó solitário fica **sem quorum**:

```
192.168.1.18:2379 is unhealthy: failed to commit proposal: context deadline exceeded
```

O nó que era primary **se rebaixa a somente-leitura** — exatamente a proteção
esperada:

```
## db-03 ainda aceita ESCRITA?
ERROR:  cannot execute INSERT in a read-only transaction
## db-03 está em recovery (read-only)?  recovery = t
```

E a escrita pelo VIP **falha** (o HAProxy não tem mais um primary saudável para
onde mandar):

```
## escrita pelo VIP:5000
psql: error: connection to server at "192.168.1.10", port 5000 failed:
      server closed the connection unexpectedly
```

> **Leia com atenção:** o cluster ficou **indisponível para escrita**, mas **não
> corrompeu nada**. "Parar com segurança" é melhor que "continuar e duplicar dados".
> Essa é a filosofia da HA bem feita.

## A recuperação (volta o quorum, volta tudo)

Basta devolver a maioria: religar o etcd (e o Patroni) nos nós que caíram.

```bash
systemctl start etcd      # nos 2 nós que caíram
systemctl start patroni   # nos 2 nós que caíram
```

### Saída real da recuperação

```
## etcd health
192.168.1.15:2379 is healthy
192.168.1.17:2379 is healthy
192.168.1.18:2379 is healthy

## list recuperado
| db-01 | 192.168.1.15 | Replica | streaming | 8 |
| db-02 | 192.168.1.17 | Replica | streaming | 8 |
| db-03 | 192.168.1.18 | Leader  | running   | 8 |

## escrita pelo VIP voltou?
INSERT 0 1
ok primary = 192.168.1.18
```

Assim que a maioria voltou, o Patroni **reassumiu sozinho**: promoveu um primary,
as réplicas reconectaram e a escrita voltou. Sem intervenção manual nos dados.

## Lições

- **Split-brain** = dois primaries = desastre. O quorum existe para impedir isso.
- Com 3 nós, você tolera a perda de **1**. Perdendo **2**, o cluster vira
  **somente-leitura** de propósito (seguro), até a maioria voltar.
- Indisponível ≠ corrompido. O Patroni escolhe **parar** em vez de arriscar.
- Por isso o etcd roda em **3 nós** e o número é **ímpar**.

## Pegadinhas

- **Não coloque os 3 etcd na mesma máquina física / mesmo host de virtualização.**
  Se esse host cair, você perde o quorum inteiro de uma vez. Em produção, espalhe os
  nós (racks/zonas diferentes).
- Rodar com só **2 nós** não dá HA real: perder 1 já tira o quorum (maioria de 2 é 2).
- Em uma partição de rede, o lado **minoritário** sempre vira só-leitura — mesmo que
  seja o que tinha o primary. É esperado e correto.
