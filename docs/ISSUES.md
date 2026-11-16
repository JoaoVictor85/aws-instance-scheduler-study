# Mapeamento de Issues — AWS Instance Scheduler Study

Este documento lista todas as issues que devem ser criadas no GitHub, organizadas por milestone/fase.
Crie-as na ordem apresentada — a numeração reflete as dependências entre etapas.

> **Como criar:** acesse `https://github.com/JoaoVictor85/aws-instance-scheduler-study/issues/new/choose`
> e selecione o template **⚙️ Etapa de Execução** para cada issue abaixo.

---

## 📌 Issues de Setup (criar antes de tudo)

---

### ISSUE #1 — Configuração inicial do repositório GitHub

| Campo | Valor |
|---|---|
| **Título** | `[SETUP] Configurar repositório, labels, milestones e project board` |
| **Labels** | `documentação` |
| **Milestone** | *(nenhuma)* |
| **Projeto** | `AWS Instance Scheduler — Estudo Prático` |

**Checklist:**
- [ ] Repositório criado no GitHub como público
- [ ] Primeiro push realizado com todos os arquivos
- [ ] 16 labels criadas conforme `docs/SETUP_GITHUB.md`
- [ ] 8 milestones criadas
- [ ] Project board criado com 6 colunas
- [ ] Project vinculado ao repositório
- [ ] Templates de issue visíveis no repositório

---

## 🏗️ Fase 1 — Fundação

*Milestone: `Fase 1 — Fundação`*
Estes são os recursos base. Tudo que vem depois depende deles — especialmente da chave KMS.

---

### ISSUE #2 — KMS Key e Alias

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 1] Criar KMS Customer Managed Key e Alias` |
| **Labels** | `execução`, `kms` |
| **Milestone** | `Fase 1 — Fundação` |

**Checklist:**
- [ ] KMS Key simétrica criada com rotação automática habilitada
- [ ] Key policy com statement para root da conta (`kms:*`)
- [ ] Key policy com statement para `logs.amazonaws.com` (Decrypt, Encrypt, ReEncrypt*, GenerateDataKey*)
- [ ] Alias `alias/AwsSolutions/InstanceScheduler/default/instance-scheduler-on-aws` criado
- [ ] ARN da key anotado para uso nas próximas etapas
- [ ] Print salvo em `evidencias/01-kms/`

**Dependências:** nenhuma — primeira etapa.

---

### ISSUE #3 — DynamoDB: ConfigTable

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 2a] Criar DynamoDB ConfigTable` |
| **Labels** | `execução`, `dynamodb` |
| **Milestone** | `Fase 1 — Fundação` |

**Checklist:**
- [ ] Tabela criada com Partition Key `type` (String) e Sort Key `name` (String)
- [ ] Billing mode: PAY_PER_REQUEST
- [ ] SSE habilitado com KMS CMK da Issue #2
- [ ] PITR habilitado
- [ ] DynamoDB Streams habilitado no modo KEYS_ONLY
- [ ] ARN da tabela anotado
- [ ] Print salvo em `evidencias/02-dynamodb/`

**Dependências:** #2

---

### ISSUE #4 — DynamoDB: StateTable

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 2b] Criar DynamoDB StateTable` |
| **Labels** | `execução`, `dynamodb` |
| **Milestone** | `Fase 1 — Fundação` |

**Checklist:**
- [ ] Tabela criada com Partition Key `service` (String) e Sort Key `account-region` (String)
- [ ] Billing mode: PAY_PER_REQUEST
- [ ] SSE habilitado com KMS CMK da Issue #2
- [ ] PITR habilitado
- [ ] ARN da tabela anotado
- [ ] Print salvo em `evidencias/02-dynamodb/`

**Dependências:** #2

---

### ISSUE #5 — DynamoDB: ResourceRegistry

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 2c] Criar DynamoDB ResourceRegistry` |
| **Labels** | `execução`, `dynamodb` |
| **Milestone** | `Fase 1 — Fundação` |

**Checklist:**
- [ ] Tabela criada com Partition Key `account` (String) e Sort Key `sk` (String)
- [ ] Billing mode: PAY_PER_REQUEST
- [ ] SSE habilitado com KMS CMK da Issue #2
- [ ] PITR habilitado
- [ ] ARN da tabela anotado
- [ ] Print salvo em `evidencias/02-dynamodb/`

**Dependências:** #2

---

