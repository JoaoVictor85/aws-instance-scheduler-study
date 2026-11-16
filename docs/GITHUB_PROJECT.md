# GitHub Project Board — Estrutura e Configuração

## Criar o Project

Acesse o perfil do GitHub e vá em:
```
https://github.com/JoaoVictor85 → Projects → New Project
```

| Campo | Valor |
|---|---|
| Template | **Board** |
| Nome | `AWS Instance Scheduler — Estudo Prático` |
| Descrição | `Acompanhamento da execução manual do Instance Scheduler on AWS em conta Sandbox` |

Após criar, vincule o project ao repositório:
```
Project Settings → Linked repositories → aws-instance-scheduler-study
```

---

## Colunas do Board

Configure as seguintes colunas **nesta ordem**, renomeando ou criando conforme necessário:

| Coluna | Propósito |
|---|---|
| `📋 Backlog` | Todas as issues criadas ainda não iniciadas |
| `🔍 Estudando` | Lendo documentação e entendendo o serviço antes de executar |
| `⚙️ Executando` | Recurso sendo criado no console AWS neste momento |
| `📸 Evidenciando` | Execução feita, capturando prints e registrando aprendizados |
| `✅ Concluído` | Issue fechada com evidências e aprendizados documentados |
| `🚫 Bloqueado` | Parado por erro, dúvida ou dependência não resolvida |

---

## Campos Customizados

Dentro do project, adicione os seguintes campos customizados em **Settings → Fields**:

| Campo | Tipo | Opções / Descrição |
|---|---|---|
| `Serviço AWS` | Single select | KMS, DynamoDB, CloudWatch, SNS, SQS, EventBridge, IAM, Lambda, Step Functions |
| `Fase` | Single select | Fase 1, Fase 2, Fase 3, Fase 4, Fase 5, Fase 6, Fase 7, Fase 8 |
| `Dificuldade` | Single select | Fácil, Médio, Difícil |
| `Evidência` | Text | Link ou nome do arquivo de evidência |

---

## Automações

Configure as automações em **Settings → Workflows** para reduzir trabalho manual:

| Gatilho | Ação automática |
|---|---|
| Issue criada | Mover para **Backlog** |
| Issue reaberta | Mover para **Backlog** |
| Issue fechada | Mover para **Concluído** |
| Label `bloqueado` adicionada | Mover para **Bloqueado** |
| Label `em andamento` adicionada | Mover para **Executando** |
| Pull request mergeado | Mover issue vinculada para **Concluído** |

---

## Visões (Views)

Crie as seguintes visões adicionais além do Board padrão:

### Visão 1 — Por Fase (Table)
- Tipo: **Table**
- Agrupar por: campo `Fase`
- Colunas visíveis: Título, Serviço AWS, Status, Dificuldade

### Visão 2 — Roadmap
- Tipo: **Roadmap**
- Usar datas das Milestones como referência de prazo

### Visão 3 — Backlog priorizado
- Tipo: **Table**
- Filtro: `Status != Concluído`
- Ordenar por: campo `Fase` (crescente)

---

## Fluxo de trabalho recomendado

Siga este fluxo para cada etapa do projeto:

```
1. Abrir a issue correspondente à etapa
      ↓
2. Mover para "Estudando" — ler o contexto no guia e entender o serviço
      ↓
3. Mover para "Executando" — criar os recursos no console AWS
      ↓
4. Mover para "Evidenciando" — capturar prints, salvar em evidencias/
      ↓
5. Preencher a seção "Aprendizados" na issue
      ↓
6. Fechar a issue → move automaticamente para "Concluído"
```

---

## Resultado esperado ao final do projeto

O board completo com todas as issues em **Concluído** serve como prova visual de execução para o portfólio. Qualquer recrutador ou avaliador técnico consegue:

- Ver cada serviço AWS que você configurou
- Ler os aprendizados que você documentou
- Verificar as evidências (prints) linkadas em cada issue
- Entender a sequência de dependências que você respeitou
