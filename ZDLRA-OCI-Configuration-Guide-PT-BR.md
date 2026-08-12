# Oracle Database Autonomous Recovery Service na OCI

## Guia de Implantação e Configuração

> Este guia de implantação é baseado na documentação Oracle e destina-se ao uso pelo cliente. Não é um documento emitido pela Oracle e deve ser validado com a documentação Oracle vigente e com o processo de gestão de mudanças do cliente.

## Objetivo

Este documento apresenta um procedimento técnico para implantar e configurar o **Oracle Database Autonomous Recovery Service** (Recovery Service, anteriormente conhecido como Zero Data Loss Autonomous Recovery Service) na Oracle Cloud Infrastructure (OCI). O objetivo é proteger bancos Oracle com backups automatizados, recuperação point-in-time e recursos avançados de resiliência.

O guia cobre:

- validação de pré-requisitos de banco, rede e capacidade;
- configuração de IAM, subnet e regras de segurança;
- criação de uma política de proteção e habilitação dos backups;
- validação do banco protegido e testes de recuperação sem impacto;
- monitoramento, troubleshooting, rollback e evidências de entrega.

## Escopo e serviços envolvidos

| Item | Descrição |
|---|---|
| Serviço principal | Oracle Database Autonomous Recovery Service |
| Plataforma | Oracle Cloud Infrastructure (OCI) |
| Bancos de origem | Oracle Base Database Service, Exadata Database Service, Exascale (quando suportado), Oracle Database@Azure/Google/AWS e ambientes on-premises via Cloud Protect |
| Operação | OCI Console, OCI CLI, RMAN e Oracle Cloud Agent |
| Segurança | IAM, Network Security Groups (NSG), Security Lists, subnets privadas e OCI Logging |

## 1. Pré-requisitos

### 1.1 Matriz de validação

| Categoria | Requisito | Validação esperada |
|---|---|---|
| Banco de dados | Versão e modelo de implantação suportados | Confirmar na matriz de compatibilidade Oracle |
| Compatibilidade | `COMPATIBLE` igual ou superior a 19.0.0 | Consultar `v$database` |
| Rede | Conectividade TCP nas portas 2484 e 8005 | Teste a partir do host de banco |
| Endereçamento | IPv4 privado na subnet do Recovery Service | Conferir VCN/subnet e resolução DNS |
| Segurança | NSG ou Security List permitindo o tráfego necessário | Regras documentadas e aprovadas |
| IAM | Grupo administrativo com políticas do Recovery Service | Políticas aplicadas no tenancy/compartments corretos |
| Capacidade | Limites de serviço suficientes | Conferir Service Limits/Quotas |
| Backup atual | Jobs ou scripts manuais identificados | Planejar coexistência, migração ou desativação controlada |

### 1.2 Referências Oracle

