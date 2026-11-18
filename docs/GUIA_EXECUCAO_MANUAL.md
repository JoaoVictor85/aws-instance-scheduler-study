# Guia de Execução Manual — AWS Instance Scheduler on AWS

Este guia descreve como recriar manualmente todos os recursos da solução **Instance Scheduler on AWS v3.2.15** usando o console AWS ou AWS CLI, seguindo o template CloudFormation como referência técnica.

> **Referência:** `instance-scheduler-on-aws.template` (8067 linhas, JSON)
> **Namespace usado neste estudo:** `default` (substitua onde indicado)
> **Região usada neste estudo:** `us-east-1` (substitua onde indicado)

---

## Pré-requisitos

- [ ] Conta AWS ativa (Sandbox ou estudo)
- [ ] Usuário/Role com permissões de administrador
- [ ] Acesso ao console AWS

---

## Usando o AWS CloudShell

Todos os comandos AWS CLI deste guia devem ser executados no **AWS CloudShell** — acessível pelo ícone de terminal no canto superior direito do console AWS.

**Vantagens do CloudShell para estudo:**
- Autenticação automática com as credenciais da sessão atual — sem chaves de acesso expostas localmente
- AWS CLI, Python e jq já pré-instalados
- A região ativa é a mesma do console no momento de abertura

**Regra importante para `aws lambda invoke`:**
O AWS CLI v2 (usado no CloudShell) exige que o `--payload` seja passado via arquivo, não inline:

```bash
# Salve o payload em um arquivo antes de invocar
echo '{"RequestType": "Create", "ResourceProperties": {}}' > payload.json

aws lambda invoke \
  --function-name <NOME_DA_FUNCAO> \
  --payload fileb://payload.json \
  output.json

# Veja o resultado
cat output.json
```

**Verificar a região ativa no CloudShell:**
```bash
aws configure get region
# Se não estiver configurada, defina:
export AWS_DEFAULT_REGION=us-east-1
```

**Obter o Account ID atual:**
```bash
aws sts get-caller-identity --query Account --output text
```

---

## Ordem de Criação

Os recursos precisam ser criados na ordem abaixo por causa das **dependências**: recursos mais simples (KMS, DynamoDB) são referenciados por recursos mais complexos (Lambda, EventBridge).

```
1. KMS
2. DynamoDB
3. CloudWatch Logs
4. SNS
5. SQS
6. EventBridge (Event Buses)
7. IAM Roles e Policies
8. Lambda Functions
9. Lambda Permissions e EventSourceMapping
10. EventBridge Rules
11. CloudWatch Logs Subscription Filters
12. Step Functions
13. Custom Resources (lógica de inicialização)
14. CloudWatch Dashboard
15. CloudWatch Logs Insights Queries
```

---

## Etapa 1 — KMS (AWS Key Management Service)

### Por que existe?
Todos os dados em repouso da solução são criptografados: tabelas DynamoDB, tópico SNS, filas SQS e log groups do CloudWatch. Uma única CMK (Customer Managed Key) centraliza essa criptografia.

### Recursos a criar

#### 1.1 — KMS Key (`InstanceSchedulerEncryptionKey`)

| Campo | Valor |
|---|---|
| Tipo | Symmetric, Encrypt and decrypt |
| Descrição | `Instance Scheduler CMK - <nome-da-stack>` |
| Rotação automática | Habilitada |
| Key policy | Permite root da conta (`arn:aws:iam::<AccountId>:root`) todas as ações; permite `logs.amazonaws.com` as ações `kms:Decrypt`, `kms:Encrypt`, `kms:ReEncrypt*`, `kms:GenerateDataKey*` |

