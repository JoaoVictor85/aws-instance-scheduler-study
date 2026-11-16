# AWS Instance Scheduler on AWS — Análise do Projeto

## O que é esse projeto?

O **AWS Instance Scheduler on AWS** (solução oficial AWS, código SO0030, versão 3.2.15) é uma solução que **liga e desliga automaticamente instâncias EC2, bancos de dados RDS e Auto Scaling Groups (ASG) em horários programados**. O objetivo é simples: parar recursos quando não estão sendo usados para economizar dinheiro.

**Exemplo prático:** sua equipe trabalha de segunda a sexta, das 8h às 18h. Com esse scheduler, você configura que as instâncias EC2 de desenvolvimento devem estar ligadas apenas nesses horários. Fora disso, ficam desligadas. Resultado: você paga por ~10 horas/dia em vez de 24.

---

## Quais serviços AWS ele usa?

| Serviço | Para que serve aqui |
|---|---|
| **Lambda** | Cérebro da solução — executa o agendamento (Python 3.12) |
| **DynamoDB** | Banco de dados que guarda os horários e estados das instâncias |
| **EventBridge** | "Despertador" que dispara a Lambda a cada N minutos (como um cron job) |
| **IAM** | Permissões para a Lambda agir em outras contas AWS |
| **STS (assume_role)** | Permite que a Lambda "entre" em outras contas AWS |
| **KMS** | Criptografa os dados no DynamoDB e SNS |
| **SNS** | Envia notificações de erro por e-mail |
| **SQS** | Fila para requisições de resize de EC2 |
| **CloudWatch** | Logs, métricas e dashboard operacional |
| **SSM Parameter Store** | Armazena configurações das contas gerenciadas |
| **AWS Organizations** | Auto-descoberta de contas membro (opcional) |

---

## Como a arquitetura funciona?

A solução usa o padrão **Hub-and-Spoke** — pense nisso como uma roda de bicicleta: um centro (Hub) que controla tudo, e raios (Spokes) que são as outras contas gerenciadas.

```
EventBridge Cron (a cada 5 min)
         │
         ▼
 ┌────────────────────┐
 │  Orquestrador      │  ← Lambda: "quem precisa ser agendado agora?"
 │  (Lambda Python)   │    Lê schedules do DynamoDB
 └────────────────────┘
         │  invoca async para cada conta × região × serviço
         ▼
 ┌────────────────────┐
 │  Scheduler Handler │  ← Lambda: "executa o start/stop"
 │  (Lambda Python)   │    Faz assume_role na conta destino
 └────────────────────┘    Lista instâncias com a tag correta
         │                 Decide: deve estar ligada agora?
         ▼
  EC2 / RDS / ASG        ← age diretamente via SDK boto3
```

**Como o usuário configura?** Simplesmente adicionando uma tag na instância:

```
Chave: Schedule
Valor: office-hours
```

O scheduler lê essa tag e aplica o horário chamado `office-hours` que foi configurado no DynamoDB.

---

## Duas stacks CDK — Hub e Spoke

O projeto é infraestrutura como código com **AWS CDK** (TypeScript). Ele define duas stacks:

**Stack Hub** (`instance-scheduler-on-aws`) — implantada na sua conta de controle:
- Cria tudo: Lambdas, DynamoDB, EventBridge, IAM Roles, KMS, SNS, Dashboard
- Você configura aqui: qual tag usar, frequência, timezone padrão, quais regiões gerenciar

**Stack Spoke** (`instance-scheduler-on-aws-remote`) — implantada em cada conta gerenciada:
- Bem mais simples: cria uma IAM Role cross-account que confia no Hub
- Registra a conta no Hub para ser monitorada

---

## As 4 tabelas DynamoDB

| Tabela | Função |
|---|---|
| **ConfigTable** | Guarda os schedules e períodos configurados |
| **StateTable** | Rastreia o estado atual das instâncias (ligada/desligada) |
| **MaintenanceWindowTable** | Dados de Maintenance Windows do SSM |
| **ResourceRegistry** | Lista de todas as instâncias que têm a tag de agendamento |

Todas são criptografadas com KMS, têm Point-in-Time Recovery (PITR) ativado e usam pay-per-request.

---

## Linguagens e tecnologias

- **TypeScript**: infraestrutura CDK (define o que será criado na AWS)
- **Python 3.12**: código das Lambdas (lógica de negócio do agendamento)
- **aws-lambda-powertools**: logging estruturado e tracing nas Lambdas
- **pydantic**: validação dos modelos de schedule/period
- **moto + pytest**: testes unitários com mock da AWS

---

## Parâmetros importantes para o deploy

Ao fazer o deploy, você configura via parâmetros CloudFormation:

| Parâmetro | Padrão | O que faz |
|---|---|---|
| `Namespace` | `default` | Identificador único — útil para múltiplos deploys |
| `TagName` | `Schedule` | Qual tag nas instâncias o scheduler vai ler |
| `SchedulerFrequency` | `5` min | Frequência de verificação (1/2/5/10/15/30/60) |
| `DefaultTimezone` | `UTC` | Timezone padrão dos schedules |
| `Regions` | (atual) | Em quais regiões gerenciar instâncias |
| `Principals` | vazio | IDs das contas spoke autorizadas |
| `UsingAWSOrganizations` | `No` | Auto-registro de contas via Organizations |
| `CreateRdsSnapshot` | `No` | Faz snapshot antes de parar RDS |
| `EnableSSMMaintenanceWindows` | `No` | Respeita janelas de manutenção SSM |

---

## Resumo em uma frase

É um sistema event-driven serverless que, a cada N minutos, verifica tags em instâncias EC2/RDS/ASG em uma ou várias contas AWS e decide se cada recurso deve estar ligado ou desligado naquele momento — tudo configurável via DynamoDB e implantado com CDK.