### ISSUE #6 — DynamoDB: MaintenanceWindowTable

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 2d] Criar DynamoDB MaintenanceWindowTable` |
| **Labels** | `execução`, `dynamodb` |
| **Milestone** | `Fase 1 — Fundação` |

**Checklist:**
- [ ] Tabela criada com Partition Key `account-region` (String) e Sort Key `name-id` (String)
- [ ] Billing mode: PAY_PER_REQUEST
- [ ] SSE habilitado com KMS CMK da Issue #2
- [ ] PITR habilitado
- [ ] ARN da tabela anotado
- [ ] Print salvo em `evidencias/02-dynamodb/`

**Dependências:** #2

---

### ISSUE #7 — CloudWatch Log Groups

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 3] Criar CloudWatch Log Groups` |
| **Labels** | `execução`, `cloudwatch` |
| **Milestone** | `Fase 1 — Fundação` |

**Checklist:**
- [ ] Log group `instance-scheduler-on-aws-default-administrative-logs` criado
- [ ] Log group `instance-scheduler-on-aws-default-scheduling-logs` criado
- [ ] Ambos com retenção de 30 dias
- [ ] Ambos com KMS Key da Issue #2 associada
- [ ] ARNs anotados
- [ ] Print salvo em `evidencias/03-cloudwatch-logs/`

**Dependências:** #2

---

### ISSUE #8 — SNS Topic

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 4] Criar SNS Topic e Topic Policy` |
| **Labels** | `execução`, `sns` |
| **Milestone** | `Fase 1 — Fundação` |

**Checklist:**
- [ ] SNS Topic Standard criado com criptografia KMS (Issue #2)
- [ ] Topic Policy adicionada negando publicações sem SSL (`aws:SecureTransport: false`)
- [ ] ARN do tópico anotado
- [ ] Print salvo em `evidencias/04-sns/`

**Dependências:** #2

---

### ISSUE #9 — SQS Queues

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 5] Criar SQS Queues (DLQs e fila principal)` |
| **Labels** | `execução`, `sqs` |
| **Milestone** | `Fase 1 — Fundação` |

**Checklist:**
- [ ] `ResourceRegistrationDLQ` criada (SSE gerenciado pelo SQS)
- [ ] Queue Policy da `ResourceRegistrationDLQ` permitindo `events.amazonaws.com` publicar
- [ ] `Ec2ResizeDLQueue` criada com KMS da Issue #2
- [ ] Queue Policy da `Ec2ResizeDLQueue` negando acesso sem SSL
- [ ] `Ec2ResizeRequestQueue` criada com KMS da Issue #2, Visibility Timeout 180s, DLQ apontando para `Ec2ResizeDLQueue`, Max receive count 1
- [ ] Queue Policy da `Ec2ResizeRequestQueue` negando acesso sem SSL
- [ ] URLs e ARNs das filas anotados
- [ ] Print salvo em `evidencias/05-sqs/`

**Dependências:** #2

---

## 🔀 Fase 2 — Eventos e Mensageria

*Milestone: `Fase 2 — Eventos e Mensageria`*

---

### ISSUE #10 — EventBridge Event Buses

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 6] Criar EventBridge Event Buses` |
| **Labels** | `execução`, `eventbridge` |
| **Milestone** | `Fase 2 — Eventos e Mensageria` |

**Checklist:**
- [ ] Event Bus `IS-GlobalEvents-default` criado
- [ ] Event Bus `default-RegistrationEvents` criado
- [ ] ARNs e nomes dos buses anotados
- [ ] Print salvo em `evidencias/06-eventbridge/`

**Dependências:** #3, #4, #5, #6 (DynamoDB concluído)

---

## 🔐 Fase 3 — Identidade e Acesso

*Milestone: `Fase 3 — Identidade e Acesso`*
Esta é a fase mais extensa. IAM é o pilar de segurança de tudo — cada role define exatamente o que cada Lambda pode ou não fazer.

---

### ISSUE #11 — IAM Roles das Lambda Functions

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 7a] Criar IAM Roles para as Lambda Functions` |
| **Labels** | `execução`, `iam` |
| **Milestone** | `Fase 3 — Identidade e Acesso` |

