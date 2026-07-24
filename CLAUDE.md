# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## O que é este projeto

POC em desenvolvimento. O objetivo é um sistema de deploy de infraestrutura **Azure** (via ARM Templates) **orientado a mudanças**: só aplica os recursos que foram modificados no git, em vez de redeployar tudo a cada execução.

A role `roles/azure/` já tem uma primeira implementação (resource groups, storage accounts, network security groups), mas hoje ela agrupa todas as instâncias de um tipo num único deployment ARM. O trabalho em andamento é migrar esse padrão para a granularidade por instância descrita abaixo.

---

## O que implementar primeiro

1. **`modules/{resource_groups,storage_accounts,network_security_groups}/`** — migrar as tasks e templates hoje em `roles/azure/tasks/` e `roles/azure/template/` para módulos independentes, um deployment ARM por instância. Ver "Como o pipeline detecta mudanças" abaixo pra saber que variáveis cada módulo recebe hoje.

---

## Arquitetura

```
interface.yml (working tree)   ← fonte da verdade, único arquivo de entrada
    ↓ comparado contra
interface.yml (HEAD)           ← lookup('pipe', 'git show HEAD:interface.yml')
    ↓ diff estrutural (por tipo + chave, não por linha de texto)
lista de instâncias alteradas/novas/removidas
    ↓
deployment ARM independente por instância (resource group + azure_rm_deployment)
```

**Não existe mais split em `resources/`.** Uma tentativa anterior gerava um arquivo por instância em `resources/{tipo}_{chave}.yml` pra dar granularidade ao `git diff`. Foi abandonada — arquivo extra pra manter e commitar depois de cada deploy, sem ganho real. Hoje o `playbook.yaml` compara duas versões do `interface.yml` (working tree vs HEAD) como estrutura de dados (dicts, via `from_yaml`), não como texto — então não precisa de arquivo por instância pra saber o que mudou.

**Princípio:** granularidade por instância, não por tipo. Mudar `storage_accounts.stpocansible001` não deve tocar `storage_accounts.stpocansible002` — cada instância (`tipo` + `chave` do `interface.yml`) vira um deployment ARM separado.

---

## Como o pipeline detecta mudanças (hoje, local/dev)

Implementado em `playbook.yaml`:

1. Carrega `interface.yml` do working tree (`interface_atual`) e do HEAD via `git show HEAD:interface.yml` (`interface_head`) — vazio se o arquivo ainda não existia no HEAD (bootstrap).
2. Monta a lista de instâncias (`{tipo, chave}`) de cada versão.
3. `changed_instances` = instâncias que estão no atual e (não existiam no HEAD OU o conteúdo mudou). `removed_instances` = instâncias que existiam no HEAD e sumiram do atual.
4. Deploy roda em `changed_instances` com `resource_data` = `{chave: interface_atual[tipo][chave]}`; destroy roda em `removed_instances` com `resource_data` vindo de `interface_head` e `stack_state: absent`.

**Isso é a versão local/dev** — compara working tree vs HEAD, então pega mudança não commitada. **Ainda não existe pipeline de CI neste lab.** Quando existir, a comparação deve trocar pra `HEAD~1` vs `HEAD` (dois commits, não working tree vs HEAD) — ver `ideia-de-cicd.md`. A lógica de diff estrutural (por tipo + chave) continua a mesma, só muda de onde vem cada versão do `interface.yml`.

Cada módulo/role deve aceitar `stack_state=absent` para o caso de destroy.

**Deployment ARM:** nome = `{tipo}_{chave}` com underscores trocados por hífens — ex: `storage-accounts-stpocansible001`.

---

## Padrão de playbook/task por módulo (tipo de recurso)

`playbook.yaml` chama cada módulo via `include_role: tasks_from: create_{{tipo}}`, passando `resource_data` (`{chave: {...}}`, um único item) e `deployment_name` já prontos — o módulo não lê arquivo nenhum, só usa o que recebeu:

```yaml
tasks:
  - template:
      src: template.j2
      dest: "/tmp/{{ deployment_name }}.json"
  - azure.azcollection.azure_rm_deployment:
      resource_group: "{{ (resource_data.values() | list | first).properties.resource_group }}"
      location: "{{ (resource_data.values() | list | first).properties.location }}"
      name: "{{ deployment_name }}"
      state: "{{ stack_state | default('present') }}"
      template: "{{ lookup('file', '/tmp/' ~ deployment_name ~ '.json') | from_json }}"
```

Resource groups são caso especial: não usam ARM template, usam o módulo nativo `azure_rm_resourcegroup` diretamente (ver `roles/azure/tasks/create_resource_groups.yml`).

---

## Referências externas

| O quê | Onde |
|---|---|
| Implementação atual (base a migrar pra `modules/`) | `roles/azure/tasks/`, `roles/azure/template/` |
| Notas de CI (rascunho, não implementado) | `ideia-de-cicd.md` |
