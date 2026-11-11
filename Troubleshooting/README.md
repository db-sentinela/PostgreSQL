# Troubleshooting de PostgreSQL

Base de conhecimento **didática** para diagnosticar problemas no dia a dia do
PostgreSQL: **bloqueios (locks), deadlocks, transações presas e queries lentas**.
Cada documento explica **o que é → por que importa → como reproduzir → o que você
vê → lições → pegadinhas**, com saídas **reais** de um laboratório.

> Laboratório: cluster `pgha` (3 nós Fedora 42, PostgreSQL 18.4). Todos os testes
> de lock/deadlock rodam no **primary** (db-01), pois escrita e locks de escrita só
> acontecem lá. Objetos de teste no schema `tshoot`.

## Índice

| # | Documento | O que você aprende |
|---|-----------|--------------------|
| 01 | [Caixa de ferramentas](01-caixa-de-ferramentas.md) | As visões e queries que respondem "o que está acontecendo agora?" |
| 02 | [Locks e bloqueios](02-locks-e-bloqueios.md) | O que trava uma query e como achar **quem bloqueia quem** |
| 03 | [Cadeia de bloqueios e como matar](03-cadeia-de-bloqueios-e-como-matar.md) | Bloqueios em cascata; `pg_cancel_backend` x `pg_terminate_backend` |
| 04 | [Deadlocks](04-deadlocks.md) | Simular, ler o erro e o log, entender a detecção automática e **prevenir** |
| 05 | [Idle in transaction](05-idle-in-transaction.md) | A transação esquecida que trava tudo (e por que `cancel` não resolve) |
| 06 | [Queries lentas e EXPLAIN](06-queries-lentas-e-explain.md) | Achar e cancelar queries longas; ler um plano; o efeito de um índice |
| 07 | [Cheat sheet](07-cheatsheet-troubleshooting.md) | A cola de queries de diagnóstico |

## Metodologia: os 4 passos de qualquer investigação

Quando "o banco está lento" ou "travou", siga sempre esta ordem, do mais rápido
para o mais profundo:

1. **O que está rodando agora?** → `pg_stat_activity`. Veja estados (`active`,
   `idle in transaction`), há quanto tempo, e se há `wait_event`.
2. **Tem alguém bloqueado?** → `pg_blocking_pids()` + `pg_locks`. Descubra a
   relação bloqueado → bloqueador.
3. **O que o banco registrou?** → **logs** do PostgreSQL. Deadlocks, erros,
   queries lentas e conexões ficam ali.
4. **É recurso de máquina?** → CPU, disco e memória do servidor (`htop`,
   `iostat`, `vmstat`). Às vezes o "banco lento" é disco cheio ou I/O saturado.

> Regra de ouro: **primeiro observe, depois aja.** Nunca saia matando processo no
> susto — identifique a causa (passos 1-3) antes de usar `terminate`.

## Onde procurar (mapa de bolso)

| Pergunta | Onde olhar |
|----------|------------|
| O que está executando agora? | `pg_stat_activity` |
| Quem bloqueia quem? | `pg_blocking_pids()`, `pg_locks` |
| Houve deadlock / erro? | Log do PostgreSQL: `/var/lib/pgsql/18/data/log/*.log` |
| Qual o arquivo de log atual? | `SELECT pg_current_logfile();` |
| A query usa índice? | `EXPLAIN (ANALYZE, BUFFERS) <query>;` |
| Estatísticas da tabela | `pg_stat_user_tables`, `pg_statio_user_tables` |
| Saúde do cluster (HA) | `patronictl list` (ver pasta `../alta-disponibilidade/`) |

## Importante antes de mexer

- Em **produção**, `pg_terminate_backend` **derruba a conexão** de alguém — pode
  abortar uma transação legítima. Tenha certeza do que está matando.
- Rode os diagnósticos como um usuário com permissão (superusuário enxerga tudo;
  usuários comuns só as próprias sessões, salvo papel `pg_monitor`).