**Checklist:**
- [ ] `IS-MetricsGeneratorRole` criada (trust: lambda.amazonaws.com)
- [ ] `IS-SnsLogForwarderRole` criada (trust: lambda.amazonaws.com)
- [ ] `default-SpokeRegistrationHandler-Role` criada (trust: lambda.amazonaws.com)
- [ ] `IS-CreateRegionalEventRulesRole` criada (trust: lambda.amazonaws.com)
- [ ] `default-ResourceRegistrationHandler-Role` criada (trust: lambda.amazonaws.com)
- [ ] `default-InvokeHubFunctionRemoteRole` criada (trust: lambda.amazonaws.com)
- [ ] `IS-RegionRegistrationWaitRole` criada (trust: lambda.amazonaws.com)
- [ ] `IS-MainLambdaRole` criada (trust: lambda.amazonaws.com)
- [ ] `default-Ec2ResizeRequestHandler-Role` criada (trust: lambda.amazonaws.com)
- [ ] `default-SchedulingRequestHandler-Role` criada (trust: lambda.amazonaws.com)
- [ ] `IS-SchedulingOrchestratorRole` criada (trust: lambda.amazonaws.com)
- [ ] `IS-HeartbeatMetricsReporterRole` criada (trust: lambda.amazonaws.com)
- [ ] ARNs de todas as roles anotados
- [ ] Print salvo em `evidencias/07-iam/`

**Dependências:** #10

---

### ISSUE #12 — IAM SchedulerRole (cross-account)

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 7b] Criar IAM SchedulerRole com permissões EC2, RDS, ASG e Tagging` |
| **Labels** | `execução`, `iam` |
| **Milestone** | `Fase 3 — Identidade e Acesso` |

**Checklist:**
- [ ] Role `default-Scheduler-Role` criada
- [ ] Trust Policy configurada permitindo as 4 Lambda roles assumirem via STS
- [ ] Policy de EC2 Scheduling anexada (Describe*, Start, Stop, CreateTags, DeleteTags, ModifyInstanceAttribute, SSM:DescribeMaintenanceWindows)
- [ ] Policy de RDS Scheduling anexada (Describe*, Start, Stop, CreateSnapshot, AddTags, RemoveTags)
- [ ] Policy de ASG Scheduling anexada (Describe*, BatchPutScheduledUpdateGroupAction, BatchDeleteScheduledAction, CreateOrUpdateTags, DeleteTags)
- [ ] Policy de Resource Tagging anexada (tag:TagResources, tag:UntagResources)
- [ ] Policy de EventBridge local bus anexada (events:PutEvents para `IS-LocalEvents-default`)
- [ ] ARN da role anotado
- [ ] Print salvo em `evidencias/07-iam/`

**Dependências:** #11

---

### ISSUE #13 — IAM Policies das Lambda Roles

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 7c] Anexar políticas IAM às roles das Lambda Functions` |
| **Labels** | `execução`, `iam` |
| **Milestone** | `Fase 3 — Identidade e Acesso` |

**Checklist:**
- [ ] `IS-MetricsGeneratorRole`: xray:PutTraceSegments, xray:PutTelemetryRecords + logs:CreateLogStream/PutLogEvents + ssm:GetParameter* (path UUID)
- [ ] `IS-SnsLogForwarderRole`: logs:*, xray:*, sns:Publish, kms:Decrypt/GenerateDataKey
- [ ] `default-SpokeRegistrationHandler-Role`: sts:AssumeRole (spokes), ssm:*, kms:*, dynamodb (Config+Registry), events:PutEvents, logs
- [ ] `IS-CreateRegionalEventRulesRole`: events:DeleteRule/PutTargets/PutRule/RemoveTargets, events:CreateEventBus/DeleteEventBus (IS-LocalEvents-*), iam:PassRole
- [ ] `default-ResourceRegistrationHandler-Role`: sts:AssumeRole, xray, logs, dynamodb (Registry+Config), kms, events:PutEvents
- [ ] `IS-MainLambdaRole`: xray, dynamodb (ConfigTable), kms, logs, sns:Publish
- [ ] `default-Ec2ResizeRequestHandler-Role`: sqs:Receive/Delete/GetAttributes (ResizeRequestQueue), kms:Decrypt, dynamodb (Registry), events:PutEvents, logs, sts:AssumeRole
- [ ] `default-SchedulingRequestHandler-Role`: kms, dynamodb (Config+State+Registry+MaintenanceWindow), sqs:SendMessage (ResizeRequestQueue), events:PutEvents, logs, sts:AssumeRole, cloudwatch:PutMetricData
- [ ] `IS-SchedulingOrchestratorRole`: xray, lambda:InvokeFunction (schedulingRequestHandler), kms, dynamodb (Config+Registry), logs, ssm:GetParameter*
- [ ] `IS-HeartbeatMetricsReporterRole`: xray, kms, dynamodb (Config+Registry), logs, cloudwatch:PutMetricData
- [ ] Print salvo em `evidencias/07-iam/`

