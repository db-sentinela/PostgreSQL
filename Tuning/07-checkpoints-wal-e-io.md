# 07 — Checkpoints, WAL e I/O

Este doc trata de **escrita em disco**: por que às vezes o banco dá "engasgos"
periódicos de I/O, e como suavizá-los.

## O que é um checkpoint

O PostgreSQL não grava cada mudança direto nos arquivos de dados na hora — primeiro
escreve no **WAL** (rápido, sequencial) e mantém as páginas alteradas na memória
(*dirty pages*). De tempos em tempos, um **checkpoint** despeja todas essas páginas
sujas para os arquivos de dados. Esse despejo é um **pico de I/O**.

## Por que importa

Checkpoints mal configurados causam **picos periódicos** de escrita: a cada X
minutos o disco satura, as queries ficam lentas, e depois normaliza. O objetivo do
tuning é **espalhar** essa escrita para não doer.

## Os parâmetros (valores reais do nosso lab)

```
checkpoint_timeout           = 300    (5 min)     # de quanto em quanto tempo
checkpoint_completion_target = 0.9                # espalha a escrita em 90% do intervalo
max_wal_size                 = 1024   (1 GB)      # força checkpoint se o WAL crescer tanto
min_wal_size                 = 80     (80 MB)
wal_compression              = off
```

### checkpoint_timeout
Intervalo máximo entre checkpoints (padrão 5 min). **Aumentar** (ex.: 15 min) faz
checkpoints menos frequentes → menos picos, porém recuperação após crash um pouco
mais longa. Comum em produção: **15–30 min**.

### max_wal_size
Se o WAL acumular mais que isso **entre** checkpoints, um checkpoint é disparado
**antes** do tempo. Se você vê checkpoints frequentes "por volume", **aumente**
`max_wal_size` (ex.: 4–8 GB) para não disparar à toa.

### checkpoint_completion_target
Espalha a escrita do checkpoint ao longo do intervalo. `0.9` (padrão moderno) =
escreve devagar durante 90% do tempo até o próximo → **picos menores**. Deixe em 0.9.

## Diagnóstico: meus checkpoints estão saudáveis?

```sql
-- PostgreSQL 17+ : estatísticas de checkpoint
SELECT num_timed, num_requested, write_time, sync_time
FROM pg_stat_checkpointer;
```

- **`num_requested`** alto em relação a **`num_timed`** = checkpoints sendo forçados
  por **volume de WAL** (não por tempo) → **aumente `max_wal_size`**.
- `num_timed` dominante = checkpoints no ritmo do `checkpoint_timeout` (saudável).

> Ligue `log_checkpoints = on` (padrão no PG moderno) para registrar cada checkpoint
> no log, com quanto escreveu e quanto demorou.

## Para onde vai meu I/O? (pg_stat_io — PG 16+)

```sql
SELECT backend_type, object, context,
       reads, writes, extends
FROM pg_stat_io
WHERE reads > 0 OR writes > 0
ORDER BY writes DESC NULLS LAST LIMIT 10;
```

Mostra **quem** gera I/O: o `checkpointer`, o `background writer`, o `autovacuum`,
ou os backends normais. Ótimo para saber se o problema é checkpoint, vacuum ou carga.

## Outros ajustes de I/O

- **`wal_compression = on`** — comprime o WAL (menos I/O de WAL, um pouco de CPU).
  Vale a pena na maioria dos casos; no nosso lab está `off`.
- **`random_page_cost`** — custo estimado de uma leitura aleatória (padrão **4**,
  pensado para HDD). Em **SSD/NVMe**, baixe para **1.1**: o planejador passa a
  preferir índices com mais frequência (leituras aleatórias são baratas em SSD).
- **`effective_io_concurrency`** — quantas leituras simultâneas o disco aguenta
  (no lab está **16**; em SSD/NVMe, valores altos ajudam prefetch).

## Ponto de partida para produção (SSD)

```ini
checkpoint_timeout = '15min'
max_wal_size = '8GB'
checkpoint_completion_target = 0.9
wal_compression = on
random_page_cost = 1.1          # SSD/NVMe
effective_io_concurrency = 200  # SSD/NVMe
```

## Lições

- Checkpoint = despejo periódico de páginas sujas → **pico de I/O**.
- `max_wal_size` maior + `checkpoint_timeout` maior = checkpoints mais raros e
  espalhados (menos engasgos).
- Em **SSD**, baixe `random_page_cost` para ~1.1 (o planejador usa mais índice).
- Diagnostique com `pg_stat_checkpointer` e `pg_stat_io`.

## Pegadinhas

- `checkpoint_timeout` muito alto → recuperação após crash mais demorada e mais WAL
  retido. É um trade-off, não "quanto maior melhor".
- `random_page_cost = 4` em servidor **SSD** faz o planejador evitar índices sem
  motivo — um dos ajustes mais esquecidos.
- `max_wal_size` é um **alvo**, não um limite rígido; o `pg_wal` pode passar disso
  temporariamente.
