# Implantacao do ZDL/ZDLRA para Bancos Oracle em Exadata

Este repositorio documenta um procedimento base para implantar protecao de backup e recuperacao com Oracle Zero Data Loss Recovery Appliance (ZDLRA), tambem chamada de Recovery Appliance, para bancos Oracle executando em Oracle Exadata.

> Ajuste os nomes de hosts, servicos, usuarios, wallets, politicas e caminhos conforme o padrao do seu ambiente.

## Objetivo

- Proteger bancos Oracle em Exadata com backups incrementais forever.
- Habilitar envio de redo para reduzir ou eliminar perda de dados em cenarios de recuperacao.
- Padronizar configuracao de rede, RMAN, catalogo e politicas de retencao.
- Validar backup, restore e recuperacao antes da liberacao em producao.

## Escopo

Este guia cobre:

- Pre-requisitos no Exadata e na Recovery Appliance.
- Cadastro do banco protegido.
- Configuracao de conectividade e credenciais.
- Configuracao RMAN.
- Execucao do primeiro backup.
- Validacoes operacionais.
- Procedimentos basicos de troubleshooting.

Este guia nao cobre:

- Instalacao fisica da Recovery Appliance.
- Patching de Exadata, Grid Infrastructure ou banco de dados.
- Desenho definitivo de capacidade.
- Politicas corporativas de backup, seguranca e retencao legal.

## Arquitetura resumida

```text
+----------------------+        Rede de backup / client access        +----------------------+
| Oracle Exadata       | --------------------------------------------> | Recovery Appliance   |
| Banco protegido      |                                               | ZDLRA                |
| RMAN / Redo Shipping | <-------------------------------------------- | Catalogo / Storage   |
+----------------------+                                               +----------------------+
```

Componentes principais:

- **Banco protegido**: banco Oracle no Exadata que sera protegido.
- **Recovery Appliance catalog**: catalogo RMAN gerenciado pela ZDLRA.
- **Protection policy**: politica que define retencao, janela de recuperacao e comportamento de backup.
- **Virtual Private Catalog (VPC)**: acesso isolado ao catalogo para o banco protegido.
- **Redo transport**: envio de redo para aproximar o RPO de zero.

## Pre-requisitos

### Informacoes do ambiente

Preencha antes de iniciar:

| Item | Valor |
|---|---|
| Nome do cliente/projeto | `<cliente>` |
| Exadata | `<exadata-host-ou-rack>` |
| Banco/CDB | `<db_unique_name>` |
| PDBs protegidas | `<lista_pdbs>` |
| Versao Oracle Database | `<versao>` |
| Recovery Appliance | `<ra-hostname>` |
| Service name da RA | `<ra_service>` |
| Politica de protecao | `<policy_name>` |
| Usuario VPC | `<vpc_user>` |
| Janela de retencao | `<n_dias>` |
| RPO esperado | `<rpo>` |

### Requisitos tecnicos

- Banco Oracle suportado pela versao da Recovery Appliance.
- Conectividade TCP entre Exadata e Recovery Appliance.
- Resolucao de nomes configurada via DNS ou `/etc/hosts`.
- Porta do listener da Recovery Appliance liberada.
- Usuario RMAN/VPC criado e autorizado na Recovery Appliance.
- Credenciais armazenadas de forma segura, preferencialmente com Oracle Wallet.
- Espaco e throughput validados conforme volumetria do banco.
- Janela operacional aprovada para primeiro backup.

## Checklist de implantacao

- [ ] Confirmar versoes suportadas.
- [ ] Confirmar conectividade entre Exadata e Recovery Appliance.
- [ ] Criar ou validar protection policy.
- [ ] Criar usuario VPC para o banco protegido.
- [ ] Registrar o banco no catalogo da Recovery Appliance.
- [ ] Configurar wallet ou credenciais seguras.
- [ ] Configurar RMAN no banco protegido.
- [ ] Executar primeiro backup full/incremental base.
- [ ] Habilitar envio de redo, quando aplicavel.
- [ ] Executar restore/recover de validacao.
- [ ] Documentar evidencias.
- [ ] Configurar monitoramento e alertas.