**Dependências:** #11, #12

---

## ⚡ Fase 4 — Computação

*Milestone: `Fase 4 — Computação`*

---

### ISSUE #14 — Lambda: funções de suporte

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 8a] Criar Lambda Functions de suporte (MetricsUuidGenerator, SnsLogForwarder, HeartbeatMetricsReporter)` |
| **Labels** | `execução`, `lambda` |
| **Milestone** | `Fase 4 — Computação` |

**Checklist:**
- [ ] `MetricsUuidGenerator` criada (runtime Python 3.12, handler `instance_scheduler.handler.metrics_uuid_custom_resource.lambda_handler`, 128MB, 60s, log group AdministrationLogs)
- [ ] `SnsLogForwarder` criada (512MB, 30s, variável `SNS_NOTIFICATION_TOPIC`)
- [ ] `HeartbeatMetricsReporter` criada (512MB, 300s, log group AdministrationLogs)
- [ ] Variáveis de ambiente comuns configuradas em todas (`POWERTOOLS_SERVICE_NAME`, `POWERTOOLS_LOG_LEVEL`, `USER_AGENT_EXTRA`)
- [ ] Print salvo em `evidencias/08-lambda/`

**Dependências:** #7, #8, #13 — log groups, SNS e roles precisam existir

---

### ISSUE #15 — Lambda: funções de registro

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 8b] Criar Lambda Functions de registro (SpokeRegistration, ResourceRegistration, RegionRegistration)` |
| **Labels** | `execução`, `lambda` |
| **Milestone** | `Fase 4 — Computação` |

**Checklist:**
- [ ] `SpokeRegistrationHandler` criada (nome `InstanceScheduler-default-SpokeRegistration`, 512MB, 900s, todas as variáveis de ambiente configuradas)
- [ ] `ResourceRegistrationCreateRegionalEventRulesLambda` criada (900s, `IS-CreateRegionalEventRulesRole`)
- [ ] `ResourceRegistrationLambda` criada (300s, Reserved Concurrency = 1)
- [ ] `RegionRegistrationCustomResourceLambda` criada (900s)
- [ ] `RegionRegistrationWaitLambda` criada (900s)
- [ ] Print salvo em `evidencias/08-lambda/`

**Dependências:** #7, #10, #13

---

### ISSUE #16 — Lambda: funções de agendamento

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 8c] Criar Lambda Functions de agendamento (MainLambda, ResizeHandler, SchedulingRequestHandler, SchedulingOrchestrator)` |
| **Labels** | `execução`, `lambda` |
| **Milestone** | `Fase 4 — Computação` |

**Checklist:**
- [ ] `MainLambda` criada (nome `instance-scheduler-on-aws-InstanceSchedulerMain`, 512MB, 300s, log group AdministrationLogs)
- [ ] `ResizeRequestHandlerLambda` criada (180s, `default-Ec2ResizeRequestHandler-Role`)
- [ ] `schedulingRequestHandlerLambda` criada (512MB, 300s, log group SchedulingLogs, todas as variáveis configuradas incluindo ARNs das tabelas DynamoDB e URL da SQS)
- [ ] `SchedulingOrchestratorLambda` criada (512MB, 300s, log group SchedulingLogs, variável `SCHEDULING_REQUEST_HANDLER_NAME` apontando para o ARN do handler)
- [ ] Print salvo em `evidencias/08-lambda/`

**Dependências:** #3, #4, #5, #6, #7, #9, #13

---

### ISSUE #17 — Lambda Permissions e Event Source Mapping

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 9] Configurar Lambda Permissions e Event Source Mapping (SQS)` |
| **Labels** | `execução`, `lambda`, `sqs` |
| **Milestone** | `Fase 4 — Computação` |

**Checklist:**
- [ ] Permission adicionada: `logs.amazonaws.com` pode invocar `SnsLogForwarder` a partir do `AdministrationLogs`
- [ ] Permission adicionada: `logs.amazonaws.com` pode invocar `SnsLogForwarder` a partir do `SchedulingLogs`
- [ ] Event Source Mapping criado: `Ec2ResizeRequestQueue` → `ResizeRequestHandlerLambda` (batch size 1)
- [ ] Print salvo em `evidencias/08-lambda/`

