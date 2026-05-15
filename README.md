# Homelab

## Git

This repository uses Git for version control of all homelab configuration, infrastructure code, and automation scripts.

## Local LLM

Local large language models are run using Ollama and LM Studio. Models are sourced from Hugging Face and sized to fit the available hardware (RTX 4090, 16GB VRAM).

## Terraform

Terraform is used to provision and manage infrastructure as code. Configuration files define the desired state of infrastructure resources, allowing repeatable and auditable deployments.

## Ansible

Ansible is used for configuration management and automation. Playbooks handle provisioning, configuration, and maintenance tasks across homelab hosts.