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


i have a proxmox setup where I setup an api so I login thru ssh with an api key.  this works well for ansible.  and I have an ansible vault.  I want to set up
  terraform to connect to the promox server so what do I need to do with api key, etc.

⏺ Terraform connects to Proxmox via the PVE API (HTTPS on port 8006), not SSH — so your SSH API key won't carry over. You'll need to create a PVE API token and reference
  it from Terraform, ideally pulled from your Ansible vault so secrets stay in one place.

  
Pick a provider

  Two maintained options:

  
bpg/proxmox (Telmate's successor, actively developed) — recommended
Telmate/proxmox — older, still works

  
Create an API token in Proxmox

  On the PVE host (or UI: Datacenter → Permissions → API Tokens):

  # Create a dedicated user
  pveum user add terraform@pve

  # Give it a role (TPVEAdmin is broad; scope down for prod)
  pveum aclmod / -user terraform@pve -role PVEAdmin

  # Create the token — SAVE the secret, it's shown once
  pveum user token add terraform@pve tf-token --privsep=0

  Output gives you:
  
Token ID: terraform@pve!tf-token
Secret: xxxxxxxx-xxxx-... (UUID)

  --privsep=0 means the token inherits the user's perms. Set to 1 and assign ACLs directly to the token for tighter scope.

  
Store secrets in Ansible vault

  Add to your existing vault file:

  proxmox_api_url: "https://pve.example.com:8006/api2/json"
  proxmox_api_token_id: "terraform@pve!tf-token"
  proxmox_api_token_secret: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

  
Wire vault → Terraform

  Terraform doesn't read Ansible vault natively. Three common approaches:

  Option A — env vars via wrapper script (cleanest):

  #!/usr/bin/env bash
  # tf-wrap.sh
  eval "$(ansible-vault view secrets.yml | yq -r 'to_entries | .[] | "export TFVAR" + .key + "=" + (.value | @sh)')"
  terraform "$@"

  Then in Terraform:

  variable "proxmox_api_url" {}
  variable "proxmox_api_token_id" {}
  variable "proxmox_api_token_secret" { sensitive = true }

  terraform {
    required_providers {
      proxmox = {
        source  = "bpg/proxmox"
        version = "~> 0.60"
      }
    }
  }

  provider "proxmox" {
    endpoint  = var.proxmox_api_url
    api_token = "${var.proxmox_api_token_id}=${var.proxmox_api_token_secret}"
    insecure  = false  # set true only if using self-signed cert
  }

  Option B — decrypt to .auto.tfvars (simpler, but secrets land on disk):

  ansible-vault view secrets.yml > secrets.auto.tfvars.json  # gitignored
  terraform apply
  shred -u secrets.auto.tfvars.json

  Option C — external data source calling ansible-vault view (avoids disk, more complex).

  
SSH (optional but common)

  bpg/proxmox also needs SSH for some operations (file uploads, snippet provisioning). Add to the provider block:

  provider "proxmox" {
    # ...
    ssh {
      agent    = true
      username = "root"
    }
  }

  Your existing SSH key setup works here.

  Quick checks before you commit

  
.gitignore: .auto.tfvars,.auto.tfvars.json, .terraform/, terraform.tfstate*
State file contains secrets in plaintext — use a remote backend (S3+KMS, Terraform Cloud) or encrypt locally
TLS cert: if PVE uses self-signed, either trust the CA system-wide or set insecure = true (lab only)

  Want me to scaffold the Terraform files in this repo, or is this for a separate infra repo?
faalbue — 12:23 PM
name: Write token to vault file
      delegate_to: localhost
      become: false
      when: tf_token_result.changed
      block:
name: Build vault payload
        ansible.builtin.set_fact:
          vault_payload: |
            proxmox_api_url: "https://{{/ ansible_host }}:8006/api2/json"
            proxmox_api_token_id: "{{ (tf_token_result.stdout | from_json)['full-tokenid'] }}"
            proxmox_api_token_secret: "{{ (tf_token_result.stdout | from_json)['value'] }}"

          
name: Encrypt and write
          ansible.builtin.shell: |
            echo "{{ vault_payload }}" | ansible-vault encrypt \--vault-password-file ~/.vault_pass \--output {{ playbook_dir }}/../group_vars/all/proxmox_tf.yml
          no_log: true
   - name: Write token to vault file
        delegate_to: localhost
        become: false
        when: tf_token_result.changed
        block:
          - name: Build vault payload
            ansible.builtin.set_fact:
              vault_payload: |
                proxmox_api_url: "https://{{ ansible_host }}:8006/api2/json"
                proxmox_api_token_id: "{{ (tf_token_result.stdout | from_json)['full-tokenid'] }}"
                proxmox_api_token_secret: "{{ (tf_token_result.stdout | from_json)['value'] }}"

          - name: Encrypt and write
            ansible.builtin.shell: |
              echo "{{ vault_payload }}" | ansible-vault encrypt \
                --vault-password-file ~/.vault_pass \
                --output {{ playbook_dir }}/../group_vars/all/proxmox_tf.yml
            no_log: true
﻿
faalbue
faalbue
 
 
 