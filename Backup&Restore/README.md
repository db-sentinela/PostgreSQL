# Backup & Restore no PostgreSQL

Base de conhecimento **didática** e completa sobre o tema **mais crítico** da
administração de banco: proteger os dados e saber **recuperá-los**. Cobre backup
**lógico**, **físico** e **PITR** (Point-In-Time Recovery), além da ferramenta de
produção **pgBackRest** — tudo com cenários **testados de verdade** no laboratório
(inclusive recuperar uma tabela dropada "no instante anterior ao desastre").

> Laboratório: PostgreSQL 18.4 em Fedora 42. Banco de exemplo `lojadb`
> (clientes + pedidos) e instâncias standalone para os testes de PITR, para **não**
> perturbar o cluster Patroni de HA.

## A regra que você nunca pode esquecer

> **Backup que nunca foi restaurado não é backup — é esperança.**
> A única prova de que um backup presta é **restaurá-lo**. Teste a restauração
> regularmente (este repositório mostra como).

## Índice

| # | Documento | O que você aprende |
|---|-----------|--------------------|
| 01 | [Conceitos e estratégia](01-conceitos-e-estrategia.md) | Lógico x físico x PITR; RPO/RTO; regra 3-2-1; retenção; testar restore |
| 02 | [Backup lógico — pg_dump](02-backup-logico-pg_dump.md) | `pg_dump`/`pg_restore`/`pg_dumpall`: formatos, restore completo e **seletivo** |
| 03 | [Backup físico — pg_basebackup](03-backup-fisico-pg_basebackup.md) | Cópia física do cluster inteiro; quando usar; formatos |
| 04 | [WAL e archiving](04-wal-e-archiving.md) | O diário WAL, `archive_mode`/`archive_command` — a base do PITR |
| 05 | [PITR — Point-In-Time Recovery](05-pitr.md) | O cenário "dropei a tabela às 14:32": recuperar no **tempo exato** (testado) |
| 06 | [pgBackRest](06-pgbackrest.md) | A ferramenta de produção: backup full/incremental, restore e PITR (testado) |
| 07 | [Backup em cluster Patroni](07-backup-em-cluster-patroni.md) | Onde e como fazer backup num ambiente de HA |
| 08 | [Cheat sheet e checklist](08-cheatsheet-e-checklist.md) | Comandos + checklist de homologação de backup |

## Os três tipos, em uma tabela

| Tipo | O que copia | Granularidade do restore | Ferramenta |
|------|-------------|--------------------------|------------|
| **Lógico** | Comandos SQL que recriam os objetos e dados | Um objeto, um schema, um banco | `pg_dump`, `pg_dumpall` |
| **Físico** | Os arquivos do cluster (byte a byte) | O **cluster inteiro** | `pg_basebackup`, pgBackRest |
| **PITR** | Backup físico **+ WAL contínuo** | Um **instante** no tempo | `pg_basebackup`+WAL, pgBackRest |

## Como ler este material

Cada documento segue: **o que é → por que importa → como reproduzir → o que você
vê (saída real) → lições → pegadinhas**. Comece pelo
[01 — Conceitos](01-conceitos-e-estrategia.md).

## Lembrete de contexto

Este é um dos sub-repositórios de administração PostgreSQL. Veja também
`../alta-disponibilidade/` (HA com Patroni) e `../troubleshooting/`. **HA não é
backup**: replicação copia **tudo**, inclusive o seu `DROP TABLE` — só o backup te
salva de um erro lógico.