## 1. Validacao de conectividade

No Exadata, valide resolucao e acesso ao listener da Recovery Appliance:

```bash
tnsping <ra_service>
```

Teste conexao SQL*Plus usando o usuario definido para o catalogo/VPC:

```bash
sqlplus <vpc_user>@<ra_service>
```

Se for utilizado Oracle Wallet:

```bash
mkstore -wrl <wallet_path> -listCredential
sqlplus /@<ra_service>
```

## 2. Parametro TNS

Exemplo de entrada no `tnsnames.ora` do ambiente do banco protegido:

```text
<ra_service> =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = <ra-hostname>)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = <ra_service_name>)
    )
  )
```

Valide:

```bash
tnsping <ra_service>
```

## 3. Registro do banco protegido

Conecte no RMAN usando o catalogo da Recovery Appliance:

```bash
rman target / catalog <vpc_user>@<ra_service>
```

Registre o banco:

```rman
REGISTER DATABASE;
```

Valide o registro:

```rman
LIST DB_UNIQUE_NAME OF DATABASE;
REPORT SCHEMA;
```

## 4. Configuracao RMAN recomendada

Exemplo base:

```rman
CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF <n_dias> DAYS;
CONFIGURE CONTROLFILE AUTOBACKUP ON;
CONFIGURE DEVICE TYPE SBT_TAPE PARALLELISM <grau_parallel> BACKUP TYPE TO BACKUPSET;
CONFIGURE DEFAULT DEVICE TYPE TO SBT_TAPE;
CONFIGURE CHANNEL DEVICE TYPE SBT_TAPE PARMS 'SBT_LIBRARY=<lib_ra>, ENV=(RA_WALLET=<wallet_path>)';
```

> Os parametros `SBT_LIBRARY`, `ENV`, wallet e canais variam conforme versao e padrao da instalacao. Use os valores homologados para a sua Recovery Appliance.

Valide configuracoes:

```rman
SHOW ALL;
```

## 5. Primeiro backup

Execute o backup inicial em janela aprovada:

```rman
RUN {
  ALLOCATE CHANNEL c1 DEVICE TYPE SBT_TAPE;
  ALLOCATE CHANNEL c2 DEVICE TYPE SBT_TAPE;
  BACKUP INCREMENTAL LEVEL 0 DATABASE PLUS ARCHIVELOG;
  RELEASE CHANNEL c1;
  RELEASE CHANNEL c2;
}
```

Valide no RMAN:

```rman
LIST BACKUP SUMMARY;
LIST BACKUP OF DATABASE;
LIST BACKUP OF ARCHIVELOG ALL;
```

## 6. Envio de redo

Quando a arquitetura exigir RPO proximo de zero, habilite o envio de redo para a Recovery Appliance conforme o procedimento oficial da versao instalada.

Checklist:

- [ ] Confirmar suporte do banco e da versao.
- [ ] Confirmar destino de redo configurado.
- [ ] Confirmar autenticacao e wallet.
- [ ] Confirmar ausencia de erros no alert log.
- [ ] Confirmar recebimento de redo pela Recovery Appliance.

Validacoes uteis no banco protegido:

```sql
SELECT dest_id, status, error
FROM v$archive_dest
WHERE status <> 'INACTIVE';
```

```sql
SELECT sequence#, first_time, next_time, applied
FROM v$archived_log
ORDER BY sequence# DESC
FETCH FIRST 20 ROWS ONLY;
```

## 7. Validacao de restore e recover

Antes da liberacao em producao, realize pelo menos um teste controlado de restore/recover em ambiente isolado.

Exemplo de validacao sem restaurar:

```rman
RESTORE DATABASE VALIDATE;
RESTORE ARCHIVELOG ALL VALIDATE;
```

Exemplo de validacao de recover:

```rman
RECOVER DATABASE VALIDATE;
```

Evidencias minimas:

- Saida do `RESTORE ... VALIDATE`.
- Saida do `RECOVER ... VALIDATE`.
- Lista de backups no catalogo.
- Evidencia de recebimento de archivelogs/redo.
- Evidencia de monitoramento ativo.

