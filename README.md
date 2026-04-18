# poc-ansible

POC de deploy de infraestrutura AWS orientado a mudanças, usando Ansible + CloudFormation + Jinja2.

---

Ideias de performance para Ansible:

┌─────────────────────┬───────────────────┬────────────────────────────────────────────────────────┐
│       Técnica       │       Ganho       │                      Complexidade                      │
├─────────────────────┼───────────────────┼────────────────────────────────────────────────────────┤
│ gather_facts: false │ alto              │ já feito                                               │
├─────────────────────┼───────────────────┼────────────────────────────────────────────────────────┤
│ strategy: free      │ médio             │ baixo — tasks não esperam umas pelas outras            │
├─────────────────────┼───────────────────┼────────────────────────────────────────────────────────┤
│ async + poll        │ alto              │ médio — tasks em paralelo                              │
├─────────────────────┼───────────────────┼────────────────────────────────────────────────────────┤
│ Mitogen             │ muito alto (2-7x) │ médio — plugin externo que substitui o executor Python │
└─────────────────────┴───────────────────┴────────────────────────────────────────────────────────┘

Mitogen

┌─────────────────────────────────────────┬───────────────────────────────────────────┐
│                Vantagem                 │                Desvantagem                │
├─────────────────────────────────────────┼───────────────────────────────────────────┤
│ Transparente — não muda nenhum playbook │ Defasagem de versão com ansible-core      │
├─────────────────────────────────────────┼───────────────────────────────────────────┤
│ Ganho em todos os hosts automaticamente │ Incompatibilidade pontual com collections │
├─────────────────────────────────────────┼───────────────────────────────────────────┤
│ Ativação única no AWX                   │ async não funciona junto                  │
└─────────────────────────────────────────┴───────────────────────────────────────────┘

O risco do Mitogen é descobrir o problema tarde — funciona no teste, quebra em produção numa task específica de uma
collection.

---

Async + Poll

┌──────────────────────────────────────────────┬────────────────────────────────────────────┐
│                   Vantagem                   │                Desvantagem                 │
├──────────────────────────────────────────────┼────────────────────────────────────────────┤
│ Nativo do Ansible — zero dependência externa │ Você reescreve as tasks                    │
├──────────────────────────────────────────────┼────────────────────────────────────────────┤
│ Controle fino de quais tasks paralelizam     │ Dependências entre recursos exigem atenção │
├──────────────────────────────────────────────┼────────────────────────────────────────────┤
│ Compatível com qualquer collection           │ Código mais verboso                        │
└──────────────────────────────────────────────┴────────────────────────────────────────────┘