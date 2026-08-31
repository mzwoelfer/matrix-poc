# Base VM Setup
 
## Prerequisites
- SSH access to both VMs (`training-12`, `training-13`) as user `training`

## Setup inventory
- Copy the example: `cp example.inventory.yaml inventory.yaml`
- Fill in `ansible_host`, `ansible_ssh_pass`, `ansible_become_pass` for each VM
- `inventory.yaml` is gitignored — never commit it

## Run from your Ansible control machine
```bash
pip install ansible
ansible-galaxy install -r requirements.yml -p roles/
ansible-playbook matrix-server.yml
```

## What the playbook does
- Updates apt cache and upgrades all packages (both VMs)
- Installs rootless nerdctl via the `ansible-nerdctl` role (both VMs)
