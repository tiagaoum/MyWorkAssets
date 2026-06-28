# Playbook: AWS SQS KMS Remediation

Resolve findings de cyber score que apontam filas SQS da AWS sem criptografia KMS.
Aplica as correções em repositórios Terraform e CloudFormation e entrega um commit + pull request.

---

## Quando usar este playbook

Use este playbook quando:
- Existir um finding de segurança (Security Hub, Prowler, Prisma Cloud ou similar) apontando filas SQS sem server-side encryption (SSE)
- O repositório contiver arquivos Terraform (`.tf`) ou CloudFormation (`.yaml`, `.yml`, `.json`) com recursos SQS

---

## Passo 1 — Identificar o escopo

Antes de editar qualquer arquivo, execute os comandos abaixo para mapear todas as filas não conformes.

**Terraform:**
```bash
grep -rn "resource \"aws_sqs_queue\"" . --include="*.tf"
```

**CloudFormation:**
```bash
grep -rn "AWS::SQS::Queue" . --include="*.yaml" --include="*.yml" --include="*.json"
```

Depois, use o script Python abaixo para listar somente as filas **sem** KMS:

```python
import re, sys, os, json
from pathlib import Path

root = Path(".")

# Terraform
for tf in root.rglob("*.tf"):
    content = tf.read_text()
    for m in re.finditer(
        r'resource\s+"aws_sqs_queue"\s+"(\w+)"\s*\{([^}]*(?:\{[^}]*\}[^}]*)*)\}',
        content, re.DOTALL
    ):
        name, body = m.group(1), m.group(2)
        if "kms_master_key_id" not in body:
            print(f"[TERRAFORM] NON-COMPLIANT: {tf} :: aws_sqs_queue.{name}")

# CloudFormation
import yaml
for cfn in list(root.rglob("*.yaml")) + list(root.rglob("*.yml")):
    try:
        tpl = yaml.safe_load(cfn.read_text())
        for name, res in (tpl or {}).get("Resources", {}).items():
            if res.get("Type") == "AWS::SQS::Queue":
                if "KmsMasterKeyId" not in res.get("Properties", {}):
                    print(f"[CFN] NON-COMPLIANT: {cfn} :: {name}")
    except Exception:
        pass
```

Reporte ao usuário quantas filas foram encontradas antes de prosseguir.

---

## Passo 2 — Definir a chave KMS

Pergunte ao usuário qual chave usar, ou assuma o padrão se não especificado:

| Opção | Valor | Quando usar |
|-------|-------|-------------|
| **Padrão** | `alias/aws/sqs` | Correção rápida, sem gestão de chave adicional |
| CMK existente | ARN ou alias fornecido pelo usuário | Organização já tem uma CMK para SQS |
| Nova CMK | Criar recurso `aws_kms_key` / `AWS::KMS::Key` | Requisito de chave gerenciada pelo cliente (BYOK) |

---

## Passo 3 — Aplicar as correções

### Terraform

Adicione `kms_master_key_id` dentro de cada `aws_sqs_queue` não conforme:

```hcl
# ANTES
resource "aws_sqs_queue" "orders" {
  name                       = "orders-queue"
  visibility_timeout_seconds = 30
}

# DEPOIS
resource "aws_sqs_queue" "orders" {
  name                       = "orders-queue"
  visibility_timeout_seconds = 30
  kms_master_key_id          = "alias/aws/sqs"
}
```

Se o repositório tiver múltiplas filas, prefira extrair a chave para uma variável:

```hcl
variable "sqs_kms_key_id" {
  description = "KMS key ID or alias for SQS server-side encryption"
  type        = string
  default     = "alias/aws/sqs"
}

resource "aws_sqs_queue" "orders" {
  name              = "orders-queue"
  kms_master_key_id = var.sqs_kms_key_id
}
```

Após editar, formate o código:
```bash
terraform fmt -recursive .
```

### CloudFormation (YAML)

Adicione `KmsMasterKeyId` nas `Properties` de cada `AWS::SQS::Queue`:

```yaml
# ANTES
Resources:
  OrdersQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: orders-queue

# DEPOIS
Resources:
  OrdersQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: orders-queue
      KmsMasterKeyId: alias/aws/sqs
```

### CloudFormation (JSON)