**Key Policy (JSON):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::<SEU_ACCOUNT_ID>:root" },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Principal": { "Service": "logs.amazonaws.com" },
      "Action": ["kms:Decrypt","kms:Encrypt","kms:ReEncrypt*","kms:GenerateDataKey*"],
      "Resource": "*"
    }
  ]
}
```

> **Nota de retenção:** se quiser manter a chave ao deletar a stack, não delete a chave. KMS tem período de espera mínimo de 7 dias para exclusão.

#### 1.2 — KMS Alias (`InstanceSchedulerEncryptionKeyAlias`)

| Campo | Valor |
|---|---|
| Alias name | `alias/AwsSolutions/InstanceScheduler/default/instance-scheduler-on-aws` |
| Target key | ARN da key criada no passo 1.1 |

**Evidência a capturar:** print da KMS Key criada com o alias visível → `evidencias/01-kms/`

---

## Etapa 2 — DynamoDB (4 tabelas)

### Por que existem?
- **ConfigTable**: armazena os schedules e períodos configurados pelo usuário
- **StateTable**: rastreia o estado atual (running/stopped) de cada instância por serviço e região
- **ResourceRegistry**: registra automaticamente todas as instâncias que possuem a tag de agendamento
- **MaintenanceWindowTable**: armazena dados de Maintenance Windows do SSM

Todas as tabelas compartilham as mesmas configurações de segurança:
- Billing mode: **PAY_PER_REQUEST** (sem capacidade provisionada)
- Criptografia: **SSE com KMS** (usar ARN da key do passo 1.1)
- **Point-in-Time Recovery (PITR):** habilitado

### Recursos a criar

#### 2.1 — ConfigTable

| Campo | Valor |
|---|---|
| Nome | `ConfigTable` (ou deixe o console gerar) |
| Partition Key | `type` (String) |
| Sort Key | `name` (String) |
| DynamoDB Streams | Habilitado — View type: **KEYS_ONLY** |
| SSE | KMS — ARN da key do passo 1.1 |
| PITR | Habilitado |

#### 2.2 — StateTable

| Partition Key | Sort Key | Streams | SSE |
|---|---|---|---|
| `service` (String) | `account-region` (String) | Desabilitado | KMS |

#### 2.3 — ResourceRegistry

| Partition Key | Sort Key | Streams | SSE |
|---|---|---|---|
| `account` (String) | `sk` (String) | Desabilitado | KMS |

#### 2.4 — MaintenanceWindowTable

| Partition Key | Sort Key | Streams | SSE |
|---|---|---|---|
| `account-region` (String) | `name-id` (String) | Desabilitado | KMS |

**Evidência a capturar:** print das 4 tabelas criadas no console DynamoDB → `evidencias/02-dynamodb/`

---

## Etapa 3 — CloudWatch Log Groups (2 grupos)

### Por que existem?
Centralizam os logs de todas as Lambda functions da solução em dois grupos temáticos — um administrativo (registro de spokes, setup) e um de scheduling (decisões de start/stop).

#### 3.1 — AdministrationLogs

| Campo | Valor |
|---|---|
| Nome | `<nome-da-stack>-default-administrative-logs` |
| Retenção | 30 dias (ajustável) |
| KMS Key ARN | ARN da key do passo 1.1 |

#### 3.2 — SchedulingLogs

| Campo | Valor |
|---|---|
| Nome | `<nome-da-stack>-default-scheduling-logs` |
| Retenção | 30 dias (ajustável) |
| KMS Key ARN | ARN da key do passo 1.1 |

**Evidência a capturar:** print dos 2 log groups → `evidencias/03-cloudwatch-logs/`

---

## Etapa 4 — SNS (Simple Notification Service)

### Por que existe?
Recebe notificações de erros e warnings das Lambda functions via CloudWatch Logs Subscription Filters. Você pode adicionar uma assinatura de e-mail para ser alertado de problemas.

#### 4.1 — InstanceSchedulerSnsTopic

| Campo | Valor |
|---|---|
| Tipo | Standard |
| Criptografia | Habilitada — KMS Key ARN do passo 1.1 |

#### 4.2 — Topic Policy

Adicione a policy abaixo ao tópico para **bloquear publicações sem SSL**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "sns:Publish",
      "Resource": "<ARN_DO_TOPICO>",
      "Condition": {
        "Bool": { "aws:SecureTransport": "false" }
      }
    }
  ]
}
```

**Evidência a capturar:** print do tópico SNS com criptografia habilitada → `evidencias/04-sns/`

---

## Etapa 5 — SQS (Simple Queue Service)

### Por que existem?
A fila `Ec2ResizeRequestQueue` recebe pedidos de resize de instâncias EC2 quando uma instância falha ao iniciar por falta de capacidade. As DLQs garantem que mensagens com falha não se percam.

#### 5.1 — ResourceRegistrationDLQ

| Campo | Valor |
|---|---|
| Tipo | Standard |
| Criptografia | SQS managed SSE |
| Finalidade | DLQ para falhas de registro de recursos via EventBridge |

#### 5.2 — Ec2ResizeDLQueue

| Campo | Valor |
|---|---|
| Tipo | Standard |
| Criptografia | KMS — ARN do passo 1.1 |
| Finalidade | DLQ para falhas na fila de resize |

#### 5.3 — Ec2ResizeRequestQueue (fila principal)

| Campo | Valor |
|---|---|
| Tipo | Standard |
| Criptografia | KMS — ARN do passo 1.1 |
| Visibility Timeout | 180 segundos |
| Dead-letter queue | ARN da `Ec2ResizeDLQueue` |
| Max receive count | 1 |

Adicione uma **Queue Policy** à `Ec2ResizeDLQueue` e à `Ec2ResizeRequestQueue` negando acesso sem SSL:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "sqs:*",
      "Resource": "<ARN_DA_FILA>",
      "Condition": {
        "Bool": { "aws:SecureTransport": "false" }
      }
    }
  ]
}
```

Adicione também uma **Queue Policy** à `ResourceRegistrationDLQ` permitindo que o EventBridge envie mensagens:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "events.amazonaws.com" },
      "Action": "sqs:SendMessage",
      "Resource": "<ARN_DA_DLQ>",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:events:<REGIAO>:<ACCOUNT_ID>:rule/*"
        }
      }
    }
  ]
}
```