## 8. Monitoramento

Monitorar:

- Status dos jobs de backup.
- Falhas RMAN.
- Lag de redo/archivelog.
- Consumo de capacidade da Recovery Appliance.
- Throughput de backup e restore.
- Alertas da RA.
- Alert log do banco protegido.

Consultas uteis no banco protegido:

```sql
SELECT start_time, end_time, status, input_type, output_device_type
FROM v$rman_backup_job_details
ORDER BY start_time DESC
FETCH FIRST 20 ROWS ONLY;
```

```sql
SELECT dest_id, status, error
FROM v$archive_dest
WHERE error IS NOT NULL;
```

## 9. Troubleshooting

### Falha de conexao com a Recovery Appliance

Validar:

- `tnsnames.ora`
- DNS ou `/etc/hosts`
- Firewall
- Listener da RA
- Service name
- Wallet e credenciais

Comandos:

```bash
tnsping <ra_service>
sqlplus <vpc_user>@<ra_service>
```

### RMAN nao conecta no catalogo

Validar:

- Usuario VPC correto.
- Senha expirada ou bloqueada.
- Privilegios do usuario VPC.
- Banco registrado no catalogo correto.
- Compatibilidade entre banco e RA.

### Backup lento

Validar:

- Paralelismo RMAN.
- Rede entre Exadata e RA.
- Carga concorrente no banco.
- Janela de backup.
- Metricas de throughput na RA.
- Eventos de espera no banco.

### Erro em archivelog ou redo transport

Validar:

- Destino de archive configurado.
- Erros em `v$archive_dest`.
- Espaco em FRA.
- Alert log.
- Wallet/autenticacao.
- Conectividade intermitente.

## 10. Plano de rollback

Caso seja necessario desfazer a implantacao:

- Parar jobs RMAN agendados para a Recovery Appliance.
- Remover ou desabilitar configuracoes de redo transport relacionadas.
- Restaurar configuracao RMAN anterior, se aplicavel.
- Manter backups ja criados ate decisao formal de expurgo.
- Registrar motivo, horario e responsavel pela reversao.

Exemplo para retornar o default device type:

```rman
CONFIGURE DEFAULT DEVICE TYPE CLEAR;
CONFIGURE CHANNEL DEVICE TYPE SBT_TAPE CLEAR;
SHOW ALL;
```

## 11. Evidencias pos-implantacao

Anexe ou registre:

- Data e hora da implantacao.
- Versao do banco e do Exadata.
- Nome do banco protegido.
- Politica de protecao aplicada.
- Evidencia de conectividade.
- Evidencia do registro no catalogo.
- Evidencia do primeiro backup.
- Evidencia de validacao restore/recover.
- Evidencia de monitoramento.
- Responsaveis pela execucao e validacao.

## 12. Referencias

- Oracle Zero Data Loss Recovery Appliance Documentation
- Oracle Database Backup and Recovery User's Guide
- Oracle Database Backup and Recovery Reference
- Oracle Exadata Database Machine Documentation

Use sempre a documentacao oficial correspondente a versao instalada no ambiente.

## Anexo A - Template de execucao

```text
Cliente:
Ambiente:
Banco:
DB_UNIQUE_NAME:
Recovery Appliance:
Politica:
Data:
Executor:

Inicio:
Fim:
Status:

Comandos executados:

Evidencias:

Pendencias:

Observacoes:
```

## Anexo B - Comandos rapidos

```bash
tnsping <ra_service>
sqlplus <vpc_user>@<ra_service>
rman target / catalog <vpc_user>@<ra_service>
```

```rman
SHOW ALL;
REGISTER DATABASE;
LIST BACKUP SUMMARY;
RESTORE DATABASE VALIDATE;
RECOVER DATABASE VALIDATE;
```

```sql
SELECT dest_id, status, error
FROM v$archive_dest
WHERE status <> 'INACTIVE';

SELECT start_time, end_time, status, input_type, output_device_type
FROM v$rman_backup_job_details
ORDER BY start_time DESC
FETCH FIRST 20 ROWS ONLY;
```
