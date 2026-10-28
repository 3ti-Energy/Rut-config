# Rut-config

## Setup:

Create secrets vault:
`cp vault.yaml.example group_vars/all/vault.yaml`
Then populate with real values

Install requirements:
`ansible-galaxy install -r requirements.yaml`

## Usage:

To run a dry-run and display results:
`ansible-playbook --inventory hosts.yaml site.yaml --limit '3ti-00XX' --diff --check`