**Evidência a capturar:** print das 3 filas SQS → `evidencias/05-sqs/`

---

## Etapa 6 — EventBridge (Event Buses)

### Por que existem?
- **GlobalEvents bus**: recebe eventos de scheduling publicados pelas Lambdas para distribuição cross-region
- **RegistrationEvents bus**: recebe eventos de tagging das contas spoke para registrar/desregistrar recursos

As **regras de agendamento** (cron) serão criadas na Etapa 10, após as Lambdas existirem.

#### 6.1 — GlobalEvents Bus

| Campo | Valor |
|---|---|
| Nome | `IS-GlobalEvents-default` |

#### 6.2 — RegistrationEvents Bus

| Campo | Valor |
|---|---|
| Nome | `default-RegistrationEvents` |

#### 6.3 — RegistrationEvents Bus Policy

Adicione uma **resource-based policy** ao `RegistrationEvents bus` para permitir que contas spoke enviem eventos. Substitua `<ACCOUNT_ID_SPOKE>` pelo ID da conta spoke (ou use `aws:PrincipalOrgID` para Organizations):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::<ACCOUNT_ID_SPOKE>:root" },
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:<REGIAO>:<SEU_ACCOUNT_ID>:event-bus/default-RegistrationEvents"
    }
  ]
}
```

> Para estudo single-account, esta policy não é necessária na etapa inicial.

**Evidência a capturar:** print dos 2 event buses → `evidencias/06-eventbridge/`

---

## Etapa 7 — IAM Roles e Policies

### Por que existem?
Cada Lambda function precisa de uma IAM Role para executar com o princípio do **least privilege** (mínimo privilégio). A `SchedulerRole` é especial: ela é assumida pelas Lambdas de scheduling via STS para agir nas contas destino.

> Esta é a etapa mais extensa. Crie primeiro todas as Roles, depois adicione as Policies.

### 7.1 — Roles das Lambda Functions

Para cada role abaixo, o **Trust Policy** é o mesmo:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

| Role | Nome sugerido | Lambda que usa |
|---|---|---|
| `MetricsGeneratorRole` | `IS-MetricsGeneratorRole` | MetricsUuidGenerator, HeartbeatMetricsReporter |
| `SnsLogForwarderRole` | `IS-SnsLogForwarderRole` | SnsLogForwarder |
| `SpokeRegistrationRole` | `default-SpokeRegistrationHandler-Role` | SpokeRegistrationHandler |
| `CreateRegionalEventRulesRole` | `IS-CreateRegionalEventRulesRole` | ResourceRegistrationCreateRegionalEventRules |
| `ResourceRegistrationLambdaRole` | `default-ResourceRegistrationHandler-Role` | ResourceRegistrationLambda |
| `RegionRegistrationLambdaRole` | `default-InvokeHubFunctionRemoteRole` | RegionRegistrationCustomResourceLambda |
| `RegionRegistrationWaitRole` | `IS-RegionRegistrationWaitRole` | RegionRegistrationWaitLambda |
| `MainLambdaRole` | `IS-MainLambdaRole` | MainLambda |
| `Ec2ResizeRequestHandlerRole` | `default-Ec2ResizeRequestHandler-Role` | ResizeRequestHandlerLambda |
| `SchedulingRequestHandlerRole` | `default-SchedulingRequestHandler-Role` | schedulingRequestHandlerLambda |
| `SchedulingOrchestratorRole` | `IS-SchedulingOrchestratorRole` | SchedulingOrchestratorLambda |
| `HeartbeatMetricsReporterRole` | `IS-HeartbeatMetricsReporterRole` | HeartbeatMetricsReporter |

### 7.2 — SchedulerRole (a mais importante)

Esta role é assumida pelas Lambdas de scheduling para agir em EC2, RDS e ASG nas contas destino.

**Nome:** `default-Scheduler-Role`

**Trust Policy** (permite que as Lambda roles a assumam via STS):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::<ACCOUNT_ID>:role/default-SchedulingRequestHandler-Role",
          "arn:aws:iam::<ACCOUNT_ID>:role/default-ResourceRegistrationHandler-Role",
          "arn:aws:iam::<ACCOUNT_ID>:role/default-Ec2ResizeRequestHandler-Role",
          "arn:aws:iam::<ACCOUNT_ID>:role/default-SpokeRegistrationHandler-Role"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Policies a anexar à SchedulerRole:**

*EC2 Scheduling:*
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:DescribeInstances","ec2:DescribeInstanceStatus","ec2:DescribeTags"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["ec2:StartInstances","ec2:StopInstances","ec2:CreateTags","ec2:DeleteTags","ec2:ModifyInstanceAttribute"],
      "Resource": "arn:aws:ec2:*:<ACCOUNT_ID>:instance/*"
    },
    {
      "Effect": "Allow",
      "Action": ["ssm:DescribeMaintenanceWindows","ssm:DescribeMaintenanceWindowExecutions"],
      "Resource": "*"
    }
  ]
}
```

