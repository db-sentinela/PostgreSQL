# 08 — Cheat sheet e checklist de homologação

## Backup lógico (pg_dump / pg_dumpall)

```bash
PG=/usr/pgsql-18/bin
# um banco, formato custom (recomendado):
$PG/pg_dump -Fc -d meubanco -f meubanco.dump
# directory, paralelo:
$PG/pg_dump -Fd -j4 -d meubanco -f meubanco_dir
# roles e objetos globais (NÃO esqueça):
$PG/pg_dumpall --globals-only -f globals.sql
# ver o conteúdo do dump sem restaurar:
$PG/pg_restore --list meubanco.dump
```

## Restore lógico

```bash
# banco inteiro (cria o banco):
$PG/pg_restore -C -d postgres meubanco.dump
# só uma tabela:
$PG/pg_restore -d meubanco -t clientes meubanco.dump
# formato plain (.sql):
$PG/psql -d meubanco -f meubanco.sql
```

## Backup físico (pg_basebackup)

```bash
# destino PRECISA pertencer ao postgres:
sudo -u postgres install -d /dest/basebackup
PGPASSWORD=senha pg_basebackup -h <ip-replica> -U replicator \
   -D /dest/basebackup -Fp -Xs -c fast -P
# verificar integridade:
pg_verifybackup /dest/basebackup
```

## WAL / archiving

```ini
# postgresql.conf
wal_level = replica
archive_mode = on            # exige RESTART
archive_command = 'test ! -f /arch/%f && cp %p /arch/%f'
restore_command = 'cp /arch/%f %p'   # usado na recuperação
```
```sql
SELECT pg_switch_wal();            -- força arquivar o segmento atual
SELECT * FROM pg_stat_archiver;    -- MONITORE failed_count!
```

## PITR manual

```ini
# no data_dir restaurado do base backup:
restore_command = 'cp /arch/%f %p'
recovery_target_time = '2026-11-11 14:31:59-03'
recovery_target_action = 'promote'     # ou 'pause' p/ inspecionar
```
```bash
touch /data_dir/recovery.signal && pg_ctl -D /data_dir start
```

## pgBackRest

```bash
# instalar (repo pgdg-common!):
dnf install -y pgbackrest
# inicializar e validar:
pgbackrest --stanza=demo stanza-create
pgbackrest --stanza=demo check
# backups:
pgbackrest --stanza=demo --type=full backup
pgbackrest --stanza=demo --type=diff backup
pgbackrest --stanza=demo --type=incr backup
# PITR:
pgbackrest --stanza=demo --delta --type=time \
   --target='2026-11-11 14:31:59' --target-action=promote restore
# estado:
pgbackrest --stanza=demo info
pgbackrest --stanza=demo verify
```

## Em cluster Patroni

```bash
# archiving vale para o cluster (não edite o .conf do nó na mão):
patronictl -c /etc/patroni/patroni.yml edit-config
# recriar uma réplica após restaurar um nó:
patronictl -c /etc/patroni/patroni.yml reinit pgha db-02
```

---

# Checklist de homologação de backup

Use isto para **provar** que sua estratégia de backup funciona. Um item não marcado
é um risco aberto.

## Configuração
- [ ] `archive_mode = on` e `archive_command` funcionando (teste com `pg_switch_wal`).
- [ ] `pg_stat_archiver.failed_count = 0` e monitorado com alerta.
- [ ] Backup roda **agendado** (cron / timer), não só manual.
- [ ] Backup inclui **globais** (`pg_dumpall --globals-only`) quando for lógico.
- [ ] Repositório de backup está **fora** do servidor do banco (regra 3-2-1).
- [ ] Política de **retenção** definida (fulls + WAL cobrindo a janela).

## Restore (o que realmente importa)
- [ ] Já restaurei um backup **completo** num ambiente separado.
- [ ] Já restaurei **uma tabela** isolada (lógico).
- [ ] Já fiz um **PITR** até um instante escolhido e **validei** os dados.
- [ ] **Cronometrei** o restore → conheço meu **RTO real**.
- [ ] Verifiquei integridade (`pg_verifybackup` / `pgbackrest verify`).
- [ ] O procedimento de restore está **documentado** e qualquer um da equipe consegue
      seguir.

## Para o cluster HA
- [ ] Backup é tirado da **réplica**, não do primary.
- [ ] `archive_command` configurado **via Patroni** (sobrevive a failover).
- [ ] Testei reconstruir réplicas com `reinit` após um restore.

> **Lembrete final:** a pergunta certa nunca é "você tem backup?", e sim
> **"quando foi a última vez que você restaurou?"**. Backup é o restore.
