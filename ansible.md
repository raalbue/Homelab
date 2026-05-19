# Ansible Proxmox Setup

Minimum professional setup for managing Proxmox infrastructure with Ansible.

## Directory Structure

```
ansible/
├── ansible.cfg
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       └── proxmox/
│           ├── vars.yml
│           └── vault.yml
├── playbooks/
│   ├── bootstrap_proxmox.yml
│   └── site.yml
└── requirements.yml
```

## 1. SSH Key Auth

```bash
# Generate a dedicated key
ssh-keygen -t ed25519 -f ~/.ssh/ansible_proxmox -C "ansible@proxmox"

# Copy to proxmox host
ssh-copy-id -i ~/.ssh/ansible_proxmox.pub root@<proxmox-ip>
```

## 2. `ansible.cfg`

```ini
[defaults]
inventory = inventory/hosts.yml
private_key_file = ~/.ssh/ansible_proxmox
host_key_checking = True
retry_files_enabled = False
vault_password_file = .vault_password

[privilege_escalation]
become = False
```

## 3. `inventory/hosts.yml`

```yaml
all:
  children:
    proxmox:
      hosts:
        pve01:
          ansible_host: 192.168.1.x
          ansible_user: root
          ansible_python_interpreter: /usr/bin/python3
```

## 4. Secrets with Ansible Vault

Create vault password file (excluded from git):

```bash
openssl rand -base64 32 > .vault_password
chmod 600 .vault_password
echo ".vault_password" >> .gitignore
```

Create encrypted vars:

```bash
ansible-vault create inventory/group_vars/proxmox/vault.yml
```

Contents of `inventory/group_vars/proxmox/vault.yml` (encrypted at rest):

```yaml
vault_proxmox_api_token_id: "ansible@pam!ansible-token"
vault_proxmox_api_token_secret: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
vault_proxmox_root_password: "changeme"
```

Reference them in `inventory/group_vars/proxmox/vars.yml` (unencrypted):

```yaml
proxmox_api_token_id: "{{ vault_proxmox_api_token_id }}"
proxmox_api_token_secret: "{{ vault_proxmox_api_token_secret }}"
```

This indirection lets you `grep` for variable usage without decrypting vault files.

## 5. `.gitignore`

```
.vault_password
*.retry
```

## 6. Proxmox API Token

Create a dedicated API token instead of using root credentials. This can be done
manually on the Proxmox host or via the bootstrap playbook below.

Manual method:

```bash
pveum user add ansible@pam
pveum aclmod / -user ansible@pam -role PVEAdmin
pveum user token add ansible@pam ansible-token
```

Store the token output in your vault file.

## 7. Bootstrap Playbook — `playbooks/bootstrap_proxmox.yml`

This playbook uses the root password (prompted interactively, never stored) to:
- Copy your SSH public key
- Create the ansible PAM user
- Assign the PVEAdmin role
- Create an API token

```yaml
---
- name: Bootstrap Proxmox for Ansible management
  hosts: proxmox
  gather_facts: false
  vars_prompt:
    - name: root_password
      prompt: "Enter Proxmox root password (one-time use)"
      private: true
  vars:
    ansible_user: root
    ansible_password: "{{ root_password }}"
    ansible_ssh_common_args: '-o PreferredAuthentications=password'
  tasks:
    - name: Copy SSH public key for ansible user
      ansible.posix.authorized_key:
        user: root
        key: "{{ lookup('file', '~/.ssh/ansible_proxmox.pub') }}"
        state: present

    - name: Create ansible PAM user
      ansible.builtin.command:
        cmd: pveum user add ansible@pam
      register: user_result
      changed_when: user_result.rc == 0
      failed_when: user_result.rc != 0 and 'already exists' not in user_result.stderr

    - name: Assign PVEAdmin role
      ansible.builtin.command:
        cmd: pveum aclmod / -user ansible@pam -role PVEAdmin

    - name: Create API token
      ansible.builtin.command:
        cmd: pveum user token add ansible@pam ansible-token
      register: token_result
      changed_when: token_result.rc == 0
      failed_when: token_result.rc != 0 and 'already exists' not in token_result.stderr

    - name: Show token (save this to vault)
      ansible.builtin.debug:
        msg: "{{ token_result.stdout }}"
      when: token_result.changed
```

Run it:

```bash
ansible-playbook playbooks/bootstrap_proxmox.yml
```

Copy the token output into your vault file:

```bash
ansible-vault edit inventory/group_vars/proxmox/vault.yml
```

## 8. `requirements.yml`

```yaml
collections:
  - name: community.general
  - name: community.proxmox
```

Install:

```bash
ansible-galaxy collection install -r requirements.yml
```

## 9. Test Playbook — `playbooks/site.yml`

```yaml
---
- name: Verify Proxmox connectivity
  hosts: proxmox
  gather_facts: true
  tasks:
    - name: Print hostname
      ansible.builtin.debug:
        msg: "Connected to {{ inventory_hostname }} running {{ ansible_distribution }} {{ ansible_distribution_version }}"
```

## 10. Validation

```bash
ansible-vault view inventory/group_vars/proxmox/vault.yml   # verify secrets are encrypted
ansible-inventory --list                                       # verify inventory parses
ansible proxmox -m ping                                        # verify connectivity
ansible-playbook playbooks/site.yml                            # run test playbook
```

## Security Practices Summary

- **SSH keys** over passwords -- no `ansible_password` anywhere in persistent config
- **Ansible Vault** for all secrets -- nothing plaintext in git
- **API tokens** over root credentials for Proxmox API calls
- **Vault indirection pattern** -- encrypted `vault.yml` + unencrypted reference vars
- **`.vault_password` excluded from git** -- store separately or use `--ask-vault-pass` in CI
- **Bootstrap uses `vars_prompt`** -- root password is interactive only, never written to disk

## Next Steps

After bootstrap, harden the server:
- Disable root SSH password auth (`PermitRootLogin prohibit-password` in `sshd_config`)
- Or create a dedicated non-root user with `sudo`/`become` and disable root SSH entirely