*RDS Scheduling:*
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds:DescribeDBInstances","rds:DescribeDBClusters",
        "rds:DescribeDBClusterSnapshots","rds:DescribeDBSnapshots",
        "tag:GetResources"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "rds:StartDBInstance","rds:StopDBInstance",
        "rds:StartDBCluster","rds:StopDBCluster",
        "rds:CreateDBSnapshot","rds:CreateDBClusterSnapshot",
        "rds:AddTagsToResource","rds:RemoveTagsFromResource"
      ],
      "Resource": "*"
    }
  ]
}
```

*ASG Scheduling:*
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["autoscaling:DescribeAutoScalingGroups","autoscaling:DescribeScheduledActions"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:BatchPutScheduledUpdateGroupAction",
        "autoscaling:BatchDeleteScheduledAction",
        "autoscaling:CreateOrUpdateTags",
        "autoscaling:DeleteTags"
      ],
      "Resource": "arn:aws:autoscaling:*:<ACCOUNT_ID>:autoScalingGroup:*"
    }
  ]
}
```

*Resource Tagging:*
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["tag:TagResources","tag:UntagResources"],
      "Resource": "*"
    }
  ]
}
```

*EventBridge local bus:*
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:<REGIAO>:<ACCOUNT_ID>:event-bus/IS-LocalEvents-default"
    }
  ]
}
```

**Evidência a capturar:** print das roles criadas no IAM → `evidencias/07-iam/`

---

## Etapa 8 — Lambda Functions (15 funções)

### Por que existem?
Cada Lambda tem uma responsabilidade específica dentro da solução. O código de todas as funções está empacotado em um único arquivo `.zip` hospedado no S3 da AWS.

---

### URL do código S3 — use esta em todas as funções

```
https://solutions-us-east-1.s3.amazonaws.com/instance-scheduler-on-aws/v3.2.15/c250a3e6cbc3254dde9ff325f9620daecd1007db263036f1cc8f32dff03dc185.zip
```

---

### Log groups — referência rápida

| Log group | Funções que usam |
|---|---|
| `instance-scheduler-on-aws-default-administrative-logs` | MetricsUuidGenerator, SpokeRegistrationHandler, ResourceRegistrationCreateRegionalEventRulesLambda, ResourceRegistrationLambda, RegionRegistrationCustomResourceLambda, RegionRegistrationWaitLambda, MainLambda, HeartbeatMetricsReporter |
| `instance-scheduler-on-aws-default-scheduling-logs` | ResizeRequestHandlerLambda, schedulingRequestHandlerLambda, SchedulingOrchestratorLambda |
| *(sem log group customizado)* | SnsLogForwarder |

---

### Passos para criar cada função

1. **Criar** — nome, runtime Python 3.12, role
2. **Código** — Upload from → Amazon S3 location → cole a URL acima
3. **Handler** — Runtime settings → Edit → substituir pelo handler da função
4. **Memória e Timeout** — Configuration → General configuration → Edit
5. **Variáveis de ambiente** — Configuration → Environment variables → Edit
6. **Log group** — Configuration → Monitoring and operations tools → Edit → campo Log group
7. **Concorrência** — apenas para ResourceRegistrationLambda: Configuration → Concurrency → Reserve concurrency → 1

---

**Runtime de todas as funções:** Python 3.12

**Variáveis de ambiente comuns a todas:**
```
POWERTOOLS_SERVICE_NAME = instance-scheduler
POWERTOOLS_LOG_LEVEL    = INFO
USER_AGENT_EXTRA        = AwsSolution/SO0030/v3.2.15
```

### Funções a criar

#### 8.1 — MetricsUuidGenerator
| Campo | Valor |
|---|---|
| Handler | `instance_scheduler.handler.metrics_uuid_custom_resource.lambda_handler` |
| Role | `IS-MetricsGeneratorRole` |
| Memória | 128 MB |
| Timeout | 60s |
| Log group | `instance-scheduler-on-aws-default-administrative-logs` |
| Variáveis extras | `UUID_KEY=/Solutions/instance-scheduler-on-aws/UUID/`, `STACK_ID=instance-scheduler-on-aws` |

#### 8.2 — SnsLogForwarder
| Campo | Valor |
|---|---|
| Handler | `instance_scheduler.handler.log_forwarding.handle_log_event` |
| Role | `IS-SnsLogForwarderRole` |
| Memória | 512 MB |
| Timeout | 30s |
| Log group | *(não configurar — usa o padrão /aws/lambda/)* |
| Variável extra | `SNS_NOTIFICATION_TOPIC=<ARN_DO_TOPICO_SNS>` |

