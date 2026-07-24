# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## O que é este projeto

POC em desenvolvimento. O objetivo é um sistema de deploy de infraestrutura **Azure** (via ARM Templates) **orientado a mudanças**: só aplica os recursos que foram modificados no git, em vez de redeployar tudo a cada execução.

A role `roles/azure/` já tem uma primeira implementação (resource groups, storage accounts, network security groups), mas hoje ela agrupa todas as instâncias de um tipo num único deployment ARM. O trabalho em andamento é migrar esse padrão para a granularidade por instância descrita abaixo.

---

## O que implementar primeiro

1. **`split_interface.yaml`** — playbook Ansible que lê `interface.yml` e gera um arquivo por instância de recurso em `resources/`. Basear no `../public-code/interface-modular/playbook.yaml`, que já faz o split por chave de topo (a lógica de split independe de cloud provider).

2. **`modules/{resource_groups,storage_accounts,network_security_groups}/`** — migrar as tasks e templates hoje em `roles/azure/tasks/` e `roles/azure/template/` para módulos independentes, um deployment ARM por instância. Cada módulo recebe `resource_file` como variável extra (`-e`) e deriva o `deployment_name` do nome do arquivo.

---

## Arquitetura

```
interface.yml
    ↓ split_interface.yaml
resources/{tipo}_{chave}.yml      ← um arquivo por instância, versionado no git
    ↓ git diff no pipeline
modules/{tipo}/playbook.yaml      ← executado só para os arquivos que mudaram
    ↓
deployment ARM independente por instância (resource group + azure_rm_deployment)
```

**Princípio:** granularidade por instância, não por tipo. `storage_accounts_stpocansible001.yml` e `storage_accounts_stpocansible002.yml` são deployments ARM separados. Mudar um não toca o outro.

---

## Convenções

**Nomes de arquivo em `resources/`:** `{tipo}_{chave}.yml` — ex: `storage_accounts_stpocansible001.yml`

**Deployment ARM:** mesmo nome com underscores trocados por hífens — ex: `storage-accounts-stpocansible001`

**Derivar tipo do arquivo no shell:**
```bash
filename=$(basename "$file" .yml)                        # storage_accounts_stpocansible001
resource_type=$(echo "$filename" | sed 's/_[^_]*$//')   # storage_accounts
deployment_name=$(echo "$filename" | tr '_' '-')        # storage-accounts-stpocansible001
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
  deployment_name: "{{ (resource_file | basename | splitext)[0] | replace('_', '-') }}"

tasks:
  - set_fact:
      resource: "{{ lookup('file', resource_file) | from_yaml }}"
  - template:
      src: template.j2
      dest: "/tmp/{{ deployment_name }}.json"
  - azure.azcollection.azure_rm_deployment:
      resource_group: "{{ (resource.values() | list | first).properties.resource_group }}"
      location: "{{ (resource.values() | list | first).properties.location }}"
      name: "{{ deployment_name }}"
      state: "{{ stack_state | default('present') }}"
      template: "{{ lookup('file', '/tmp/' ~ deployment_name ~ '.json') | from_json }}"
```

Resource groups são caso especial: não usam ARM template, usam o módulo nativo `azure_rm_resourcegroup` diretamente (ver `roles/azure/tasks/create_resource_groups.yml`).

---

## Referências externas

| O quê | Onde |
|---|---|
| Implementação atual (base a migrar) | `roles/azure/tasks/`, `roles/azure/template/` |
| Split de interface.yml (referência, agnóstico de cloud) | `../public-code/interface-modular/playbook.yaml` |
