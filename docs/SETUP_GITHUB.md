# Setup do Repositório GitHub

## 1. Criar o repositório no GitHub

Acesse [github.com/new](https://github.com/new) e preencha:

| Campo | Valor |
|---|---|
| Repository name | `aws-instance-scheduler-study` |
| Description | `Estudo prático de implantação manual do AWS Instance Scheduler on AWS (SO0030 v3.2.15) em conta Sandbox. Portfólio de aprendizado AWS.` |
| Visibility | **Public** (portfólio) |
| Initialize repository | **Não marque nenhuma opção** — o repositório deve ser criado vazio |

---

## 2. Inicializar o repositório local e fazer o primeiro push

Abra um terminal na pasta do projeto e execute os comandos abaixo **em ordem**:

```bash
# Entrar na pasta do projeto
cd "c:\Users\VICTO\Documents\Pessoal\Projetos\instance_scheduler_on_aws"

# Inicializar o repositório git local
git init

# Adicionar os arquivos ao staging (exceto o que está no .gitignore)
git add .

# Criar o primeiro commit
git commit -m "chore: estrutura inicial do projeto de estudo

- README.md com contexto do projeto e portfólio
- ANALISE_PROJETO.md com análise técnica detalhada
- docs/GUIA_EXECUCAO_MANUAL.md com 15 etapas de execução
- docs/SETUP_GITHUB.md com instruções de setup
- evidencias/ com guia de organização
- .github/ISSUE_TEMPLATE com templates de issue
- instance-scheduler-on-aws.template como referência CloudFormation"

# Definir a branch principal como main
git branch -M main

# Conectar ao repositório remoto no GitHub
git remote add origin https://github.com/JoaoVictor85/aws-instance-scheduler-study.git

# Enviar para o GitHub
git push -u origin main
```

---

## 3. Verificar o resultado

Após o push, acesse:
```
https://github.com/JoaoVictor85/aws-instance-scheduler-study
```

Confirme que os seguintes arquivos estão visíveis:
- [ ] `README.md` renderizado na página inicial
- [ ] Pasta `docs/` com os guias
- [ ] Pasta `evidencias/` com o README
- [ ] Pasta `.github/ISSUE_TEMPLATE/` com os 3 templates
- [ ] `instance-scheduler-on-aws.template`
- [ ] `ANALISE_PROJETO.md`

> A pasta `codigo_fonte/` **não deve aparecer** — está no `.gitignore`.

---

## 4. Configurar Labels no GitHub

As labels organizam visualmente as issues. Acesse:
```
https://github.com/JoaoVictor85/aws-instance-scheduler-study/labels
```

Delete as labels padrão do GitHub e crie as seguintes:

| Label | Cor | Descrição |
|---|---|---|
| `execução` | `#0075ca` | Tarefa de execução de recurso AWS |
| `em andamento` | `#e4e669` | Issue sendo trabalhada agora |
| `concluído` | `#0e8a16` | Etapa finalizada com evidências |
| `bloqueado` | `#d93f0b` | Bloqueado por erro ou dependência |
| `aprendizado` | `#7057ff` | Anotação de conceito aprendido |
| `documentação` | `#cfd3d7` | Melhoria ou adição de documentação |
| `bug` | `#ee0701` | Erro encontrado durante execução |
| `kms` | `#f9d0c4` | Relacionado ao serviço KMS |
| `dynamodb` | `#fef2c0` | Relacionado ao DynamoDB |
| `cloudwatch` | `#c5def5` | Relacionado ao CloudWatch |
| `sns` | `#bfd4f2` | Relacionado ao SNS |
| `sqs` | `#d4c5f9` | Relacionado ao SQS |
| `eventbridge` | `#e99695` | Relacionado ao EventBridge |
| `iam` | `#f9d0c4` | Relacionado ao IAM |
| `lambda` | `#fbca04` | Relacionado ao Lambda |
| `stepfunctions` | `#b60205` | Relacionado ao Step Functions |

---

## 5. Criar Milestones no GitHub

As milestones agrupam issues em fases do projeto. Acesse:
```
https://github.com/JoaoVictor85/aws-instance-scheduler-study/milestones/new
```

Crie as seguintes milestones **nesta ordem**:

| # | Título | Descrição |
|---|---|---|
| 1 | `Fase 1 — Fundação` | KMS, DynamoDB, CloudWatch Logs, SNS, SQS |
| 2 | `Fase 2 — Eventos e Mensageria` | EventBridge Event Buses |
| 3 | `Fase 3 — Identidade e Acesso` | IAM Roles e Policies |
| 4 | `Fase 4 — Computação` | Lambda Functions, Permissions e Event Source Mapping |
| 5 | `Fase 5 — Orquestração` | EventBridge Rules, Subscription Filters, Step Functions |
| 6 | `Fase 6 — Inicialização` | Custom Resources e setup inicial |
| 7 | `Fase 7 — Observabilidade` | CloudWatch Dashboard e Logs Insights Queries |
| 8 | `Fase 8 — Validação` | Teste end-to-end com EC2 |

---

## 6. Próximo passo

Com o repositório criado, labels e milestones configuradas, siga para o documento de issues:

📄 [`docs/ISSUES.md`](ISSUES.md)