**Dependências:** #14, #15, #16

---

## 🔀 Fase 5 — Orquestração

*Milestone: `Fase 5 — Orquestração`*

---

### ISSUE #18 — EventBridge Rules

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 10] Criar EventBridge Rules (cron e tag events)` |
| **Labels** | `execução`, `eventbridge` |
| **Milestone** | `Fase 5 — Orquestração` |

**Checklist:**
- [ ] `IS-SchedulerEventRule` criada: cron `0/5 * * * ? *` → `SchedulingOrchestratorLambda` (estado ENABLED)
- [ ] `MetricsGathererEventRule` criada: `rate(1 day)` → `HeartbeatMetricsReporter`
- [ ] `TaggingEventsToLambda` criada: source `aws.tag`, detail-type `Tag Change on Resource`, changed-tag-keys `[Schedule]` → `ResourceRegistrationLambda` com DLQ `ResourceRegistrationDLQ`
- [ ] `AsgTaggingEventsToLambda` criada: source `aws.autoscaling`, detail-type com eventos de criação e modificação de ASG → `ResourceRegistrationLambda`
- [ ] Print salvo em `evidencias/09-eventbridge-rules/`

**Dependências:** #14, #15, #16

---

### ISSUE #19 — CloudWatch Logs Subscription Filters

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 11] Criar Subscription Filters nos Log Groups` |
| **Labels** | `execução`, `cloudwatch` |
| **Milestone** | `Fase 5 — Orquestração` |

**Checklist:**
- [ ] Subscription Filter criado no `AdministrationLogs`: padrão `{ ($.level = "ERROR") || ($.level = "WARNING") }` → `SnsLogForwarder`
- [ ] Subscription Filter criado no `SchedulingLogs`: mesmo padrão → `SnsLogForwarder`
- [ ] Print salvo em `evidencias/09-eventbridge-rules/`

**Dependências:** #14, #17

---

### ISSUE #20 — Step Functions State Machine

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 12] Criar Step Functions State Machine (waiter para registro de regiões)` |
| **Labels** | `execução`, `stepfunctions` |
| **Milestone** | `Fase 5 — Orquestração` |

**Checklist:**
- [ ] State machine criada com pattern de polling: intervalo 30s, máximo 40 tentativas
- [ ] State machine integrada com `RegionRegistrationWaitLambda` (isComplete) e `RegionRegistrationCustomResourceLambda` (onTimeout)
- [ ] Role da state machine com permissão de invocar as duas Lambda functions
- [ ] ARN da state machine anotado
- [ ] Print salvo em `evidencias/10-stepfunctions/`

**Dependências:** #15

---

## 🚀 Fase 6 — Inicialização

*Milestone: `Fase 6 — Inicialização`*

---

### ISSUE #21 — Custom Resources: inicialização via CloudShell

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 13] Executar Custom Resources via CloudShell (UUID, schedules padrão, regras regionais)` |
| **Labels** | `execução`, `lambda` |
| **Milestone** | `Fase 6 — Inicialização` |

**Checklist:**
- [ ] `MetricsUuidGenerator` invocada via CloudShell com payload `{"RequestType": "Create", "ResourceProperties": {}}` — UUID gerado e armazenado no SSM
- [ ] `MainLambda` invocada via CloudShell — schedules padrão criados na ConfigTable
- [ ] Verificado no DynamoDB que a ConfigTable possui itens após a invocação
- [ ] `ResourceRegistrationCreateRegionalEventRulesLambda` invocada — event buses locais `IS-LocalEvents-default` criados na região
- [ ] Print do CloudShell com outputs das invocações salvo em `evidencias/11-inicializacao/`
- [ ] Print da ConfigTable com itens criados salvo em `evidencias/11-inicializacao/`

**Dependências:** #16, #17, #18, #19, #20

---

## 📊 Fase 7 — Observabilidade

*Milestone: `Fase 7 — Observabilidade`*

---

