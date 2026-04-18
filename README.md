# poc-ansible

POC de deploy de infraestrutura AWS orientado a mudanças, usando Ansible + CloudFormation + Jinja2.

---

## Contexto e objetivo

O projeto [`public-code/develop-local`](../public-code/develop-local) já gera templates CloudFormation dinamicamente a partir de um `interface.yml` usando Jinja2 e Ansible. O problema atual: qualquer mudança redeploya **tudo** em um único stack.

O objetivo desta POC é tornar o deploy **orientado à mudança**: só aplicar os recursos que foram de fato modificados no git.

---

## Arquitetura planejada

```
interface.yml (monolito — editado pelo usuário)
    ↓ split (playbook/script)
resources/                    ← arquivos versionados no git, um por instância
    api_gateway_gateway1.yml
    api_gateway_gateway2.yml
    sqs_fila_pedidos.yml
    sns_topico_alertas.yml
    ↓ git diff no pipeline
changed_files = [api_gateway_gateway1.yml]
    ↓ resolve tipo do recurso e executa só o módulo correspondente
modules/api_gateway/playbook.yaml  →  stack CFN "api-gateway-gateway1"
```

**Princípio chave:** granularidade por **instância**, não por tipo. Cada instância de recurso vira um arquivo separado em `resources/` e mapeia para um stack CFN independente. Mudar `gateway1` não toca `gateway2`.

---

## Estrutura de diretórios alvo

```
poc-ansible/
├── interface.yml                  # contrato monolítico (ponto de edição do usuário)
├── split_interface.yaml           # playbook que explode interface.yml → resources/
├── resources/                     # arquivos gerados pelo split, versionados no git
│   ├── api_gateway_gateway1.yml
│   └── sqs_fila_pedidos.yml
└── modules/                       # um módulo por tipo de recurso
    ├── api_gateway/
    │   ├── template.j2            # Jinja2 → CloudFormation JSON
    │   └── playbook.yaml          # recebe resource_file como variável extra
    └── sqs/
        ├── template.j2
        └── playbook.yaml
```

---

## Convenção de nomes

| Item | Padrão | Exemplo |
|---|---|---|
| Arquivo em `resources/` | `{tipo}_{chave}.yml` | `api_gateway_gateway1.yml` |
| Stack CFN | `{tipo}-{chave}` (underscores → hífens) | `api-gateway-gateway1` |
| Módulo em `modules/` | nome do tipo de recurso | `modules/api_gateway/` |

---

## Pipeline CI/CD (GitHub Actions / GitLab CI)

### Apply — recursos modificados ou adicionados

```yaml
- name: Detectar mudanças
  run: |
    git diff --name-only --diff-filter=AM HEAD~1 HEAD -- resources/ > changed_files.txt

- name: Apply só o que mudou
  run: |
    while IFS= read -r file; do
      # extrai o tipo do recurso a partir do nome do arquivo
      # ex: resources/api_gateway_gateway1.yml → api_gateway
      filename=$(basename "$file" .yml)           # api_gateway_gateway1
      resource_type=$(echo "$filename" | sed 's/_[^_]*$//')  # api_gateway
      ansible-playbook modules/$resource_type/playbook.yaml \
        -e "resource_file=$file"
    done < changed_files.txt
```

### Destroy — recursos deletados

```yaml
- name: Detectar deleções
  run: |
    git diff --name-only --diff-filter=D HEAD~1 HEAD -- resources/ > deleted_files.txt

- name: Destruir stacks removidos
  run: |
    while IFS= read -r file; do
      filename=$(basename "$file" .yml)
      resource_type=$(echo "$filename" | sed 's/_[^_]*$//')
      stack_name=$(echo "$filename" | tr '_' '-')
      ansible-playbook modules/$resource_type/playbook.yaml \
        -e "resource_file=$file" \
        -e "stack_state=absent" \
        -e "stack_name=$stack_name"
    done < deleted_files.txt
```

### Bootstrap (primeiro commit — sem HEAD~1)

```bash
# Fallback: se não existe commit anterior, aplica tudo
if ! git rev-parse HEAD~1 >/dev/null 2>&1; then
  ls resources/*.yml > changed_files.txt
fi
```

---

## Estrutura do `interface.yml`

O contrato suporta múltiplos tipos de recurso no mesmo arquivo:

```yaml
api_gateway:
  gateways:
    gateway1:
      properties:
        gateway_name: meu-gateway
        stage_name: dev
        resources:
          - path: "/admin"
            methods:
              GET:
                integration:
                  uri: "https://backend.com/admin"

sqs:
  queues:
    fila_pedidos:
      properties:
        queue_name: fila-pedidos
        visibility_timeout: 30

sns:
  topics:
    topico_alertas:
      properties:
        topic_name: alertas-producao
```

---

## Estrutura do `playbook.yaml` de cada módulo

O playbook de cada módulo deve:
1. Receber `resource_file` como variável extra (`-e`)
2. Derivar o `stack_name` a partir do nome do arquivo
3. Renderizar o template Jinja2
4. Chamar `amazon.aws.cloudformation` com `state: "{{ stack_state | default('present') }}"`

```yaml
---
- name: Deploy {{ resource_type }}
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    stack_name: "{{ (resource_file | basename | splitext)[0] | replace('_', '-') }}"

  tasks:
    - name: Carregar resource file
      set_fact:
        resource: "{{ lookup('file', resource_file) | from_yaml }}"

    - name: Gerar CloudFormation JSON
      template:
        src: template.j2
        dest: "/tmp/{{ stack_name }}.json"

    - name: Apply CloudFormation
      amazon.aws.cloudformation:
        stack_name: "{{ stack_name }}"
        state: "{{ stack_state | default('present') }}"
        region: us-east-1
        template: "/tmp/{{ stack_name }}.json"
```

---

## Referências

- Projeto base com o template Jinja2 de API Gateway: [`public-code/develop-local`](../public-code/develop-local)
- Experimento de split do `interface.yml`: [`public-code/interface-modular`](../public-code/interface-modular)
- Template CFN de referência para API Gateway: `public-code/cfn/api-gateway/`
