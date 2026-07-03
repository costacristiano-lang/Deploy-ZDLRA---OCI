# Deploy e Configuracao do Oracle Database Zero Data Loss Autonomous Recovery Service na OCI

[English](./README-oci-zero-data-loss-recovery-service.md) | **Português (Brasil)**

Este README descreve um procedimento base para implantar e configurar o **Oracle Database Zero Data Loss Autonomous Recovery Service** na Oracle Cloud Infrastructure (OCI), protegendo bancos Oracle em OCI, Exadata Database Service ou ambientes multicloud suportados.

> Na OCI, o servico gerenciado e chamado de **Database Autonomous Recovery Service** ou **Oracle Database Zero Data Loss Autonomous Recovery Service**. O termo **ZDLRA** normalmente se refere ao Oracle Zero Data Loss Recovery Appliance, que e o appliance/engineered system. Para OCI, use este guia como referencia para o servico gerenciado.

## Objetivo

- Configurar backup gerenciado com Recovery Service na OCI.
- Habilitar protecao com baixa ou zero perda de dados quando o recurso de real-time data protection estiver disponivel.
- Criar ou selecionar uma protection policy.
- Configurar rede, IAM, subnet e backup automatico.
- Validar backup, restore e operacao.

## Servicos envolvidos

- Oracle Cloud Infrastructure IAM
- Oracle Cloud Infrastructure Networking
- Oracle Database Autonomous Recovery Service
- Oracle Base Database Service ou Exadata Database Service
- Oracle Cloud Guard/Monitoring, quando aplicavel

## Cenario suportado

Use este guia para:

- Oracle Base Database Service na OCI.
- Oracle Exadata Database Service on Dedicated Infrastructure.
- Oracle Exadata Database Service on Exascale Infrastructure, se suportado na regiao e versao.
- Oracle Database@Azure, Oracle Database@Google Cloud ou Oracle Database@AWS, quando a assinatura multicloud estiver habilitada.

Para banco on-premises, o fluxo e diferente e usa **Oracle Database Zero Data Loss Cloud Protect**.

## Pre-requisitos

### Informacoes do ambiente

| Item | Valor |
|---|---|
| Tenancy | `<tenancy_name>` |
| Regiao OCI | `<region>` |
| Compartment do banco | `<db_compartment>` |
| Compartment do Recovery Service | `<recovery_compartment>` |
| VCN | `<vcn_name>` |
| Subnet do banco | `<db_subnet>` |
| Backup subnet / Recovery Service subnet | `<recovery_service_subnet>` |
| DB System / VM Cluster | `<db_system_or_vm_cluster>` |
| Banco/CDB | `<db_name>` |
| DB unique name | `<db_unique_name>` |
| Versao Oracle Database | `<db_version>` |
| Protection policy | `<protection_policy>` |
| Retencao | `<retention_days>` |
| RPO esperado | `<rpo>` |

### Requisitos obrigatorios

- Banco Oracle suportado pelo Recovery Service.
- `COMPATIBLE` do banco em `19.0.0` ou superior.
- Portas liberadas para o Recovery Service:
  - `2484`: conexao SQL*Net com o RMAN catalog usado pelo Recovery Service.
  - `8005`: trafego de backup entre o banco e o Recovery Service.
- Subnet privada IPv4 para operacoes do Recovery Service.
- Regras de seguranca via Security List ou Network Security Group.
- Limites de servico suficientes para quantidade de protected databases e consumo de backup.
- Backup manual ou scripts paralelos desabilitados antes de habilitar backup automatico para Recovery Service.

Referencias oficiais:

