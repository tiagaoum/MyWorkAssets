# Skills

Coleção de automações reutilizáveis para agentes de IA. Cada skill existe em dois formatos:

| Formato | Arquivo | Agente |
|---------|---------|--------|
| `.skill` | Zip com `SKILL.md` + referências | **Claude Code** |
| `.playbook.md` | Markdown com instruções diretas | **Devin** (e outros agentes) |

---

## Índice

- [aws-sqs-kms-remediation](#aws-sqs-kms-remediation)

---

## aws-sqs-kms-remediation

Resolve o finding de cyber score que aponta filas SQS da AWS sem criptografia KMS.
Detecta recursos não conformes em repositórios Terraform e CloudFormation, aplica as correções e entrega um commit + pull request.

### Arquivos

```
skills/
├── aws-sqs-kms-remediation.skill          # Claude Code
└── aws-sqs-kms-remediation.playbook.md    # Devin
```

---

### Uso com Claude Code

**Instalação:**

Copie `aws-sqs-kms-remediation.skill` para o projeto e rode no Claude Code:

```
/install-skill aws-sqs-kms-remediation.skill
```

Ou instale manualmente:

1. Extraia o `.skill` (é um zip) em:
   ```
   ~/.claude/plugins/cache/claude-plugins-official/aws-sqs-kms-remediation/1.0.0/skills/
   ```
2. Adicione em `~/.claude/plugins/installed_plugins.json`:
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

**Como acionar:**

A skill dispara automaticamente ao mencionar qualquer termo relacionado no contexto de um repositório IaC:

```
Tenho um repositório Terraform com filas SQS sem KMS. Pode remediar e abrir um PR?
```

---

### Uso com Devin

**Instalação:**

1. No Devin, acesse **Knowledge → Playbooks → New Playbook**
2. Cole o conteúdo de `aws-sqs-kms-remediation.playbook.md`
3. Salve com o nome `AWS SQS KMS Remediation`

**Como acionar:**

Referencie o playbook explicitamente ao iniciar uma sessão:

```
Use the playbook "AWS SQS KMS Remediation" to fix all SQS queues
without KMS encryption in this repository and open a PR.
```

---

### Estratégias de chave KMS

| Estratégia | Quando usar | Como especificar |
|------------|-------------|-----------------|
| AWS managed (`alias/aws/sqs`) | Correção rápida, sem gestão adicional | Não especificar nada (padrão) |
| CMK existente | Organização já tem uma CMK para SQS | "usa a chave `alias/minha-chave`" |
| Nova CMK | Requisito BYOK / controle total | "cria uma nova CMK para as filas" |

### Requisitos

- `git` no PATH
- `gh` (GitHub CLI) para criação automática do PR — sem ele, o agente commita e instrui a abrir o PR manualmente
- Terraform (`.tf`) ou CloudFormation (`.yaml`/`.yml`/`.json`) no repositório