#### 8.3 — SpokeRegistrationHandler
| Campo | Valor |
|---|---|
| Nome | `InstanceScheduler-default-SpokeRegistration` |
| Handler | `instance_scheduler.handler.spoke_registration.lambda_handler` |
| Role | `default-SpokeRegistrationHandler-Role` |
| Memória | 512 MB |
| Timeout | 900s |
| Log group | `instance-scheduler-on-aws-default-administrative-logs` |
| Variáveis extras | `CONFIG_TABLE`, `REGISTRY_TABLE`, `SCHEDULER_ROLE_NAME=default-Scheduler-Role`, `SCHEDULE_TAG_KEY=Schedule`, `HUB_STACK_NAME=instance-scheduler-on-aws`, `HUB_STACK_ARN=instance-scheduler-on-aws`, `SCHEDULING_INTERVAL_MINUTES=5`, `LOCAL_EVENT_BUS_NAME=IS-LocalEvents-default`, `GLOBAL_EVENT_BUS_NAME=IS-GlobalEvents-default`, `SSM_PARAM_PATH_NAME=/IS/default/regions`, `SSM_PARAM_UPDATE_ROLE_NAME=default-SpokeRegistrationUpdateSSMParamRole`, `ENABLE_INFORMATIONAL_TAGGING=True`, `ASG_SCHEDULED_RULES_PREFIX=IS-`, `ASG_METADATA_TAG_KEY=scheduled` |

#### 8.4 — ResourceRegistrationCreateRegionalEventRulesLambda
| Campo | Valor |
|---|---|
| Handler | `instance_scheduler.handler.create_region_event_rules_handler.lambda_handler` |
| Role | `IS-CreateRegionalEventRulesRole` |
| Memória | 512 MB |
| Timeout | 900s |
| Log group | `instance-scheduler-on-aws-default-administrative-logs` |
| Variáveis extras | `SCHEDULE_TAG_KEY=Schedule`, `REGIONAL_EVENT_BUS_NAME=IS-LocalEvents-default`, `EVENT_RULE_PREFIX=IS-Tagging-default`, `VERSION=v3.2.15`, `TAGGING_EVENT_BUS_ARN=<ARN_DO_default-RegistrationEvents>`, `TAGGING_EVENT_RULE_ROLE_ARN=<ARN_DA_IS-CreateRegionalEventRulesRole>` |

#### 8.5 — ResourceRegistrationLambda
| Campo | Valor |
|---|---|
| Handler | `instance_scheduler.handler.resource_registration_handler.lambda_handler` |
| Role | `default-ResourceRegistrationHandler-Role` |
| Memória | 512 MB |
| Timeout | 300s |
| Log group | `instance-scheduler-on-aws-default-administrative-logs` |
| Concorrência reservada | **1** |
| Variáveis extras | `CONFIG_TABLE`, `REGISTRY_TABLE`, `SCHEDULER_ROLE_NAME=default-Scheduler-Role`, `SCHEDULE_TAG_KEY=Schedule`, `HUB_STACK_NAME=instance-scheduler-on-aws`, `HUB_STACK_ARN=instance-scheduler-on-aws`, `SCHEDULING_INTERVAL_MINUTES=5`, `LOCAL_EVENT_BUS_NAME=IS-LocalEvents-default`, `GLOBAL_EVENT_BUS_NAME=IS-GlobalEvents-default`, `ENABLE_INFORMATIONAL_TAGGING=True`, `ASG_SCHEDULED_RULES_PREFIX=IS-`, `ASG_METADATA_TAG_KEY=scheduled` |

#### 8.6 — RegionRegistrationCustomResourceLambda
| Campo | Valor |
|---|---|
| Handler | `instance_scheduler.handler.region_registration_events_handler.lambda_handler` |
| Role | `default-InvokeHubFunctionRemoteRole` |
| Memória | 512 MB |
| Timeout | 900s |
| Log group | `instance-scheduler-on-aws-default-administrative-logs` |
| Variáveis extras | `HUB_ACCOUNT_ID=<ACCOUNT_ID>`, `STACK_ID=instance-scheduler-on-aws`, `SSM_PARAM_PATH=/IS/default/regions`, `HUB_REGISTRATION_FUNCTION_ARN=<ARN_DA_SpokeRegistrationHandler>`, `HUB_REGISTRATION_ROLE_NAME=default-SpokeRegistrationInvokeFunction-Role` |

#### 8.7 — RegionRegistrationWaitLambda
| Campo | Valor |
|---|---|
| Handler | `instance_scheduler.handler.region_registration_events_iscomplete_handler.lambda_handler` |
| Role | `IS-RegionRegistrationWaitRole` |
| Memória | 512 MB |
| Timeout | 900s |
| Log group | `instance-scheduler-on-aws-default-administrative-logs` |
| Variáveis extras | `HUB_ACCOUNT_ID=<ACCOUNT_ID>`, `STACK_ID=instance-scheduler-on-aws`, `SSM_PARAM_PATH=/IS/default/regions`, `HUB_REGISTRATION_FUNCTION_ARN=<ARN_DA_SpokeRegistrationHandler>`, `HUB_REGISTRATION_ROLE_NAME=default-SpokeRegistrationInvokeFunction-Role` |