- [Overview of Oracle Database Autonomous Recovery Service](https://docs.oracle.com/en-us/iaas/recovery-service/doc/overview-recovery-service.html)
- [Onboarding Oracle Database to Recovery Service](https://docs.oracle.com/en-us/iaas/recovery-service/doc/getting-started-recovery-service.html)
- [Database Autonomous Recovery Service documentation](https://docs.oracle.com/en-us/iaas/recovery-service/index.html)

## Arquitetura resumida

```text
+--------------------------+        Portas 2484 / 8005        +----------------------------------+
| Banco Oracle na OCI      | --------------------------------> | OCI Autonomous Recovery Service  |
| Base DB / Exadata DB     |                                   | Backup catalog / protected DB    |
+--------------------------+                                   +----------------------------------+
          |
          | VCN / subnet privada / security rules
          v
+--------------------------+
| OCI Networking / IAM     |
+--------------------------+
```

## Checklist de implantacao

- [ ] Confirmar versao e compatibilidade do banco.
- [ ] Confirmar se o banco esta em OCI, Exadata DB Service ou multicloud suportado.
- [ ] Revisar limites de Recovery Service na regiao.
- [ ] Validar IAM policies.
- [ ] Validar VCN, subnet e security rules.
- [ ] Confirmar portas `2484` e `8005`.
- [ ] Criar ou selecionar protection policy.
- [ ] Habilitar automatic backup usando Recovery Service.
- [ ] Validar protected database criado.
- [ ] Validar primeiro backup.
- [ ] Validar restore/recover.
- [ ] Configurar monitoramento e alertas.
- [ ] Registrar evidencias.

## 1. Validar versao do banco

No banco:

```sql
SELECT name, db_unique_name, database_role, open_mode
FROM v$database;

SHOW PARAMETER compatible
```

Para real-time data protection, valide a release update minima suportada pela documentacao da Oracle para sua versao.

## 2. Validar limites do servico

Na Console OCI:

```text
Governance & Administration
  Tenancy Management
    Limits, Quotas and Usage
      Service: Autonomous Recovery Service
```

Validar:

- Protected database count.
- Space used for recovery window.
- Limites por regiao.
- Quotas por compartment, se existirem.

## 3. Configurar IAM

Para bancos Oracle na OCI, o Database Service normalmente ja possui permissoes para acessar o Recovery Service. Ainda assim, crie permissoes administrativas para o grupo que vai operar o servico.

Exemplo:

```text
Allow group <recovery_admin_group> to manage recovery-service-family in tenancy
```

Permissao mais restrita para protection policies:

```text
Allow group <recovery_admin_group> to manage recovery-service-policy in compartment <recovery_compartment>
```

Permissao para subnets do Recovery Service:

```text
Allow group <recovery_admin_group> to manage recovery-service-subnet in compartment <network_compartment>
```

Para multicloud, use os templates oficiais do Policy Builder para Oracle Database@Azure, Oracle Database@Google Cloud ou Oracle Database@AWS.

## 4. Configurar rede

O Recovery Service usa uma subnet privada IPv4 na mesma VCN do banco.

Para Oracle Database na OCI:

- Exadata Database Service on Dedicated Infrastructure: a backup subnet pode ser registrada automaticamente como Recovery Service subnet.
- Base Database Service: a database subnet pode ser registrada automaticamente como Recovery Service subnet.
- Opcionalmente, crie uma subnet dedicada para Recovery Service.

Regras minimas:

| Origem | Destino | Porta | Protocolo | Uso |
|---|---|---:|---|---|
| Subnet do banco | Recovery Service subnet | 2484 | TCP | RMAN catalog / SQL*Net |
| Subnet do banco | Recovery Service subnet | 8005 | TCP | Trafego de backup |

Se usar NSG, associe o NSG ao recurso/subnet correto conforme o tipo de banco.

## 5. Criar ou selecionar protection policy

Na Console OCI:

```text
Oracle Database
  Database Autonomous Recovery
    Protection Policies
      Create Protection Policy
```

Defina:

- Nome da politica.
- Compartment.
- Retencao.
- Opcao de retention lock, se exigida por compliance.
- Localizacao de backup, quando aplicavel a multicloud.

Observacao: politicas customizadas possuem limites de retencao definidos pelo servico. Valide o intervalo permitido na Console/documentacao da versao vigente.

## 6. Habilitar backup automatico para Recovery Service

Na Console OCI:

```text
Oracle Database
  Databases
    <database>
      Backups
        Configure automatic backups
```

Selecionar:

- Backup destination: `Autonomous Recovery Service`.
- Protection policy: `<protection_policy>`.
- Retention.
- Real-time data protection, se disponivel e requerido.

Ao habilitar o backup automatico, o Recovery Service pode registrar automaticamente a Recovery Service subnet, dependendo do tipo de banco.

## 7. Validar protected database

Na Console OCI:

```text
Oracle Database
  Database Autonomous Recovery
    Protected Databases
```

Validar:

- Banco aparece como protected database.
- Estado esta `Active` ou equivalente.
- Protection policy correta.
- Compartment correto.
- Recovery window esperada.
- Ultimo backup concluido com sucesso.
- Real-time protection ativa, quando aplicavel.

## 8. Validar backup pelo banco

No banco:

```sql
SELECT start_time, end_time, status, input_type, output_device_type
FROM v$rman_backup_job_details
ORDER BY start_time DESC
FETCH FIRST 20 ROWS ONLY;
```

Validar archivelogs:

```sql
SELECT sequence#, first_time, next_time, applied
FROM v$archived_log
ORDER BY sequence# DESC
FETCH FIRST 20 ROWS ONLY;
```

Validar destinos de archive:

```sql
SELECT dest_id, status, error
FROM v$archive_dest
WHERE status <> 'INACTIVE';
```

## 9. Testar restore/recover

Execute primeiro validacoes nao destrutivas:

```rman
RESTORE DATABASE VALIDATE;
RESTORE ARCHIVELOG ALL VALIDATE;
RECOVER DATABASE VALIDATE;
```

Para teste real, use ambiente isolado, clone ou procedimento formal de restore homologado.

Evidencias minimas:

- Protected database ativo na OCI.
- Primeiro backup concluido.
- Saida do RMAN validate.
- Restore/recover testado em ambiente isolado, quando requerido.
- Alertas configurados.

## 10. Monitoramento

Monitorar:

- Estado do protected database.
- Ultimo backup.
- Recovery window.
- Erros de backup.
- Consumo de armazenamento.
- Limites de servico.
- Eventos OCI.
- Alarmes no OCI Monitoring.

Metricas e eventos devem ser vinculados a um canal de notificacao:

```text
Observability & Management
  Monitoring
    Alarm Definitions
      Create Alarm
```

Sugestoes de alarmes:

- Falha de backup.
- Protected database nao ativo.
- Consumo proximo do limite.
- Ausencia de backup recente.
- Erros de redo/real-time protection.

## 11. Troubleshooting

### Backup nao inicia

Validar:

- Backup automatico esta habilitado.
- Destination esta configurado como Autonomous Recovery Service.
- Protection policy existe e esta no compartment correto.
- IAM policies permitem gerenciamento.
- Limites de servico nao foram excedidos.

### Erro de rede

Validar:

- Portas `2484` e `8005`.
- Security List ou NSG.
- Subnet privada IPv4.
- Routing dentro da VCN.
- Se a subnet foi registrada como Recovery Service subnet.

### Protected database nao aparece

Validar:

- Compartment selecionado na Console.
- Regiao correta.
- Backup automatico habilitado.
- Permissoes IAM do usuario.
- Eventos de work request na OCI.

### Real-time data protection indisponivel

Validar:

- Release Update minima do banco.
- Tipo de banco suportado.
- Compatibilidade do banco.
- Protection policy selecionada.
- Restricoes da regiao.

### Backup manual paralelo

Antes de habilitar o Recovery Service, desabilite scripts manuais ou jobs que enviem backup operacional para outro destino. Backups operacionais em dois destinos podem causar cenarios de perda de dados ou inconsistencias operacionais.

## 12. Plano de rollback

Caso seja necessario desfazer:

- Desabilitar automatic backup para Recovery Service.
- Reativar politica anterior de backup somente apos aprovacao.
- Manter backups existentes ate decisao formal de retencao/expurgo.
- Remover alarmes ou policies apenas se nao forem mais usadas.
- Registrar work requests, horario e responsavel.

## 13. Evidencias para entrega

| Evidencia | Status |
|---|---|
| Compatibilidade do banco validada | `[ ]` |
| Limites de servico revisados | `[ ]` |
| IAM configurado | `[ ]` |
| Rede liberada nas portas 2484/8005 | `[ ]` |
| Protection policy criada/selecionada | `[ ]` |
| Backup automatico habilitado | `[ ]` |
| Protected database ativo | `[ ]` |
| Primeiro backup concluido | `[ ]` |
| Restore/recover validate executado | `[ ]` |
| Monitoramento configurado | `[ ]` |

## Anexo A - Template de mudanca

```text
Mudanca:
Cliente:
Tenancy:
Regiao:
Compartment:
Banco:
DB unique name:
Tipo de banco:
Protection policy:
Retencao:
Real-time protection:

Inicio:
Fim:
Executor:
Validador:

Resultado:
Pendencias:
Rollback necessario:
Observacoes:
```

## Anexo B - Comandos SQL uteis

```sql
SELECT name, db_unique_name, database_role, open_mode
FROM v$database;

SHOW PARAMETER compatible

SELECT start_time, end_time, status, input_type, output_device_type
FROM v$rman_backup_job_details
ORDER BY start_time DESC
FETCH FIRST 20 ROWS ONLY;

SELECT dest_id, status, error
FROM v$archive_dest
WHERE status <> 'INACTIVE';
```
