# AWS Instance Scheduler on AWS — Estudo Prático

![Status](https://img.shields.io/badge/status-em%20execução-yellow?style=flat-square)
![AWS](https://img.shields.io/badge/AWS-Cloud-orange?style=flat-square&logo=amazonaws)
![Solução](https://img.shields.io/badge/Solução%20AWS-SO0030%20v3.2.15-blue?style=flat-square)
![Tipo](https://img.shields.io/badge/deploy-manual%20via%20console-purple?style=flat-square)
![Licença](https://img.shields.io/badge/licença-Apache%202.0-green?style=flat-square)

Estudo prático de implantação **manual** da solução oficial da AWS **Instance Scheduler on AWS** em uma conta Sandbox. Em vez do deploy automático via CloudFormation, todos os recursos foram criados individualmente no console AWS, com o objetivo de compreender cada serviço, sua configuração e as dependências entre eles.

---

## Sobre a Solução

O **Instance Scheduler on AWS** é uma solução open-source mantida pela AWS que automatiza o **start e stop de instâncias EC2, RDS e Auto Scaling Groups** com base em agendamentos configuráveis. O objetivo principal é reduzir custos operacionais parando recursos fora do horário de uso.

| | |
|---|---|
| **Solução oficial** | [aws.amazon.com/solutions — Instance Scheduler](https://aws.amazon.com/solutions/implementations/instance-scheduler-on-aws/) |
| **Repositório original** | [aws-solutions-library-samples/instance-scheduler-on-aws](https://github.com/aws-solutions-library-samples/instance-scheduler-on-aws) |
| **Versão estudada** | v3.2.15 |
| **Referência de deploy** | `instance-scheduler-on-aws.template` (CloudFormation, 8067 linhas) |

---

## Arquitetura

A solução é baseada no padrão **Hub-and-Spoke**: uma conta central (Hub) gerencia o agendamento de recursos em uma ou várias contas gerenciadas (Spokes).

```mermaid
flowchart TD
    subgraph HUB["🏠 Hub Account"]
        direction TB

        EB["⏰ EventBridge\nCron Rule\n(a cada 5 min)"]

        subgraph MAIN["Main Lambda\n(CLI + CFN Custom Resource)"]
            CLI["CLI"]
            CFN_SCHED["CFN Schedule\nCustom Resource"]
            SAMPLE["Sample Schedules\nCustom Resource"]
        end

        ORCH["λ Orchestrator\n(SchedulingOrchestrator)"]
        SRH["λ Scheduling\nRequest Handler"]
        ICE["λ ICE Retry\n(Ec2ResizeHandler)"]
        TEH["λ Tag Event\nHandler"]
        SRG["λ Spoke Registration\nHandler"]

        CONFIG[("🗄️ Config Table\n(DynamoDB)")]
        REGISTRY[("🗄️ Registry Table\n(DynamoDB)")]
        STATE[("🗄️ State Table\n(DynamoDB)")]
        MW[("🗄️ MW Table\n(DynamoDB)")]

        GLOBAL_BUS["📡 Global\nEvent Bus"]
        TAG_BUS["📡 Tagging\nEvent Bus"]

        subgraph OBS["📊 Observabilidade"]
            CW["CloudWatch\n(Logs + Dashboard)"]
            SNS_T["SNS\n(Alertas)"]
            LOCAL_BUS["Local\nEvent Bus"]
        end

        SSM["🔑 SSM Parameter\nStore"]
        SCHED_ROLE["🔐 scheduler-role\n(IAM)"]
        SELF_REG_ROLE["🔐 self-registration\nrole (IAM)"]
        SSM_ROLE["🔐 ssm-role\n(IAM)"]
    end

    subgraph SPOKE["☁️ Spoke Accounts (contas gerenciadas)"]
        direction TB
        EC2["Amazon EC2"]
        RDS["Amazon RDS"]
        NEPTUNE["Amazon Neptune"]
        DOCDB["Amazon DocumentDB"]
        ASG["EC2 Auto Scaling\nGroups"]
        SSM_MW["SSM Maintenance\nWindows"]
        TAG_EVT["🏷️ Tagging Events"]
        REG_CR["Register Account\nCustom Resource"]
        SSM_PARAM["SSM Parameter\nStore"]
    end

    EB --> ORCH
    ORCH --> SRH
    ORCH --> CONFIG
    ORCH --> REGISTRY

    SRH --> MW
    SRH --> STATE
    SRH --> ICE
    SRH --> GLOBAL_BUS
    SRH --> LOCAL_BUS

    SRH --> SCHED_ROLE
    SCHED_ROLE -->|assume_role| EC2
    SCHED_ROLE -->|assume_role| RDS
    SCHED_ROLE -->|assume_role| NEPTUNE
    SCHED_ROLE -->|assume_role| DOCDB
    SCHED_ROLE -->|assume_role| ASG
    SCHED_ROLE -->|assume_role| SSM_MW

    TAG_EVT -->|EventBridge| TAG_BUS
    TAG_BUS --> TEH
    TEH --> REGISTRY
    TEH --> CONFIG

    REG_CR --> SELF_REG_ROLE
    SELF_REG_ROLE --> SRG
    SRG --> CONFIG
    SRG --> REGISTRY
    SRG --> SSM_ROLE
    SSM_ROLE --> SSM_PARAM
    SSM_PARAM --> SSM

    MAIN --> CONFIG
    SRH --> CW
    SRH --> SNS_T
```

---

## Serviços AWS Praticados

| Serviço | O que foi configurado manualmente |
|---|---|
| ![KMS](https://img.shields.io/badge/-KMS-orange?style=flat-square&logo=amazonaws) | Customer Managed Key com rotação automática e key policy customizada |
| ![DynamoDB](https://img.shields.io/badge/-DynamoDB-blue?style=flat-square&logo=amazondynamodb) | 4 tabelas com criptografia KMS, PITR habilitado e DynamoDB Streams |
| ![Lambda](https://img.shields.io/badge/-Lambda-orange?style=flat-square&logo=awslambda) | 15 funções Python 3.12 com roles, variáveis de ambiente e log groups |
| ![EventBridge](https://img.shields.io/badge/-EventBridge-pink?style=flat-square&logo=amazonaws) | 2 event buses customizados e 4 regras (cron + tag events) |
| ![IAM](https://img.shields.io/badge/-IAM-red?style=flat-square&logo=amazonaws) | 19 roles com trust policies, incluindo cross-account via STS |
| ![SNS](https://img.shields.io/badge/-SNS-red?style=flat-square&logo=amazonaws) | Topic criptografado com topic policy restritiva (SSL-only) |
| ![SQS](https://img.shields.io/badge/-SQS-orange?style=flat-square&logo=amazonaws) | 3 filas (1 principal + 2 DLQs) com KMS e visibility timeout |
| ![CloudWatch](https://img.shields.io/badge/-CloudWatch-purple?style=flat-square&logo=amazonaws) | Log groups, subscription filters, dashboard operacional e Insights queries |
| ![STS](https://img.shields.io/badge/-STS-gray?style=flat-square&logo=amazonaws) | assume_role para execução cross-account |
| ![Step Functions](https://img.shields.io/badge/-Step%20Functions-pink?style=flat-square&logo=amazonaws) | State machine de polling assíncrono para custom resources |
| ![SSM](https://img.shields.io/badge/-SSM-green?style=flat-square&logo=amazonaws) | Parameter Store para registro de regiões gerenciadas |

---

## Estrutura do Repositório

```
.
├── README.md                               # Este arquivo
├── ANALISE_PROJETO.md                      # Análise técnica detalhada da solução
├── instance-scheduler-on-aws.template     # Template CloudFormation original (referência)
├── guide.pdf                               # Guia oficial da solução AWS
│
├── docs/
│   └── GUIA_EXECUCAO_MANUAL.md            # 15 etapas de criação manual dos recursos
│
└── evidencias/
    ├── README.md                           # Guia de organização das evidências
    ├── 01-kms/
    ├── 02-dynamodb/
    ├── 03-cloudwatch-logs/
    ├── 04-sns/
    ├── 05-sqs/
    ├── 06-eventbridge/
    ├── 07-iam/
    ├── 08-lambda/
    ├── 09-eventbridge-rules/
    ├── 10-stepfunctions/
    ├── 11-inicializacao/
    ├── 12-dashboard/
    └── 13-validacao/
```

> O código-fonte original não está incluído neste repositório.
> Acesse em: [github.com/aws-solutions-library-samples/instance-scheduler-on-aws](https://github.com/aws-solutions-library-samples/instance-scheduler-on-aws)

---

## Progresso da Execução

| # | Etapa | Serviço | Status |
|---|---|---|---|
| 1 | KMS Key e Alias | KMS | ✅ Concluído |
| 2 | DynamoDB Tables (4) | DynamoDB | ✅ Concluído |
| 3 | CloudWatch Log Groups | CloudWatch | ✅ Concluído |
| 4 | SNS Topic | SNS | ✅ Concluído |
| 5 | SQS Queues | SQS | ✅ Concluído |
| 6 | EventBridge Event Buses | EventBridge | ✅ Concluído |
| 7 | IAM Roles e Policies | IAM | ⏳ Pendente |
| 8 | Lambda Functions (15) | Lambda | ⏳ Pendente |
| 9 | Lambda Permissions e ESM | Lambda / SQS | ⏳ Pendente |
| 10 | EventBridge Rules | EventBridge | ⏳ Pendente |
| 11 | CloudWatch Subscription Filters | CloudWatch | ⏳ Pendente |
| 12 | Step Functions State Machine | Step Functions | ⏳ Pendente |
| 13 | Inicialização via CloudShell | Lambda / DynamoDB | ⏳ Pendente |
| 14 | CloudWatch Dashboard | CloudWatch | ⏳ Pendente |
| 15 | Validação end-to-end com EC2 | EC2 | ⏳ Pendente |

---

## Aprendizados

> *Seção preenchida ao longo da execução*

- [ ] Como uma Customer Managed Key (KMS) centraliza criptografia em múltiplos serviços
- [ ] Diferença entre Partition Key simples e composta no DynamoDB
- [ ] Como IAM Roles cross-account funcionam via STS assume_role
- [ ] Como EventBridge cron expressions disparam Lambdas em intervalos fixos
- [ ] Como o padrão Hub-and-Spoke escala o gerenciamento para múltiplas contas AWS
- [ ] Validar o fluxo completo de agendamento via CloudWatch Logs

---

## Referências

- [AWS Instance Scheduler — Implementation Guide](https://docs.aws.amazon.com/solutions/latest/instance-scheduler-on-aws/solution-overview.html)
- [Repositório original — aws-solutions-library-samples](https://github.com/aws-solutions-library-samples/instance-scheduler-on-aws)
- [AWS CloudFormation User Guide](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html)
- [AWS IAM — Cross-account roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html)
- [Amazon EventBridge — Cron expressions](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-scheduled-rule-pattern.html)
