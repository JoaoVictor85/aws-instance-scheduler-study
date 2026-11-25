# Evidências de Execução

Pasta destinada a armazenar prints, logs e comprovações da execução manual do projeto.

## O que adicionar aqui

### Por etapa de execução
Organize os arquivos por prefixo numérico seguindo o guia de execução manual:

```
evidencias/
├── 01-kms/
│   ├── kms-key-criada.png          # Print da KMS Key criada no console
│   └── kms-alias-criado.png
├── 02-dynamodb/
│   ├── config-table.png            # Print de cada tabela criada
│   ├── state-table.png
│   ├── resource-registry.png
│   └── maintenance-window-table.png
├── 03-cloudwatch-logs/
│   └── log-groups.png
├── 04-sns/
│   └── sns-topic.png
├── 05-sqs/
│   └── filas-sqs.png
├── 06-eventbridge/
│   └── event-buses.png
├── 07-iam/
│   └── roles-criadas.png
├── 08-lambda/
│   └── funcoes-criadas.png
├── 10-eventbridge-rules/
│   └── regras-criadas.png
├── 10-execucao/
│   ├── scheduler-executando.png    # CloudWatch Logs mostrando o scheduler rodando
│   ├── instancia-parada.png        # EC2 sendo parada pelo scheduler
│   └── instancia-iniciada.png      # EC2 sendo iniciada pelo scheduler
└── 11-dashboard/
    └── cloudwatch-dashboard.png    # Dashboard operacional
```

## Dicas para boas evidências

- Sempre inclua a data/hora visível no print (canto do sistema operacional ou timestamp do console AWS)
- Capture o ARN ou nome do recurso visível na tela
- Para logs do CloudWatch, exporte como JSON quando possível
- Prints do CloudWatch Logs mostrando o scheduler atuando sobre instâncias são as melhores evidências
