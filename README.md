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
- Deploys Matrix via `nerdctl compose`: Postgres + Synapse + Element Web (both VMs, independently, not yet connected to each other)
- Creates a demo user `alice` / `S0meLongPocPassword!` on each server

## Try it 
- Open `http://<vm-ip>:8080` in a browser (Element Web)
- Set homeserver URL to `http://<vm-ip>:8008` if not already pre-filled
- Log in as `alice` / `S0meLongPocPassword!`
- Start chatting (only works within that single server for now — no federation yet)

## Notes
- Single-node setup only: `server_name` is set to the VM's IP, no TLS, no federation. This is intentional for now — connecting the two servers is the next step.
- Change `postgres_password` / `registration_shared_secret` in `site.yml` before using this beyond a throwaway test.
 

