# 06 — VACUUM, autovacuum e bloat

Este é um dos temas que mais causa "lentidão misteriosa" em produção. Entender o
`VACUUM` é obrigatório.

## Por que o VACUUM existe (MVCC)

O PostgreSQL usa **MVCC**: quando você faz `UPDATE` ou `DELETE`, a versão antiga da
linha **não** é apagada na hora — ela vira uma **linha morta** (*dead tuple*),
porque transações em andamento ainda podem precisar vê-la. O `VACUUM` é quem
**limpa** essas linhas mortas depois.

## Teste real: UPDATE gera lixo (bloat)

```sql
-- tabela com 500k linhas (autovacuum desligado para enxergar o acúmulo):
CREATE TABLE perf.bloat (id int, v text);
ALTER TABLE perf.bloat SET (autovacuum_enabled=false);
INSERT INTO perf.bloat SELECT g, md5(g::text) FROM generate_series(1,500000) g;
```

```
## início:        tamanho=33 MB   (500000 vivas / 0 mortas)
UPDATE perf.bloat SET v = md5(random()::text);   -- 2x
## após 2 UPDATEs: tamanho=98 MB   (500000 vivas / 1000000 mortas)
```

Dois `UPDATE` em todas as linhas → a tabela **triplicou** (33 → 98 MB) e ganhou **1
milhão de linhas mortas**. Isso é **bloat**: espaço ocupado por lixo. Bloat deixa
tudo mais lento (mais páginas para ler).

## VACUUM x VACUUM FULL — testado

```
VACUUM perf.bloat;
## após VACUUM:      0 mortas, 98 MB   <- limpou as mortas, mas o ARQUIVO continua 98 MB
VACUUM FULL perf.bloat;
## após VACUUM FULL: 0 mortas, 33 MB   <- reescreveu e ENCOLHEU de volta
```

| | `VACUUM` | `VACUUM FULL` |
|---|---|---|
| O que faz | Marca o espaço morto como **reutilizável** | **Reescreve** a tabela do zero |
| Devolve espaço ao SO? | **Não** (arquivo continua do mesmo tamanho) | **Sim** (encolhe) |
| Trava a tabela? | Não (roda junto com a aplicação) | **Sim** — `AccessExclusiveLock` (bloqueia tudo!) |
| Uso | Rotineiro (o autovacuum faz) | Emergência de bloat, em janela de manutenção |

> **Nunca** rode `VACUUM FULL` despreocupado em produção: ele **trava a tabela
> inteira**. Para desinchar sem travar, use a extensão `pg_repack`.

## A armadilha real: quando o VACUUM NÃO limpa — testado ao vivo

No nosso teste, aconteceu algo instrutivo: logo após os UPDATEs, **nem o VACUUM nem
o VACUUM FULL** conseguiram remover as linhas mortas. Investigando:

```sql
SELECT slot_name, active, xmin FROM pg_replication_slots;
```
```
 db_02 | t | 12671
 db_03 | t | 12671        <-- os slots de replicação seguravam o xmin em 12671
SHOW hot_standby_feedback;  -> on
```

O `VACUUM (VERBOSE)` confirmou:
```
tuples: ... are dead but not yet removable
removable cutoff: 12671
```

**O que estava acontecendo:** uma réplica (db-02) estava **ressincronizando** e, com
`hot_standby_feedback = on`, segurava o **horizonte de xmin** do primary. O
PostgreSQL **não pode** remover uma linha morta que uma réplica/transação antiga
ainda enxerga. Resultado: mortas acumulam mesmo com VACUUM rodando. Assim que a
réplica terminou de sincronizar, o horizonte avançou e o `VACUUM` removeu **1.000.000**
de linhas de uma vez.

> Esse é **o** motivo nº 1 de "minha tabela incha e o VACUUM não resolve" em
> produção. As três causas de um horizonte travado:
> 1. **Replication slot** parado/atrasado (`pg_replication_slots.xmin`).
> 2. **Transação longa / `idle in transaction`** (ver `../Troubleshooting/`).
> 3. **`hot_standby_feedback`** com query longa numa réplica.

## Autovacuum: quem faz isso por você

Normalmente você **não** roda VACUUM na mão — o **autovacuum** faz, sozinho, em
background. Ele dispara quando as linhas mortas passam de um limite:

```
mortas > autovacuum_vacuum_threshold (50) + autovacuum_vacuum_scale_factor (0.2) × n_linhas
```

Ou seja, por padrão ele espera **20%** da tabela virar lixo. Para **tabelas grandes
e movimentadas**, 20% é muito — ajuste **por tabela**:

```sql
ALTER TABLE pedidos SET (autovacuum_vacuum_scale_factor = 0.02);  -- vacuum a cada 2%
```

Monitore se o autovacuum está dando conta:
```sql
SELECT relname, n_dead_tup, last_autovacuum, autovacuum_count
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;
```

## Lições

- `UPDATE`/`DELETE` geram **linhas mortas** (MVCC); o VACUUM limpa.
- `VACUUM` marca reutilizável (não encolhe); `VACUUM FULL` encolhe mas **trava**.
- VACUUM não limpa o que um **slot/transação/réplica** ainda enxerga (horizonte de
  xmin) — a causa clássica de bloat persistente.
- O **autovacuum** faz o trabalho; ajuste `scale_factor` **por tabela** nas grandes.

## Pegadinhas

- `VACUUM FULL` em produção sem janela = **incidente** (trava a tabela).
- Slot de replicação de uma réplica que **sumiu** segura o xmin **para sempre** e
  incha o banco **e** enche o `pg_wal`. Monitore `pg_replication_slots`.
- Desligar autovacuum "para ganhar performance" é um tiro no pé — o bloat volta
  multiplicado.
