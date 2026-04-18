# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## O que é este projeto

POC em desenvolvimento. O objetivo é um sistema de deploy de infraestrutura AWS **orientado a mudanças**: só aplica os recursos que foram modificados no git, em vez de redeployar tudo a cada execução.

Este repositório ainda não tem código implementado — apenas a arquitetura está definida. O ponto de partida para implementação está abaixo.

---

## O que implementar primeiro

1. **`split_interface.yaml`** — playbook Ansible que lê `interface.yml` e gera um arquivo por instância de recurso em `resources/`. Basear no `../public-code/interface-modular/playbook.yaml`, que já faz o split por chave de topo.

2. **`modules/api_gateway/`** — primeiro módulo a implementar. Adaptar o template Jinja2 de `../public-code/develop-local/create_api_gateway.j2` e o playbook de `../public-code/develop-local/playbook.yaml`, ajustando para receber `resource_file` como variável extra (`-e`) e derivar o `stack_name` do nome do arquivo.

---

## Arquitetura

```
interface.yml
    ↓ split_interface.yaml
resources/{tipo}_{chave}.yml      ← um arquivo por instância, versionado no git
    ↓ git diff no pipeline
modules/{tipo}/playbook.yaml      ← executado só para os arquivos que mudaram
    ↓
stack CFN independente por instância
```

**Princípio:** granularidade por instância, não por tipo. `api_gateway_gateway1.yml` e `api_gateway_gateway2.yml` são stacks CFN separados. Mudar um não toca o outro.

---

## Convenções

**Nomes de arquivo em `resources/`:** `{tipo}_{chave}.yml` — ex: `api_gateway_gateway1.yml`

**Stack CFN:** mesmo nome com underscores trocados por hífens — ex: `api-gateway-gateway1`

**Derivar tipo do arquivo no shell:**
```bash
filename=$(basename "$file" .yml)                        # api_gateway_gateway1
resource_type=$(echo "$filename" | sed 's/_[^_]*$//')   # api_gateway
stack_name=$(echo "$filename" | tr '_' '-')             # api-gateway-gateway1
```

---

## Como o pipeline detecta mudanças

```bash
# Apply (adicionados ou modificados)
git diff --name-only --diff-filter=AM HEAD~1 HEAD -- resources/

# Destroy (deletados)
git diff --name-only --diff-filter=D HEAD~1 HEAD -- resources/

# Bootstrap (sem commit anterior)
if ! git rev-parse HEAD~1 >/dev/null 2>&1; then
  ls resources/*.yml > changed_files.txt
fi
```

Cada módulo deve aceitar `stack_state=absent` para o caso de destroy.

---

## Padrão de playbook por módulo

```yaml
vars:
  stack_name: "{{ (resource_file | basename | splitext)[0] | replace('_', '-') }}"

tasks:
  - set_fact:
      resource: "{{ lookup('file', resource_file) | from_yaml }}"
  - template:
      src: template.j2
      dest: "/tmp/{{ stack_name }}.json"
  - amazon.aws.cloudformation:
      stack_name: "{{ stack_name }}"
      state: "{{ stack_state | default('present') }}"
      region: us-east-1
      template: "/tmp/{{ stack_name }}.json"
```

---

## Referências externas

| O quê | Onde |
|---|---|
| Template Jinja2 de API Gateway (referência) | `../public-code/develop-local/create_api_gateway.j2` |
| Playbook de deploy atual (referência) | `../public-code/develop-local/playbook.yaml` |
| Split de interface.yml (referência) | `../public-code/interface-modular/playbook.yaml` |
| Templates CFN prontos | `../public-code/cfn/api-gateway/` |
