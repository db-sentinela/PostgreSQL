# Alta Disponibilidade no PostgreSQL com Patroni

Base de conhecimento **didática** sobre alta disponibilidade (HA) no PostgreSQL
usando **Patroni + etcd + HAProxy + Keepalived**. Escrita para quem está começando
(estagiários e juniores): cada documento explica **o que é**, **por que importa**,
**como testar** e **o que observar**, com saídas reais do nosso laboratório.

> Laboratório usado: 3 VMs Fedora 42, PostgreSQL 18.4, cluster `pgha`
> (db-01 = 192.168.1.15, db-02 = 192.168.1.17, db-03 = 192.168.1.18),
> VIP 192.168.1.10. Montagem completa em
> [00-instalacao-cluster-patroni.md](00-instalacao-cluster-patroni.md).

## Por onde começar

Leia na ordem. Os primeiros são conceito; do 04 em diante são os **cenários de
falha** que todo DBA precisa entender.

| # | Documento | O que você aprende |
|---|-----------|--------------------|
| 00 | [Instalação do cluster](00-instalacao-cluster-patroni.md) | Como montar o ambiente do zero (runbook) |
| 01 | [Conceitos fundamentais](01-conceitos-fundamentais.md) | Primary, réplica, failover x switchover, quorum, papéis dos componentes |
| 02 | [Entendendo o `patronictl list`](02-entendendo-o-cluster.md) | Ler o estado do cluster: Role, State, Timeline, LSN, Lag |
| 03 | [Replicação e consistência](03-replicacao-e-consistencia.md) | Como os dados viajam; medir atraso; réplica é só-leitura |
| 04 | [Switchover planejado](04-switchover-planejado.md) | Trocar o primary de propósito, sem perder dados |
| 05 | [Failover automático](05-failover-automatico.md) | O que acontece quando o primary cai sozinho |
| 06 | [Reintegração de nós](06-reintegracao-de-nos.md) | Como um nó que caiu volta ao cluster (pg_rewind) |
| 07 | [Falha de uma réplica](07-falha-de-replica.md) | Perder uma réplica sem derrubar o banco |
| 08 | [Manutenção (pause/resume)](08-manutencao-pause-resume.md) | Mexer no cluster sem disparar failover acidental |
| 09 | [Perda de quorum e split-brain](09-perda-de-quorum-split-brain.md) | Por que 3 nós; o cenário mais perigoso e como o Patroni protege |
| 10 | [Cheat sheet de comandos](10-cheatsheet-comandos.md) | A cola do dia a dia |

## O problema que a HA resolve, em uma frase

Um banco com **um único servidor** é um **ponto único de falha**: se ele cai,
a aplicação inteira para. Alta disponibilidade é ter **cópias prontas para
assumir** automaticamente, de forma que a parada seja de **segundos**, não de horas.

## Como ler os testes deste material

Em todos os documentos de cenário você verá este padrão:

- **O que é** — a ideia em linguagem simples.
- **Por que importa** — o risco real que isso cobre.
- **Como reproduzir** — os comandos exatos.
- **O que você vê** — a saída real do laboratório.
- **Lições** — o que levar para a vida.
- **Pegadinhas** — onde os iniciantes erram.

## Um aviso importante

Alta disponibilidade **não é backup**. HA protege contra a queda de um servidor;
**não** protege contra um `DELETE` errado, corrupção lógica ou a perda dos 3 nós
ao mesmo tempo. Backup é um assunto separado (e obrigatório).