```json
{
  "Resources": {
    "OrdersQueue": {
      "Type": "AWS::SQS::Queue",
      "Properties": {
        "QueueName": "orders-queue",
        "KmsMasterKeyId": "alias/aws/sqs"
      }
    }
  }
}
```

### Regras para todos os casos

- Nunca remover atributos existentes da fila — apenas adicionar o campo de KMS
- Filas FIFO (sufixo `.fifo` ou `FifoQueue: true`) recebem o mesmo tratamento
- Dead Letter Queues (DLQ) também são filas SQS — aplicar a mesma correção
- Se a fila já tiver `kms_master_key_id = ""` ou `KmsMasterKeyId: ""`, substituir pelo valor correto

---

## Passo 4 — Validar

Confirme que não restou nenhuma fila sem KMS:

```bash
# Terraform — não deve retornar nenhuma linha sem kms_master_key_id
grep -rn "aws_sqs_queue" . --include="*.tf" -A 20 | grep -v "kms_master_key_id"

# CloudFormation — não deve retornar nenhum recurso SQS sem KmsMasterKeyId
grep -rn "AWS::SQS::Queue" . --include="*.yaml" --include="*.yml" -A 10 | grep -v "KmsMasterKeyId"
```

Se `cfn-lint` estiver disponível:
```bash
cfn-lint <template.yaml>
```

---

## Passo 5 — Commit e Pull Request

1. Crie uma branch dedicada:
```bash
git checkout -b fix/sqs-kms-encryption
```

2. Adicione apenas os arquivos alterados (nunca `git add .` sem revisar):
```bash
git add <lista dos arquivos modificados>
```

3. Crie o commit:
```bash
git commit -m "fix(security): enable KMS encryption on all SQS queues

Adds server-side encryption (SSE) via KMS to SQS queues flagged
as non-compliant by the security score tool.

Key used: alias/aws/sqs
Affected resources:
- <nome do recurso> in <arquivo>"
```

4. Publique a branch e abra o PR:
```bash
git push -u origin fix/sqs-kms-encryption

gh pr create \
  --title "fix(security): enable KMS encryption on all SQS queues" \
  --body "## Summary

- Adds KMS server-side encryption to all SQS queues missing it
- Key: alias/aws/sqs (AWS managed)

## Security finding

SQS queues without KMS are flagged as HIGH by AWS Security Hub and tools
like Prowler and Prisma Cloud.

## Test plan
- [ ] terraform plan shows no unintended changes
- [ ] Security score re-scan shows finding as resolved
- [ ] Queue send/receive functionality unchanged"
```

Se `gh` não estiver disponível, informe ao usuário o nome da branch e a descrição sugerida para o PR.

---

## Boilerplate: Nova CMK (Terraform)

Use quando o usuário exigir chave gerenciada pelo cliente:

```hcl
resource "aws_kms_key" "sqs" {
  description             = "KMS key for SQS server-side encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}

resource "aws_kms_alias" "sqs" {
  name          = "alias/myapp-sqs"
  target_key_id = aws_kms_key.sqs.key_id
}

resource "aws_sqs_queue" "example" {
  name              = "example-queue"
  kms_master_key_id = aws_kms_alias.sqs.name
}
```

## Boilerplate: Nova CMK (CloudFormation)

```yaml
Resources:
  SqsKmsKey:
    Type: AWS::KMS::Key
    Properties:
      Description: KMS key for SQS server-side encryption
      EnableKeyRotation: true
      PendingWindowInDays: 30
      KeyPolicy:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              AWS: !Sub "arn:aws:iam::${AWS::AccountId}:root"
            Action: "kms:*"
            Resource: "*"

  SqsKmsAlias:
    Type: AWS::KMS::Alias
    Properties:
      AliasName: alias/myapp-sqs
      TargetKeyId: !Ref SqsKmsKey

  ExampleQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: example-queue
      KmsMasterKeyId: !Ref SqsKmsKey
```

---

## Observações finais

- **IAM:** qualquer producer/consumer da fila precisará de `kms:GenerateDataKey` e `kms:Decrypt` na chave utilizada. Mencionar isso na descrição do PR
- **Custo:** KMS cobra por chamada de API (~$0,03 por 10 mil requests). Para filas de alto volume, considerar `kms_data_key_reuse_period_seconds: 3600`
- **CDK:** se o template for gerado por CDK, a correção deve ser feita no código CDK (não no template sintetizado) usando `QueueEncryption.KMS_MANAGED`
