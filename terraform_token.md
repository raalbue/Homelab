# Terraform API Token Setup

## Ansible Playbook — `ansible/playbooks/06_create_terraform_token.yml`

```yaml
---
- name: Create Terraform API token in Proxmox
  hosts: proxmox
  gather_facts: false
  become: true

  tasks:
    - name: Create terraform@pve user
      ansible.builtin.command:
        cmd: pveum user add terraform@pve
      register: user_result
      changed_when: user_result.rc == 0
      failed_when: user_result.rc != 0 and 'already exists' not in user_result.stderr

    - name: Assign PVEAdmin role to terraform@pve
      ansible.builtin.command:
        cmd: pveum aclmod / -user terraform@pve -role PVEAdmin
      changed_when: true

    - name: Create terraform API token
      ansible.builtin.command:
        cmd: pveum user token add terraform@pve tf-token --privsep=0
      register: token_result
      changed_when: token_result.rc == 0
      failed_when: token_result.rc != 0 and 'already exists' not in token_result.stderr

    - name: Show token secret — save this to terraform.tfvars
      ansible.builtin.debug:
        msg: "{{ token_result.stdout }}"
      when: token_result.changed
```

## Run it

```bash
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/06_create_terraform_token.yml
```

## Output

The playbook prints the token secret only on first creation (`when: token_result.changed`).
Copy the secret into `terraform/terraform.tfvars`:

```hcl
proxmox_api_token_id     = "terraform@pve!tf-token"
proxmox_api_token_secret = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

## If you lose the secret

Tokens cannot be retrieved after creation. Delete and recreate:

```bash
pveum user token remove terraform@pve tf-token
pveum user token add terraform@pve tf-token --privsep=0
```