#### 8.8 — MainLambda
| Campo | Valor |
|---|---|
| Nome | `instance-scheduler-on-aws-InstanceSchedulerMain` |
| Handler | `instance_scheduler.main.lambda_handler` |
| Role | `IS-MainLambdaRole` |
| Memória | 512 MB |
| Timeout | 300s |
| Log group | `instance-scheduler-on-aws-default-administrative-logs` |
| Variáveis extras | `CONFIG_TABLE`, `ISSUES_TOPIC_ARN=<ARN_DO_SNS_TOPIC>`, `DEFAULT_TIMEZONE=UTC`, `ENABLE_AWS_ORGANIZATIONS=False`, `SCHEDULING_INTERVAL_MINUTES=5`, `HUB_ACCOUNT_ID=<ACCOUNT_ID>`, `SOLUTION_ID=SO0030`, `SOLUTION_VERSION=v3.2.15`, `SEND_METRICS=True`, `METRICS_URL=https://metrics.awssolutionsbuilder.com/generic` |

#### 8.9 — ResizeRequestHandlerLambda
| Campo | Valor |
|---|---|
| Handler | `instance_scheduler.handler.resize_handler.lambda_handler` |
| Role | `default-Ec2ResizeRequestHandler-Role` |
| Memória | 512 MB |
| Timeout | 180s |
| Log group | `instance-scheduler-on-aws-default-scheduling-logs` |
| Variáveis extras | `CONFIG_TABLE`, `REGISTRY_TABLE`, `SCHEDULER_ROLE_NAME=default-Scheduler-Role`, `SCHEDULE_TAG_KEY=Schedule`, `LOCAL_EVENT_BUS_NAME=IS-LocalEvents-default`, `GLOBAL_EVENT_BUS_NAME=IS-GlobalEvents-default`, `ENABLE_INFORMATIONAL_TAGGING=True`, `ASG_SCHEDULED_RULES_PREFIX=IS-`, `ASG_METADATA_TAG_KEY=scheduled`, `HUB_ACCOUNT_ID=<ACCOUNT_ID>`, `SCHEDULING_INTERVAL_MINUTES=5`, `SOLUTION_ID=SO0030`, `SOLUTION_VERSION=v3.2.15`, `SEND_METRICS=True`, `METRICS_URL=https://metrics.awssolutionsbuilder.com/generic` |

#### 8.10 — schedulingRequestHandlerLambda
| Campo | Valor |
|---|---|
| Handler | `instance_scheduler.handler.scheduling_request.handle_scheduling_request` |
| Role | `default-SchedulingRequestHandler-Role` |
| Memória | 512 MB |
| Timeout | 300s |
| Log group | `instance-scheduler-on-aws-default-scheduling-logs` |
| Variáveis extras | `CONFIG_TABLE`, `REGISTRY_TABLE`, `MAINT_WINDOW_TABLE=<NOME_DA_MaintenanceWindowTable>`, `SCHEDULER_ROLE_NAME=default-Scheduler-Role`, `SCHEDULE_TAG_KEY=Schedule`, `DEFAULT_TIMEZONE=UTC`, `ENABLE_EC2_SSM_MAINTENANCE_WINDOWS=False`, `ENABLE_RDS_SNAPSHOTS=False`, `ENABLE_OPS_MONITORING=True`, `LOCAL_EVENT_BUS_NAME=IS-LocalEvents-default`, `GLOBAL_EVENT_BUS_NAME=IS-GlobalEvents-default`, `HUB_STACK_NAME=instance-scheduler-on-aws`, `HUB_STACK_ARN=instance-scheduler-on-aws`, `RESIZE_REQUEST_SQS_URL=<URL_DA_Ec2ResizeRequestQueue>`, `ENABLE_INFORMATIONAL_TAGGING=True`, `ASG_SCHEDULED_RULES_PREFIX=IS-`, `ASG_METADATA_TAG_KEY=scheduled`, `SCHEDULING_INTERVAL_MINUTES=5`, `HUB_ACCOUNT_ID=<ACCOUNT_ID>`, `SOLUTION_ID=SO0030`, `SOLUTION_VERSION=v3.2.15`, `SEND_METRICS=True`, `METRICS_URL=https://metrics.awssolutionsbuilder.com/generic` |

#### 8.11 — SchedulingOrchestratorLambda
| Campo | Valor |
|---|---|
| Handler | `instance_scheduler.handler.scheduling_orchestrator.lambda_handler` |
| Role | `IS-SchedulingOrchestratorRole` |
| Memória | 512 MB |
| Timeout | 300s |
| Log group | `instance-scheduler-on-aws-default-scheduling-logs` |
| Variáveis extras | `CONFIG_TABLE`, `REGISTRY_TABLE`, `SCHEDULING_REQUEST_HANDLER_NAME=<ARN_DA_schedulingRequestHandlerLambda>`, `SCHEDULING_INTERVAL_MINUTES=5` |

> ⚠️ Crie a função 8.10 antes desta. `SCHEDULING_REQUEST_HANDLER_NAME` precisa do ARN da 8.10.

