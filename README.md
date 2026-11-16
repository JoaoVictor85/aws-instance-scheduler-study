# AWS Instance Scheduler — Projeto de Estudo

Estudo prático de implantação manual da solução oficial da AWS **Instance Scheduler on AWS** (SO0030 v3.2.15) em uma conta Sandbox, com o objetivo de entender os serviços envolvidos e demonstrar capacidade técnica de operação de recursos AWS.

---

## Sobre o Projeto Original

O **Instance Scheduler on AWS** é uma solução open-source mantida pela AWS que automatiza o **start e stop de instâncias EC2, bancos de dados RDS e Auto Scaling Groups** com base em agendamentos configuráveis. O principal objetivo é reduzir custos parando recursos fora do horário de uso.

- Repositório oficial: [aws-solutions-library-samples/instance-scheduler-on-aws](https://github.com/aws-solutions-library-samples/instance-scheduler-on-aws)
- Documentação AWS: [Instance Scheduler on AWS](https://aws.amazon.com/solutions/implementations/instance-scheduler-on-aws/)
- Versão estudada: **v3.2.15**

---

## Objetivo do Estudo

Em vez de usar o deploy automático via CDK ou CloudFormation com um clique, optei por **criar todos os recursos manualmente** seguindo o template CloudFormation como guia. Isso permite compreender cada serviço, sua configuração e as dependências entre eles.

**Serviços praticados neste projeto:**

| Serviço | O que foi praticado |
|---|---|
| AWS KMS | Criação de CMK com rotação automática e política de chave |
| Amazon DynamoDB | Criação de tabelas com criptografia KMS, PITR e streams |
| AWS Lambda | Criação de funções Python 3.12, IAM roles, variáveis de ambiente |
| Amazon EventBridge | Event buses, regras de agendamento (cron) e regras de eventos de tagging |
| AWS IAM | Roles com trust policies, inline policies e managed policies cross-account |
| Amazon SNS | Topic criptografado com topic policy restritiva |
| Amazon SQS | Filas com DLQ, criptografia KMS e visibility timeout |
| Amazon CloudWatch | Log groups, subscription filters, queries e dashboard operacional |
| AWS STS | assume_role para execução cross-account |
| AWS Step Functions | State machine de waiter para custom resources assíncronos |
| AWS SSM Parameter Store | Parâmetros de configuração de spoke accounts |

---

## Arquitetura da Solução

A solução utiliza o padrão **Hub-and-Spoke**:

```
EventBridge Cron (a cada N minutos)
         │
         ▼
 ┌────────────────────────┐
 │  SchedulingOrchestrator│  Lê schedules do DynamoDB
 │  (Lambda Python 3.12)  │  Consulta ResourceRegistry
 └────────────────────────┘
         │  Invoca async por conta × região × serviço
         ▼
 ┌────────────────────────┐
 │  SchedulingRequest     │  assume_role na conta destino
 │  Handler (Lambda)      │  Lista instâncias com a tag
 └────────────────────────┘  Decide start/stop → executa
         │
         ▼
   EC2 / RDS / ASG
```

**Como associar uma instância a um schedule:**
Adicione a tag abaixo na instância EC2 ou banco RDS:
```
Chave:  Schedule
Valor:  nome-do-schedule-configurado
```

---

## Estrutura do Repositório

```
.
├── README.md                    # Este arquivo
├── ANALISE_PROJETO.md           # Análise técnica detalhada da solução
├── instance-scheduler-on-aws.template  # Template CloudFormation original (referência)
├── guide.pdf                    # Guia oficial da solução
│
├── docs/                        # Documentação complementar produzida durante o estudo
│
└── evidencias/                  # Prints e logs comprovando a execução
    └── README.md                # Guia de organização das evidências
```

> O código-fonte original não está incluído neste repositório.
> Acesse em: https://github.com/aws-solutions-library-samples/instance-scheduler-on-aws

---

## Como Reproduzir

Consulte o arquivo [`docs/GUIA_EXECUCAO_MANUAL.md`](docs/GUIA_EXECUCAO_MANUAL.md) para o passo a passo completo de criação manual de todos os recursos na ordem correta de dependências.

**Pré-requisitos:**
- Conta AWS (Sandbox ou conta de estudo)
- Usuário/Role IAM com permissões de administrador
- Acesso ao console AWS ou AWS CLI configurado

---

## Aprendizados

> *Seção a ser preenchida ao longo da execução*

- [ ] Entender como IAM Roles cross-account funcionam na prática
- [ ] Configurar EventBridge cron expressions
- [ ] Validar o fluxo completo de agendamento via CloudWatch Logs
- [ ] Observar o scheduler parando e iniciando instâncias EC2

---

## Referências

- [AWS Instance Scheduler — Implementation Guide](https://docs.aws.amazon.com/solutions/latest/instance-scheduler-on-aws/solution-overview.html)
- [GitHub — aws-solutions-library-samples/instance-scheduler-on-aws](https://github.com/aws-solutions-library-samples/instance-scheduler-on-aws)
- [AWS CDK Documentation](https://docs.aws.amazon.com/cdk/v2/guide/home.html)
- [AWS CloudFormation User Guide](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html)
