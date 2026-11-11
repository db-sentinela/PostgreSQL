# 04 — WAL e archiving (a base do PITR)

## O que é o WAL

**WAL** = *Write-Ahead Log*, o **diário** do PostgreSQL. **Antes** de mudar qualquer
dado nos arquivos, o banco escreve a mudança no WAL. É o que garante durabilidade
(se faltar energia, o banco relê o WAL e se recupera) e é o que a replicação envia
para as réplicas.

O WAL é dividido em **segmentos** de 16 MB em `pg_wal/`. Normalmente o PostgreSQL
**recicla** (reusa/apaga) segmentos antigos. Para o PITR, a gente precisa **guardar
uma cópia** de cada segmento antes que ele seja reciclado — isso é o **archiving**.

## Por que importa

- Sem archiving, você só tem a "foto" do base backup. **Com** archiving, você tem a
  foto **mais** o filme contínuo de tudo que mudou → recupera a **qualquer instante**
  (PITR).
- Também é usado para alimentar réplicas que ficaram muito atrás.

## Como ligar o archiving

No `postgresql.conf`:

```ini
wal_level = replica          # mínimo para archiving/replicação (padrão já é 'replica')
archive_mode = on            # liga o arquivamento (MUDAR exige restart)
archive_command = 'test ! -f /arquivo/archive/%f && cp %p /arquivo/archive/%f'
```

Como ler o `archive_command`:

- `%p` → caminho completo do segmento WAL a arquivar (origem).
- `%f` → só o nome do arquivo (destino).
- `test ! -f .../%f &&` → só copia se **ainda não existe** lá (não sobrescreve).
- O comando deve retornar **0** em sucesso. Se falhar, o PostgreSQL **tenta de novo**
  e **segura** o WAL (não recicla) até conseguir.

> `archive_mode` só muda com **restart**. Em cluster Patroni, altere via
> `patronictl edit-config` e deixe o Patroni reiniciar (doc [07]).

## Testar que o archiving funciona (comprovado no PITR)

Force a troca de segmento e veja ele aparecer no diretório de archive:

```sql
SELECT pg_switch_wal();      -- fecha o segmento atual -> dispara o archive_command
```

```bash
ls -l /arquivo/archive/      # o segmento recém-fechado deve estar aqui
```

No teste de PITR (doc [05](05-pitr.md)), foi exatamente esse archiving que permitiu
reaplicar o WAL e recuperar a tabela dropada. O log de recuperação mostrou o WAL
sendo lido do archive até o instante-alvo:

```
LOG:  starting point-in-time recovery to 2026-11-11 19:04:36.999577-03
LOG:  restored log file "000000010000000000000003" from archive
LOG:  recovery stopping before commit of transaction 755 ...
```

## O caminho de volta: `restore_command`

Na **recuperação**, o PostgreSQL busca os segmentos no archive com o inverso do
`archive_command`:

```ini
restore_command = 'cp /arquivo/archive/%f %p'
```

Aqui `%f` é o arquivo que ele **quer** e `%p` é onde deve **colocá-lo**. É assim que
o PITR "reaplica o filme".

## Monitorar o archiving (obrigatório em produção)

```sql
SELECT * FROM pg_stat_archiver;
```

Olhe `archived_count` (sucessos), `last_archived_time`, e principalmente
**`failed_count`** / `last_failed_time`. Se `failed_count` sobe, o archive está
quebrado — e o WAL vai **se acumular**.

## O perigo nº 1 do archiving

Se o `archive_command` **falha** (destino cheio, sem permissão, rede caída), o
PostgreSQL **não recicla** o WAL — ele segura tudo em `pg_wal/` esperando arquivar.
O `pg_wal/` **cresce** e pode **encher o disco**, e aí **o banco para**.

> **Monitore `pg_stat_archiver.failed_count` e o espaço em `pg_wal/`.** Archiving
> sem monitoramento é uma bomba-relógio.

## Lições

- WAL = diário de todas as mudanças; archiving = guardar cada segmento antes de
  reciclar.
- `archive_mode` + `archive_command` ligam o arquivamento (restart necessário).
- `restore_command` é o caminho de volta, usado no PITR.
- Teste com `pg_switch_wal()`; monitore com `pg_stat_archiver`.

## Pegadinhas

- `archive_command` que falha **enche o disco** com WAL retido — monitore sempre.
- Um `archive_command` que retorna 0 **sem** ter copiado de verdade cria um "buraco"
  no archive e **quebra o PITR** silenciosamente. Teste o comando de verdade.
- Trocar `archive_mode` exige **restart** (não basta `reload`).
- Guarde o archive **fora** do servidor do banco (senão perde os dois juntos).