#### 8.12 — HeartbeatMetricsReporter
| Campo | Valor |
|---|---|
| Handler | `instance_scheduler.handler.heartbeat_metrics_reporter.report_heartbeat_metric` |
| Role | `IS-MetricsGeneratorRole` |
| Memória | 256 MB |
| Timeout | 300s |
| Log group | `instance-scheduler-on-aws-default-administrative-logs` |
| Variáveis extras | `CONFIG_TABLE`, `REGISTRY_TABLE`, `DEFAULT_TIMEZONE=UTC`, `ENABLE_RDS_SNAPSHOTS=False`, `ENABLE_AWS_ORGANIZATIONS=False`, `ENABLE_EC2_SSM_MAINTENANCE_WINDOWS=False`, `OPS_DASHBOARD_ENABLED=True`, `SCHEDULING_INTERVAL_MINUTES=5`, `HUB_ACCOUNT_ID=<ACCOUNT_ID>`, `SOLUTION_ID=SO0030`, `SOLUTION_VERSION=v3.2.15`, `SEND_METRICS=True`, `METRICS_URL=https://metrics.awssolutionsbuilder.com/generic` |

**Evidência a capturar:** print das funções Lambda criadas → `evidencias/08-lambda/`

---

## Etapa 9 — Lambda Permissions e Event Source Mapping

### 9.1 — Permissão: logs.amazonaws.com → SnsLogForwarder (2x)

Adicione as permissões abaixo para que o CloudWatch Logs possa invocar a `SnsLogForwarder`:

```
Principal:   logs.amazonaws.com
Action:      lambda:InvokeFunction
Source ARN:  <ARN_DO_AdministrationLogs>
```

```
Principal:   logs.amazonaws.com
Action:      lambda:InvokeFunction
Source ARN:  <ARN_DO_SchedulingLogs>
```

### 9.2 — Event Source Mapping: SQS → ResizeRequestHandlerLambda

| Campo | Valor |
|---|---|
| Trigger | `Ec2ResizeRequestQueue` |
| Batch size | 1 |
| Function | `ResizeRequestHandlerLambda` |

---

## Etapa 10 — EventBridge Rules (4 regras)

### 10.1 — SchedulerEventRule (cron principal)

| Campo | Valor |
|---|---|
| Nome | `IS-SchedulerEventRule` |
| Bus | default (event bus padrão da AWS) |
| Schedule | `cron(0/5 * * * ? *)` — a cada 5 minutos |
| Target | ARN da `SchedulingOrchestratorLambda` |
| Estado | ENABLED |

> Ajuste o cron conforme o `SchedulerFrequency` desejado. Exemplos: `cron(0/10 * * * ? *)` para 10 min, `cron(0 * * * ? *)` para 60 min.

### 10.2 — MetricsGathererEventRule (diário)

| Campo | Valor |
|---|---|
| Schedule | `rate(1 day)` |
| Target | ARN da `HeartbeatMetricsReporter` |

### 10.3 — TaggingEventsToLambda (tag events EC2/RDS)

| Campo | Valor |
|---|---|
| Bus | default |
| Event pattern | `{"source": ["aws.tag"], "detail-type": ["Tag Change on Resource"], "detail": {"changed-tag-keys": ["Schedule"]}}` |
| Target | ARN da `ResourceRegistrationLambda` |
| DLQ | ARN da `ResourceRegistrationDLQ` |

### 10.4 — AsgTaggingEventsToLambda (tag events ASG)

| Campo | Valor |
|---|---|
| Bus | default |
| Event pattern | `{"source": ["aws.autoscaling"], "detail-type": ["EC2 Auto Scaling Group Created", "EC2 Auto Scaling Group Tags Modified"]}` |
| Target | ARN da `ResourceRegistrationLambda` |
| DLQ | ARN da `ResourceRegistrationDLQ` |

**Evidência a capturar:** print das 4 regras no console EventBridge → `evidencias/09-eventbridge-rules/`

---

## Etapa 11 — CloudWatch Logs Subscription Filters

Estes filtros encaminham logs de ERROR e WARNING ao `SnsLogForwarder` automaticamente.

### 11.1 — Filtro em AdministrationLogs
| Campo | Valor |
|---|---|
| Log group | `AdministrationLogs` |
| Filter pattern | `{ ($.level = "ERROR") \|\| ($.level = "WARNING") }` |
| Destination ARN | ARN da `SnsLogForwarder` |

### 11.2 — Filtro em SchedulingLogs
| Campo | Valor |
|---|---|
| Log group | `SchedulingLogs` |
| Filter pattern | `{ ($.level = "ERROR") \|\| ($.level = "WARNING") }` |
| Destination ARN | ARN da `SnsLogForwarder` |

---

## Etapa 12 — Step Functions State Machine

### Por que existe?
O Custom Resource de registro de regiões é **assíncrono** — pode demorar até 20 minutos para concluir. A state machine implementa um padrão de **polling** (verifica a cada 30 segundos por até 40 tentativas = ~20 minutos) para aguardar a conclusão.

