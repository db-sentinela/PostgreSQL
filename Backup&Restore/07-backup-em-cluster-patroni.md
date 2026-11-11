# 07 — Backup em um cluster Patroni (HA)

Fazer backup num cluster de alta disponibilidade (ver `../alta-disponibilidade/`)
tem particularidades. Este documento junta os dois mundos.

## Primeiro: HA **não** é backup

Vale repetir porque é o erro conceitual mais comum. O cluster Patroni replica tudo
em tempo real — **inclusive** o seu `DROP TABLE` e o `DELETE` sem `WHERE`. A réplica
não te salva de um erro lógico. **Você precisa de backup de verdade, sempre.**

## Onde rodar o backup: na réplica, não no primary

Backup físico (`pg_basebackup`/pgBackRest) **lê** muito disco. Rodar isso no
**primary** rouba I/O das transações da aplicação. A boa prática:

> Faça o backup a partir de uma **réplica** (standby). Ela tem os mesmos dados e
> está ociosa de escrita. Alivia o primary.

No nosso cluster, as réplicas são db-02 e db-03. Você aponta o `pg_basebackup`
(ou o `pg1-host` do pgBackRest) para uma delas.

> Atenção: `pg_dump` (lógico) numa réplica pode conflitar com a replicação se a query
> for longa (cancelamento por conflito). Para dumps longos na réplica, considere
> `hot_standby_feedback = on` (já ligado no nosso lab) ou
> `max_standby_streaming_delay`.

## Configurar archiving no cluster: pelo Patroni, não no nó

Em um cluster Patroni, **não** edite o `postgresql.conf` de um nó na mão — o Patroni
gerencia a configuração e sobrescreve. Parâmetros como `archive_mode` e
`archive_command` precisam valer para **todo o cluster** e sobreviver a failover:

```bash
patronictl -c /etc/patroni/patroni.yml edit-config
```

e defina, na seção `postgresql.parameters`:

```yaml
    archive_mode: "on"
    archive_command: "pgbackrest --stanza=pgha archive-push %p"
    wal_level: replica
```

> `archive_mode` exige **restart** do PostgreSQL. O Patroni faz isso de forma
> controlada: `patronictl restart pgha` (idealmente nó a nó, começando pelas
> réplicas, ou numa janela com `pause`).

Assim, **qualquer** nó que vire primary já sabe arquivar o WAL — o archiving
**acompanha o primary** no failover.

## pgBackRest + Patroni (arranjo recomendado)

- O `archive_command` (via Patroni) empurra o WAL para o repositório pgBackRest a
  partir de **quem for o primary** no momento.
- O **repositório** fica num servidor dedicado / storage remoto (S3, NFS), não no
  disco de um dos nós (3-2-1).
- Os **backups** são tirados de uma réplica (`pg1-host` apontando para o standby).
- O pgBackRest lida com a troca de primary naturalmente, porque o WAL chega ao repo
  independentemente de qual nó o gerou.

## Restaurar num cluster Patroni

Aqui mora a pegadinha mais importante:

> **PITR/restore físico é feito FORA do controle do Patroni.** Você não "restaura
> dentro" de um cluster rodando.

Fluxo típico de recuperação de desastre:

1. **Pare o Patroni** nos nós (ou trabalhe num ambiente isolado) — você não quer o
   Patroni tentando gerenciar um banco que está em recuperação.
2. **Restaure** o backup (pgBackRest/`pg_basebackup` + WAL) para o `data_dir` de
   **um** nó, com o alvo de PITR desejado, e deixe-o recuperar e promover.
3. **Recrie as réplicas** a partir desse nó já recuperado:
   `patronictl reinit pgha <replica>` (faz um novo base backup do novo primary).
4. **Religue o Patroni** e confirme com `patronictl list` (um Leader, réplicas em
   `streaming`, mesma timeline).

Em outras palavras: **restaura-se um nó**, e o cluster é **reconstruído** em volta
dele.

## Para testar PITR sem derrubar o cluster

Foi o que fizemos na homologação (docs [05](05-pitr.md) e [06](06-pgbackrest.md)):
restaurar o backup numa **instância separada** (outra porta/`data_dir`), validar que
os dados voltaram, e **só então** decidir o que fazer no cluster real. Restaurar num
ambiente à parte é a forma segura de:

- treinar o procedimento,
- validar que o backup presta,
- extrair uma tabela específica de um backup físico (restaura à parte e faz
  `pg_dump` da tabela).

## Lições

- HA não substitui backup — a réplica copia seus erros lógicos.
- Faça backup **da réplica** para poupar o primary.
- Configure `archive_mode`/`archive_command` **pelo `patronictl edit-config`**
  (vale para o cluster e sobrevive a failover).
- Restore/PITR acontece **fora** do Patroni: restaura um nó, depois `reinit` das
  réplicas.

## Pegadinhas

- Editar `postgresql.conf` de um nó na mão: o Patroni **sobrescreve**. Use
  `edit-config`.
- Tentar restaurar "por cima" de um nó com o Patroni ativo: ele vai brigar com a
  recuperação. Pare o Patroni ou use ambiente isolado.
- Guardar o repositório de backup **nos próprios nós** do cluster viola o 3-2-1 —
  um incêndio/ransomware leva tudo junto.