- [Documentação do Autonomous Recovery Service](https://docs.oracle.com/en-us/iaas/recovery-service/index.html)
- [Tutorial de início rápido](https://docs.oracle.com/en/learn/oci-recovery-service/)
- [Requisitos de rede do Recovery Service](https://docs.oracle.com/en-us/iaas/recovery-service/doc/prepare-network.html)

## 2. Arquitetura de referência

```text
+----------------------+       TCP 2484 / 8005       +---------------------------+
| Banco Oracle         | --------------------------> | Autonomous Recovery       |
| DB System / Exadata  |                             | Service (subnet privada)  |
| Oracle Cloud Agent   |                             |                           |
+----------+-----------+                             +-------------+-------------+
           |                                                       |
           | IAM / Policy de proteção                              | Object Storage gerenciado
           v                                                       v
     OCI Console / CLI                                      Backups e metadados
```

O caminho de dados deve permanecer privado. Planeje DNS, rotas e regras de segurança antes de habilitar a proteção.

## 3. Checklist de implantação

| Etapa | Responsável sugerido | Evidência |
|---|---|---|
| Validar versão e `COMPATIBLE` | DBA | Saída SQL arquivada |
| Confirmar limites e quotas | Cloud Administrator | Captura do OCI Console |
| Criar grupo e políticas IAM | IAM Administrator | Policies e OCIDs |
| Criar ou selecionar subnet do Recovery Service | Network Administrator | OCID da subnet e CIDR |
| Abrir e testar as portas 2484 e 8005 | Network Administrator + DBA | Regras, teste TCP e Flow Logs |
| Criar a política de proteção | Recovery Administrator | OCID e parâmetros da policy |
| Associar o banco à política | DBA | Banco com status Protected |
| Executar backup e validação de restore | DBA | Logs RMAN e evidências do Console |
| Configurar alarmes | Operations | Alarmes, destinatários e runbook |

## 4. Validar a versão do banco

Conecte-se como usuário com privilégio adequado e execute:

```sql
SELECT name,
       db_unique_name,
       log_mode,
       force_logging,
       platform_name,
       compatible
FROM   v$database;
```

Critérios mínimos:

- o banco deve estar em versão/modelo suportados;
- `COMPATIBLE` deve ser igual ou superior a `19.0.0`;
- valide o modo ARCHIVELOG e FORCE LOGGING de acordo com o padrão de proteção e recuperação definido para o ambiente.

## 5. Verificar limites de serviço

No OCI Console, acesse **Governance & Administration > Limits, Quotas and Usage** e pesquise por *Recovery Service*. Confirme previamente a capacidade necessária para políticas, bancos protegidos e componentes de rede associados.

Registre solicitações de aumento de limite antes da janela de implantação, quando aplicável.

## 6. Configurar IAM

Crie ou utilize um grupo dedicado, por exemplo `<grupo-recovery-admin>`. Ajuste tenancy e compartments aos padrões de segregação do cliente.

```text
Allow group <grupo-recovery-admin> to manage recovery-service-family in tenancy

Allow group <grupo-recovery-admin> to manage recovery-service-policy in compartment <compartment-recovery>

Allow group <grupo-recovery-admin> to manage recovery-service-subnet in compartment <compartment-network>
```

Para Oracle Database@Azure, Oracle Database@Google Cloud ou Oracle Database@AWS, utilize as políticas específicas do modelo multicloud publicadas pela Oracle. Aplique sempre o princípio de menor privilégio e valide as policies com um usuário membro do grupo.

## 7. Configurar a rede

### 7.1 Inventário de rede obrigatório

Antes de criar a política de proteção, registre:

| Campo | Exemplo / orientação |
|---|---|
| VCN do banco | `vcn-producao` |
| Subnet/VLAN dos DB nodes | CIDR de origem dos bancos protegidos |
| NSG de origem | NSG associado às VNICs do banco |
| Subnet do Recovery Service | Subnet privada IPv4 dedicada ou aprovada |
| NSG de destino | NSG associado à subnet do serviço, quando aplicável |
| Rotas e DNS | Caminho privado e resolução do endpoint |
| Responsáveis | Time de rede, segurança e DBA |

Use endereços IPv4 privados. Não exponha essas portas à internet e não use `0.0.0.0/0` como origem ou destino.

### 7.2 Abertura e validação das portas 2484 e 8005

Esta etapa deve ser concluída **antes** de habilitar o banco no Recovery Service.

**Responsabilidade:** o time OCI de rede/segurança implementa as regras; o DBA valida a conectividade a partir de cada host de banco.

#### A. Criar/identificar os grupos de segurança

1. No OCI Console, abra **Networking > Virtual Cloud Networks > [VCN] > Network Security Groups**.
2. Identifique o NSG aplicado às VNICs dos DB nodes. Caso não exista, crie um NSG dedicado, por exemplo `nsg-db-to-recovery-service`.
3. Identifique o NSG da subnet do Recovery Service ou crie um NSG para esse destino, conforme o desenho de rede.
4. Confirme se as regras serão **stateful**. Regras stateful são recomendadas para este fluxo e permitem o retorno da conexão sem regra adicional de portas efêmeras.

#### B. Configurar regras de saída do banco

No NSG da origem (DB nodes), crie regras de **egress TCP**:

| Direção | Protocolo | Porta destino | Destino permitido | Descrição |
|---|---:|---:|---|---|
| Egress | TCP | 2484 | CIDR privado ou NSG do Recovery Service | Canal seguro para o Recovery Service |
| Egress | TCP | 8005 | CIDR privado ou NSG do Recovery Service | Comunicação de serviço/controle |

Prefira selecionar o NSG de destino na regra quando ambos os lados estiverem na mesma VCN. Quando o desenho exigir CIDR, restrinja a regra ao menor CIDR possível da subnet do serviço.

#### C. Configurar regras de entrada quando o destino for gerenciado pelo cliente

Se a subnet/NSG de destino for gerenciada pelo cliente, crie também as regras **ingress TCP** abaixo no NSG do destino:

| Direção | Protocolo | Porta destino | Origem permitida | Descrição |
|---|---:|---:|---|---|
| Ingress | TCP | 2484 | CIDR ou NSG dos DB nodes | Recebe conexões dos bancos protegidos |
| Ingress | TCP | 8005 | CIDR ou NSG dos DB nodes | Recebe comunicação de serviço/controle |

Se a Oracle gerencia totalmente o componente de destino, não tente alterar recursos gerenciados. Garanta as regras de saída do banco e siga os requisitos da documentação Oracle para o serviço.

#### D. Alternativa com Security Lists

Se o ambiente usar Security Lists em vez de NSGs, replique o mesmo fluxo:

- regra de saída TCP do CIDR da subnet de banco para o CIDR da subnet do Recovery Service nas portas 2484 e 8005;
- regra de entrada correspondente no destino, quando esse destino for administrado pelo cliente;
- confirme que não há firewall de sistema operacional, appliance virtual, DRG ou rota bloqueando o caminho.

#### E. Testar a conectividade

Depois da aprovação e propagação das regras, execute em **cada DB node**:

```bash
nc -vz <endpoint-privado-recovery-service> 2484
nc -vz <endpoint-privado-recovery-service> 8005
```

Resultados esperados: ambas as conexões devem retornar `succeeded` ou `open`. Se `nc` não estiver disponível, use uma ferramenta equivalente aprovada pelo sistema operacional.

Em caso de falha, valide nesta ordem:

1. endpoint/DNS e endereço IPv4 privado corretos;
2. rota entre origem e destino;
3. regras de egress do banco;
4. regras de ingress do destino, quando aplicáveis;
5. firewalls locais, appliances e regras no DRG;
6. VCN Flow Logs para identificar `ACCEPT` ou `REJECT`.

#### F. Evidência e rollback

Anexe ao change record:

- OCIDs dos NSGs ou Security Lists alterados;
- captura ou export das regras aprovadas;
- saída dos testes TCP por DB node;
- consulta de VCN Flow Logs, se habilitados.

Para rollback, remova exclusivamente as regras criadas para 2484 e 8005 após desassociar o banco do Recovery Service ou após a aprovação do change manager. Não remova regras compartilhadas sem análise de impacto.

## 8. Criar ou selecionar uma política de proteção

No OCI Console, acesse **Oracle Database > Autonomous Recovery Service > Protection Policies**.

1. Crie uma política ou selecione uma policy existente aprovada.
2. Defina a janela de recuperação e a retenção conforme RPO/RTO e requisitos regulatórios.
3. Associe a subnet privada e as opções de rede validadas anteriormente.
4. Registre o OCID da policy, a retenção, a janela de recuperação e o owner operacional.

Evite alterar políticas compartilhadas sem avaliar o impacto em todos os bancos associados.

## 9. Habilitar backups e registrar o banco protegido

Selecione o banco no OCI Console e habilite a proteção pela policy definida. Confirme os pré-requisitos indicados pelo assistente e aguarde o status de registro/proteção.

O procedimento pode variar por modelo de banco (Base DB, Exadata ou multicloud). A execução deve seguir o fluxo indicado no Console e na documentação Oracle correspondente ao seu deployment.

## 10. Validar o banco protegido

No Console, confirme:

- banco visível na lista de protected databases;
- policy correta associada;
- primeiro backup iniciado ou concluído;
- ausência de erros de Oracle Cloud Agent, rede ou autorização;
- métricas de backup e consumo atualizando conforme esperado.

Registre data/hora, OCID do banco, OCID da policy e evidências do primeiro job.

## 11. Validar backups e recuperação com RMAN

Em ambiente controlado e sem alterar dados, execute validações RMAN:

```rman
RESTORE DATABASE VALIDATE;
RESTORE ARCHIVELOG ALL VALIDATE;
RECOVER DATABASE VALIDATE;
```

Esses comandos verificam a recuperabilidade sem restaurar arquivos de dados de forma definitiva. Planeje testes periódicos completos de recuperação de acordo com o RTO acordado.

## 12. Monitoramento e alarmes

Configure monitoramento no OCI Console para capacidade, falhas de backup, erros de comunicação e eventos operacionais relevantes.

Recomendações:

- criar alarmes para falhas persistentes de backup e ausência de backups dentro da janela acordada;
- direcionar notificações para a equipe de operação/on-call;
- manter um runbook com owner, severidade, diagnóstico e escalonamento;
- revisar métricas e tendências de consumo periodicamente.

## 13. Troubleshooting inicial

| Sintoma | Verificações prioritárias |
|---|---|
| Backup não inicia | Status do Cloud Agent, associação à policy, IAM e limites |
| Erro de conectividade | DNS, rota, IPv4 privado, portas 2484/8005, NSG/Security List e Flow Logs |
| Banco não aparece como protegido | Compatibilidade, permissões e conclusão do workflow no Console |
| Backup lento | Latência de rede, volume de archivelogs, janela de backup e concorrência |
| Falha no restore validate | Disponibilidade dos backups, logs RMAN e consistência de archivelogs |

Colete o job ID, horário, mensagem de erro, logs RMAN, logs do agente e evidências de rede antes de abrir chamado com a Oracle.

## 14. Rollback e desativação controlada

1. Confirme o impacto de remover a proteção e valide a existência de uma estratégia de backup alternativa aprovada.
2. Desassocie o banco da policy ou desabilite a proteção conforme o fluxo do OCI Console.
3. Preserve logs, evidências de backup e o change record conforme a retenção operacional.
4. Remova apenas regras de rede exclusivas da integração, se não forem mais necessárias.
5. Revogue policies IAM dedicadas somente quando não houver outro recurso dependente.

## 15. Evidências de entrega

| Evidência | Conteúdo mínimo |
|---|---|
| Inventário de bancos | Nome, `DB_UNIQUE_NAME`, ambiente, região e compartment |
| Política de proteção | Nome, OCID, retenção, janela de recuperação e owner |
| Configuração IAM | Grupo, statements de policy e compartments |
| Configuração de rede | VCN, subnets, NSGs/Security Lists, CIDRs e portas 2484/8005 |
| Teste de conectividade | Saída TCP por DB node e, quando disponível, Flow Logs |
| Primeiro backup | Job ID, horário, status e log associado |
| Teste de recuperação | Saída do `RESTORE ... VALIDATE` e responsável pela validação |
| Monitoramento | Alarmes, canais de notificação e runbook |

## Apêndice A – Modelo de solicitação de mudança de rede

```text
Título: Liberação de conectividade para Oracle Autonomous Recovery Service

Origem: <CIDR ou NSG dos DB nodes>
Destino: <CIDR ou NSG da subnet privada do Recovery Service>
Protocolo: TCP
Portas de destino: 2484 e 8005
Direção: Egress na origem; Ingress no destino quando administrado pelo cliente
Tipo: Stateful
Justificativa: Proteção de banco Oracle por Autonomous Recovery Service
Janela: <data/hora>
Rollback: Remover exclusivamente as regras criadas, após aprovação
Evidências: OCIDs, export/captura das regras, testes TCP e Flow Logs
```

## Apêndice B – Comandos úteis

```sql
SELECT name, db_unique_name, log_mode, force_logging, compatible
FROM   v$database;
```

```bash
nc -vz <endpoint-privado-recovery-service> 2484
nc -vz <endpoint-privado-recovery-service> 8005
```

```rman
RESTORE DATABASE VALIDATE;
RESTORE ARCHIVELOG ALL VALIDATE;
RECOVER DATABASE VALIDATE;
```

---

Este material é independente e educacional; não substitui a documentação oficial Oracle. Oracle, Java e MySQL são marcas registradas da Oracle Corporation e/ou de suas afiliadas.