### ISSUE #22 — CloudWatch Dashboard

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 14] Criar CloudWatch Operational Insights Dashboard` |
| **Labels** | `execução`, `cloudwatch` |
| **Milestone** | `Fase 7 — Observabilidade` |

**Checklist:**
- [ ] Dashboard `instance-scheduler-on-aws-default-Operational-Insights-Dashboard` criado
- [ ] Widget: EC2 instances controlled (métrica `InstanceScheduler/EC2 → InstanceCount`)
- [ ] Widget: RDS instances controlled (métrica `InstanceScheduler/RDS → InstanceCount`)
- [ ] Widget: Hours saved (métrica `InstanceScheduler → SavedHours`)
- [ ] Widget: Lambda Errors (SchedulingOrchestrator + SchedulingRequestHandler)
- [ ] Widget: Lambda Duration (SchedulingOrchestrator + SchedulingRequestHandler)
- [ ] Print do dashboard salvo em `evidencias/12-dashboard/`

**Dependências:** #21

---

### ISSUE #23 — CloudWatch Logs Insights Queries

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 15] Criar CloudWatch Logs Insights Queries salvas` |
| **Labels** | `execução`, `cloudwatch` |
| **Milestone** | `Fase 7 — Observabilidade` |

**Checklist:**
- [ ] Query "Histórico de scheduling decisions" criada no SchedulingLogs
- [ ] Query "Eventos de registro de recursos" criada no AdministrationLogs
- [ ] Query "Erros e warnings" criada em ambos os log groups
- [ ] Print das queries salvo em `evidencias/12-dashboard/`

**Dependências:** #21

---

## ✅ Fase 8 — Validação

*Milestone: `Fase 8 — Validação`*

---

### ISSUE #24 — Teste end-to-end com EC2

| Campo | Valor |
|---|---|
| **Título** | `[ETAPA 16] Validar o scheduler com instância EC2 real` |
| **Labels** | `execução`, `lambda`, `dynamodb` |
| **Milestone** | `Fase 8 — Validação` |

**Checklist:**
- [ ] Instância EC2 (t3.micro) criada em estado Running
- [ ] Tag `Schedule=stopped` adicionada à instância
- [ ] Aguardado até 5 minutos pelo próximo ciclo do scheduler
- [ ] CloudWatch Logs do `SchedulingLogs` mostram a decisão de parar a instância
- [ ] Instância EC2 parada pelo scheduler (estado `stopped`)
- [ ] Tag alterada para `Schedule=running`
- [ ] Instância EC2 iniciada pelo scheduler (estado `running`)
- [ ] ResourceRegistry no DynamoDB inspecionado — instância registrada
- [ ] StateTable no DynamoDB inspecionado — estado da instância registrado
- [ ] Prints de todas as evidências salvos em `evidencias/13-validacao/`
- [ ] Instância EC2 de teste **terminada** após a validação (evitar custos)

**Dependências:** #21, #22, #23

---

### ISSUE #25 — Documentar aprendizados e encerrar projeto

| Campo | Valor |
|---|---|
| **Título** | `[ENCERRAMENTO] Consolidar aprendizados e finalizar documentação do portfólio` |
| **Labels** | `documentação`, `aprendizado` |
| **Milestone** | `Fase 8 — Validação` |

**Checklist:**
- [ ] Seção "Aprendizados" do `README.md` preenchida com os principais conceitos absorvidos
- [ ] Todas as issues fechadas com evidências documentadas
- [ ] Todas as evidências organizadas nas pastas corretas e commitadas
- [ ] Commit final com mensagem descritiva feito
- [ ] `ANALISE_PROJETO.md` revisado e atualizado com observações pós-execução
- [ ] Projeto compartilhado em perfil profissional (LinkedIn, etc.)

**Dependências:** #24

---

## Resumo

| Fase | Issues | Serviços |
|---|---|---|
| Setup | #1 | GitHub |
| Fase 1 — Fundação | #2 → #9 | KMS, DynamoDB, CloudWatch, SNS, SQS |
| Fase 2 — Eventos | #10 | EventBridge |
| Fase 3 — IAM | #11 → #13 | IAM |
| Fase 4 — Computação | #14 → #17 | Lambda, SQS |
| Fase 5 — Orquestração | #18 → #20 | EventBridge, CloudWatch, Step Functions |
| Fase 6 — Inicialização | #21 | Lambda (CloudShell) |
| Fase 7 — Observabilidade | #22 → #23 | CloudWatch |
| Fase 8 — Validação | #24 → #25 | EC2, DynamoDB, CloudWatch |
| **Total** | **25 issues** | **11 serviços AWS** |
