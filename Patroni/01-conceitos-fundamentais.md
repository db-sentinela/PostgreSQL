# 01 — Conceitos fundamentais

Antes de mexer no cluster, entenda o vocabulário. Sem isso, os comandos viram
decoreba. Com isso, tudo faz sentido.

## Primary e réplica (standby)

- **Primary** (ou *leader*): o **único** servidor que aceita **escrita**
  (`INSERT`, `UPDATE`, `DELETE`). Só existe **um** por vez.
- **Réplica** (ou *standby*): uma cópia que recebe tudo o que o primary grava e
  fica em modo **somente-leitura**. Serve para consultas de leitura e, principalmente,
  para **assumir** se o primary cair.

> Analogia: o primary é o caixa que registra as vendas; as réplicas são cópias do
> livro-caixa atualizadas em tempo real, prontas para virar o caixa oficial.

## Replicação física (streaming)

O PostgreSQL grava tudo primeiro num diário chamado **WAL** (Write-Ahead Log).
Na replicação **física/streaming**, o primary envia esse WAL pela rede para as
réplicas, que **reproduzem** exatamente as mesmas mudanças. É cópia byte a byte,
não "exportar e importar". Detalhes no doc [03](03-replicacao-e-consistencia.md).

## Os 4 componentes e o papel de cada um

| Componente | Papel | Analogia |
|------------|-------|----------|
| **PostgreSQL** | Guarda os dados; faz a replicação | O banco em si |
| **Patroni** | O **cérebro**: decide quem é primary, monitora, promove réplica, coordena o failover | O maestro |
| **etcd** | A **fonte da verdade**: guarda, de forma distribuída, "quem é o líder"; dá o *lock* de liderança | A ata oficial da reunião |
| **HAProxy** | O **porteiro do tráfego**: manda a conexão para quem for o primary | O recepcionista que te leva à sala certa |
| **Keepalived** | Mantém um **IP virtual (VIP)** sempre vivo, para a aplicação ter um endereço fixo | O número de telefone que nunca muda |

Por que separar assim? Porque cada um faz **uma** coisa bem feita. O Patroni decide,
o etcd registra a decisão, o HAProxy executa o roteamento e o Keepalived garante o
endereço fixo. Juntos formam a HA.

## Failover x Switchover — não confunda

| | **Switchover** | **Failover** |
|---|---|---|
| Quando | **Você decide** (planejado) | **Sozinho** (o primary caiu) |
| Situação | Primary está **saudável** | Primary **falhou** |
| Perda de dados | **Nenhuma** (troca ordenada) | Possível, se replicação assíncrona |
| Exemplo | Atualizar o SO do primary | A VM do primary travou |

Os dois acabam no mesmo lugar (uma réplica vira primary), mas o switchover é uma
**decisão sua**; o failover é uma **reação automática**. Docs
[04](04-switchover-planejado.md) e [05](05-failover-automatico.md).

## Quorum — a regra que evita o caos

**Quorum** é a **maioria** dos membros do etcd que precisa estar de acordo para o
cluster tomar decisões. Com **3 nós**, o quorum é **2**.

Por que isso existe? Para evitar o **split-brain**: dois servidores achando que são
o primary ao mesmo tempo, cada um aceitando escritas diferentes — um desastre de
dados impossível de reconciliar. A regra da maioria garante que **só um lado**
possa agir. Doc [09](09-perda-de-quorum-split-brain.md).

> Por isso o número de nós é **ímpar** (3, 5, 7): com número par, um empate (ex.:
> 2 contra 2) não tem maioria.

## Replicação síncrona x assíncrona

- **Assíncrona** (nosso padrão): o primary confirma o `commit` **sem esperar** a
  réplica. É rápido, mas se o primary cair no exato instante seguinte, as últimas
  transações podem não ter chegado na réplica → **pequena perda possível**.
- **Síncrona**: o primary só confirma o `commit` **depois** que uma réplica
  confirmou ter recebido. **Zero perda**, mas cada escrita espera a rede → mais lento.

Escolha conforme o negócio: dados financeiros tendem a síncrono; logs toleram
assíncrono.

## RTO e RPO (dois siglões que vão te cobrar)

- **RTO** (Recovery Time Objective): **quanto tempo** o sistema fica fora até
  voltar. Num failover do Patroni, costuma ser **segundos**.
- **RPO** (Recovery Point Objective): **quanto de dado** você aceita perder.
  Assíncrono → RPO > 0 (segundos de dados); síncrono → RPO = 0.

## Próximo passo

Agora que o vocabulário está claro, aprenda a **ler o estado do cluster** no doc
[02 — Entendendo o `patronictl list`](02-entendendo-o-cluster.md).
