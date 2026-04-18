
```yaml
---
O pipeline (GitHub Actions / GitLab CI)

- name: Detectar mudanças
run: |
    git diff --name-only HEAD~1 HEAD -- resources/ > changed_files.txt
    cat changed_files.txt

- name: Apply só o que mudou
run: |
    while IFS= read -r file; do
    resource_type=$(basename "$file" .yml)   # ex: api_gateway
    ansible-playbook modules/$resource_type/playbook.yaml \
        -e "resource_file=$file"
    done < changed_files.txt
```