# README

## Prerequisites
- SSH key access to both VMs as user `training` (see SSH config below)

## SSH config (`~/.ssh/config`)
Add entries so you can reach both VMs by key:
```
Host t12
    HostName 46.224.69.212
    User training
    IdentityFile ~/.ssh/training_ed25519

Host t13
    HostName 167.233.198.193
    User training
    IdentityFile ~/.ssh/training_ed25519
```
`ssh t12` and `ssh t13` should login

## Inventory
`inventory.yaml` uses SSH key:
```yaml
all:
  vars:
    ansible_user: training
    ansible_ssh_private_key_file: ~/.ssh/training_ed25519
    ansible_python_interpreter: /usr/bin/python3
  hosts:
    training-12:
      ansible_host: 46.224.69.212
    training-13:
      ansible_host: 167.233.198.193
```

## Run from your Ansible control machine
```bash
pip install ansible
ansible-galaxy install -r requirements.yml -p roles/
ansible-playbook matrix-server.yaml
```

## What the playbook does
- Updates apt cache and upgrades all packages (both VMs)
- Installs rootless nerdctl via the `ansible-nerdctl` role (both VMs)
