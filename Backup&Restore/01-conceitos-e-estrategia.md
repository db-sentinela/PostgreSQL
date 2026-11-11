# 01 — Conceitos e estratégia

Antes dos comandos, a cabeça certa. Backup é **gestão de risco**: você decide
**quanto** pode perder e **quão rápido** precisa voltar — e escolhe a estratégia
que entrega isso.

## RPO e RTO (os dois números que definem tudo)

- **RPO** (Recovery Point Objective) — **quanto de dado** você aceita perder,
  medido em tempo. "RPO de 5 min" = no pior caso, perco os últimos 5 minutos.
- **RTO** (Recovery Time Objective) — **quanto tempo** você aceita ficar fora até
  restaurar. "RTO de 1 h" = em até 1 hora o banco está de volta.

Esses dois números **guiam a escolha**:

| Necessidade | Estratégia típica |
|-------------|-------------------|
| RPO de horas, RTO de horas | `pg_dump` diário |
| RPO de minutos | Backup físico **+ WAL archiving** (PITR) |
| RPO perto de zero | PITR com archiving frequente / replicação síncrona |
| RTO baixo | Backup físico (restaura rápido) + réplica de HA |

## Os três tipos (e quando usar cada um)

### Lógico (`pg_dump`)
- **É** um arquivo com os comandos SQL (ou dados) para recriar objetos.
- **Vantagens**: granular (um objeto, um schema, um banco), portável entre versões
  e arquiteturas, ótimo para **migração** e para restaurar **uma tabela**.
- **Desvantagens**: **lento** para bancos grandes (reconstrói tudo), não serve para
  PITR, consome CPU ao restaurar (recria índices).
- **Use para**: bancos pequenos/médios, migrações, cópia de um objeto específico.

### Físico (`pg_basebackup`, pgBackRest)
- **É** uma cópia dos arquivos do cluster (byte a byte).
- **Vantagens**: **rápido** de restaurar (é só copiar de volta), captura o cluster
  inteiro exatamente como está.
- **Desvantagens**: é **tudo ou nada** (o cluster inteiro; não restaura "uma
  tabela"), preso à mesma versão major e arquitetura.
- **Use para**: bancos grandes, base para PITR, recuperação rápida.

### PITR (físico + WAL contínuo)
- **É** um backup físico **somado** ao arquivo contínuo do **WAL** (o diário de
  todas as mudanças).
- **Vantagens**: recupera até um **instante exato** ("segundos antes do `DROP`").
  É o padrão-ouro para RPO baixo.
- **Desvantagens**: mais peças para configurar e monitorar (archiving).
- **Use para**: produção séria. É o que salva de um erro humano às 14:32.

> Regra prática: **lógico para granularidade e migração; físico/PITR para
> produção e desastre.** Muitas empresas fazem **os dois**.

## A regra 3-2-1

Uma política de retenção sólida:

- **3** cópias dos dados (a produção + 2 backups),
- em **2** mídias/locais diferentes,
- sendo **1** fora do local (offsite / outro datacenter / nuvem).

Se um ransomware ou um incêndio pegar o servidor **e** o backup que está do lado,
a cópia offsite te salva.

## Retenção

Defina **por quanto tempo** guardar cada backup. Exemplo comum:

- Full **diário**, retido por 7 dias.
- Full **semanal**, retido por 4 semanas.
- Full **mensal**, retido por 12 meses.
- WAL retido o suficiente para o PITR cobrir a janela dos fulls.

Ferramentas como o pgBackRest automatizam isso (`repo1-retention-full`).

## "Backup não testado não é backup"

O erro nº 1 das empresas: ter backups que **nunca foram restaurados** — e
descobrir no dia do desastre que estavam corrompidos, incompletos ou que ninguém
sabia o procedimento. **Homologue a restauração** periodicamente:

- Restaure num ambiente separado e **compare** (contagens, checksums).
- Cronometre o restore (isso é o seu RTO **real**, não o teórico).
- Documente o passo a passo (é o que este repositório faz).

## Lições

- Comece por **RPO/RTO**: eles ditam a estratégia.
- Lógico = granular/portável; físico = rápido/cluster inteiro; PITR = instante exato.
- Siga **3-2-1** e uma política de **retenção**.
- **Teste o restore** — sempre. Backup é o restore, não o arquivo.

## Pegadinhas

- **HA/replicação não é backup.** A réplica copia o `DROP TABLE` em milissegundos.
- `pg_dump` de um banco **não** inclui *roles* e permissões globais — para isso
  existe `pg_dumpall --globals-only` (doc [02](02-backup-logico-pg_dump.md)).
- Guardar o backup **no mesmo disco** do banco é quase não ter backup.