> Em um estudo single-account, esta etapa pode ser simplificada. A state machine é parte do padrão CDK `custom-resources.Provider` e raramente precisa ser recriada manualmente fora de um deploy CDK.

---

## Etapa 13 — Inicialização (Custom Resources)

Nesta etapa, as configurações iniciais são aplicadas chamando as Lambda functions diretamente.

### 13.1 — Gerar UUID de métricas
Invoque a `MetricsUuidGenerator` pelo CloudShell:
```bash
echo '{"RequestType": "Create", "ResourceProperties": {}}' > payload.json

aws lambda invoke \
  --function-name <NOME_DA_FUNCAO_MetricsUuidGenerator> \
  --payload fileb://payload.json \
  output.json

cat output.json
```

### 13.2 — Setup inicial de schedules (MainLambda)
Invoque a `MainLambda` para criar os schedules padrão no DynamoDB:
```bash
echo '{"RequestType": "Create", "ResourceProperties": {"ServiceToken": "<ARN_MAIN_LAMBDA>"}}' > payload.json

aws lambda invoke \
  --function-name <STACK_NAME>-InstanceSchedulerMain \
  --payload fileb://payload.json \
  output.json

cat output.json
```

### 13.3 — Criar regras EventBridge regionais
Invoque a `ResourceRegistrationCreateRegionalEventRulesLambda`:
```bash
echo '{"RequestType": "Create", "ResourceProperties": {"regions": ["us-east-1"], "namespace": "default"}}' > payload.json

aws lambda invoke \
  --function-name <NOME_DA_FUNCAO> \
  --payload fileb://payload.json \
  output.json

cat output.json
```

---

## Etapa 14 — CloudWatch Dashboard (opcional)

Crie um dashboard chamado `<STACK_NAME>-default-Operational-Insights-Dashboard` com os seguintes widgets:

| Widget | Métrica |
|---|---|
| EC2 instances controlled | `InstanceScheduler/EC2` — `InstanceCount` |
| RDS instances controlled | `InstanceScheduler/RDS` — `InstanceCount` |
| Hours saved | `InstanceScheduler` — `SavedHours` |
| Lambda errors | `AWS/Lambda` — `Errors` (SchedulingOrchestrator + SchedulingRequestHandler) |
| Lambda duration | `AWS/Lambda` — `Duration` |

---

## Etapa 15 — CloudWatch Logs Insights Queries

Crie as queries salvas abaixo para facilitar a análise de logs durante o estudo:

### Histórico de scheduling decisions
```
fields @timestamp, account, region, service, instance_id, action, schedule_name
| filter @logStream like /scheduling/
| sort @timestamp desc
| limit 100
```

### Eventos de registro de recursos
```
fields @timestamp, account, region, resource_id, event_type
| filter @logStream like /registration/
| sort @timestamp desc
| limit 100
```

### Erros e warnings
```
fields @timestamp, level, message
| filter level in ["ERROR", "WARNING"]
| sort @timestamp desc
| limit 50
```

---

## Testando o Scheduler

Após todos os recursos criados:

1. **Crie uma instância EC2** de teste (t2.micro ou t3.micro)
2. **Adicione a tag:**
   ```
   Chave: Schedule
   Valor: stopped
   ```
   > O schedule `stopped` vem pré-configurado e para a instância imediatamente
3. **Aguarde o próximo ciclo** (até 5 minutos) e observe nos CloudWatch Logs a instância sendo parada
4. **Altere a tag** para `running` e observe a instância sendo iniciada

**Evidência a capturar:** logs do CloudWatch Logs mostrando a decisão de scheduling → `evidencias/10-execucao/`

---

## Checklist Final

- [ ] KMS Key criada com rotação habilitada
- [ ] 4 tabelas DynamoDB criadas com KMS e PITR
- [ ] 2 Log Groups criados com retenção configurada
- [ ] SNS Topic criado e criptografado
- [ ] 3 filas SQS criadas (2 DLQs + 1 principal)
- [ ] 2 Event Buses criados (GlobalEvents + RegistrationEvents)
- [ ] 19 IAM Roles criadas com trust policies corretas
- [ ] SchedulerRole com todas as policies de EC2/RDS/ASG/tag
- [ ] 15 Lambda Functions criadas com variáveis de ambiente corretas
- [ ] 4 EventBridge Rules criadas (cron + tag events)
- [ ] 2 Subscription Filters criados nos Log Groups
- [ ] Scheduler testado com instância EC2

---

## Recursos de Apoio

- [Template CloudFormation completo](../instance-scheduler-on-aws.template)
- [Análise do projeto](../ANALISE_PROJETO.md)
- [Documentação oficial da solução](https://docs.aws.amazon.com/solutions/latest/instance-scheduler-on-aws/solution-overview.html)
- [Código-fonte original no GitHub](https://github.com/aws-solutions-library-samples/instance-scheduler-on-aws)
