# Skills

Coleção de skills customizadas para Claude Code. Cada arquivo `.skill` é portável e pode ser instalado em qualquer projeto.

---

## Índice

- [aws-sqs-kms-remediation](#aws-sqs-kms-remediation)

---

## aws-sqs-kms-remediation

**Arquivo:** `aws-sqs-kms-remediation.skill`

Resolve o finding de cyber score que aponta filas SQS da AWS sem criptografia KMS. Detecta recursos não conformes em repositórios Terraform e CloudFormation, aplica as correções necessárias e entrega um commit + pull request pronto.

### O que a skill faz

1. Varre o repositório em busca de recursos `aws_sqs_queue` (Terraform) ou `AWS::SQS::Queue` (CloudFormation) sem `kms_master_key_id` / `KmsMasterKeyId`
2. Reporta quantas filas estão não conformes antes de alterar qualquer arquivo
3. Aplica a chave KMS em todos os recursos afetados — incluindo DLQs e filas FIFO
4. Roda `terraform fmt` ou `cfn-lint` se disponíveis no ambiente
5. Cria a branch `fix/sqs-kms-encryption`, commita apenas os arquivos alterados e abre o PR via `gh`

### Instalação

**Opção A — via Claude Code (recomendado):**

Copie o arquivo `.skill` para o projeto de destino e rode no terminal do Claude Code:

```
/install-skill aws-sqs-kms-remediation.skill
```

**Opção B — manual:**

1. Extraia o `.skill` (é um zip) dentro de:
   ```
   ~/.claude/plugins/cache/claude-plugins-official/aws-sqs-kms-remediation/1.0.0/skills/
   ```
2. Adicione a entrada abaixo em `~/.claude/plugins/installed_plugins.json`, dentro do objeto `"plugins"`:
   ```json
   "aws-sqs-kms-remediation@claude-plugins-official": [
     {
       "scope": "user",
       "installPath": "~/.claude/plugins/cache/claude-plugins-official/aws-sqs-kms-remediation/1.0.0",
       "version": "1.0.0",
       "installedAt": "2026-06-28T00:00:00.000Z",
       "lastUpdated": "2026-06-28T00:00:00.000Z"
     }
   ]
   ```
3. Reinicie o Claude Code.

### Como usar

A skill dispara automaticamente quando você menciona qualquer um dos termos abaixo em uma conversa com Claude Code dentro de um repositório IaC:

| Português | Inglês |
|-----------|--------|
| "SQS sem KMS" | "SQS without KMS" |
| "remediar SQS" | "fix SQS encryption" |
| "corrigir cyber score SQS" | "SQS KMS finding" |
| "regularizar SQS" | "SQS security score" |
| "SQS não tem KMS" | "enable KMS on SQS" |

**Exemplo de prompt:**

```
Tenho um repositório Terraform aqui. Preciso resolver o finding de cyber score
das filas SQS que não possuem KMS. Pode remediar e abrir um PR?
```

### Estratégias de chave KMS

| Estratégia | Quando usar | Como especificar no prompt |
|------------|-------------|---------------------------|
| AWS managed (`alias/aws/sqs`) | Correção rápida, sem custo de gestão de chave | "usa a chave gerenciada da AWS" ou não especificar nada (padrão) |
| CMK existente | Organização já tem uma CMK para SQS | "usa a chave `alias/minha-chave`" ou "usa o ARN `arn:aws:kms:...`" |
| Nova CMK | Requisito de BYOK / controle total do ciclo de vida | "cria uma nova CMK para as filas" |

### Requisitos

- `git` disponível no PATH
- `gh` (GitHub CLI) para criação automática do PR — sem ele, a skill commita e instrui a abrir o PR manualmente
- Terraform ou arquivos CloudFormation no repositório atual

### Conteúdo do arquivo .skill

```
aws-sqs-kms-remediation/
├── SKILL.md                        # Fluxo principal (5 passos)
└── references/
    ├── terraform.md                # Padrões de patch para HCL
    ├── cloudformation.md           # Padrões para YAML/JSON CFN
    └── key-strategy.md             # Decisão de chave + boilerplate CMK
```
