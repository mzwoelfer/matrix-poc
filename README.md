# Matrix POC

single-node Matrix deployment using:

- [Synapse](https://github.com/element-hq/synapse) — Matrix homeserver
- PostgreSQL — Synapse database
- [Element Web](https://github.com/element-hq/element-web) — Matrix client
- [Caddy](https://caddyserver.com/) — reverse proxy and automatic HTTPS
- rootless nerdctl + nerdctl compose
- Ansible

small, reproducible Matrix deployment for demonstrating the Matrix protocol.

## Architecture

```text
                         Internet
                            │
                            │ HTTPS :443
                            │ HTTP  :80
                            ▼
                    ┌─────────────────┐
                    │      Caddy      │
                    │ TLS termination │
                    └────────┬────────┘
                             │
                 ┌───────────┴───────────┐
                 │                       │
                 ▼                       ▼
          Element Web :80          Synapse :8008
                                         │
                                         ▼
                                  PostgreSQL :5432
```

Caddy exposes ports to the host:

```text
80/tcp   HTTP / ACME
443/tcp  HTTPS
```

Element, Synapse and PostgreSQL communicate over the internal container network.

## Prerequisites

- A Linux VM/server with SSH access
- SSH key access as user `training`
- A DNS hostname pointing to the server
- TCP ports 80 and 443 reachable from the Internet
- Ansible installed on the control machine

Current POC:

```text
svatrix.duckdns.org → 46.224.69.212
```

## SSH config (`~/.ssh/config`)

Add entries so you can reach both VMs by key:

```sshconfig
Host t12
    HostName 46.224.69.212
    User training
    IdentityFile ~/.ssh/training_ed25519

Host t13
    HostName 167.233.198.193
    User training
    IdentityFile ~/.ssh/training_ed25519
```

Verify connectivity:

```bash
ssh t12
ssh t13
```

## Inventory

Matrix hostname configured per host.

```yaml
all:
  vars:
    ansible_user: training
    ansible_ssh_private_key_file: ~/.ssh/training_ed25519
    ansible_python_interpreter: /usr/bin/python3

  hosts:
    training-12:
      ansible_host: 46.224.69.212
      matrix_domain: svatrix.duckdns.org

    training-13:
      ansible_host: 167.233.198.193
      # Add a DNS hostname here if Matrix should also be deployed
      # on training-13.
```

`matrix_domain` resolves server's public IP.

For example:

```bash
# TEST
dig +short svatrix.duckdns.org

# RETURNS
46.224.69.212
```

## Rollout with Ansible
```bash
# 1. Install ansible roles
ansible-galaxy install -r requirements.yml -p roles/

# Run the deployment:
ansible-playbook -i inventory.yaml matrix-server.yaml
```

Configures the server, 
- installs rootless nerdctl,
- renders the Matrix configuration, 
- starts containers 
- waits for the Matrix client API to become available through HTTPS.

## What the playbook does

For each host with a configured `matrix_domain`:

1. Updates the apt package cache and upgrades installed packages.
2. Installs rootless nerdctl using the `ansible-nerdctl` role.
3. Enables systemd user lingering so the rootless containers survive logout/reboot.
4. Creates the Matrix data directories.
5. Configures PostgreSQL.
6. Configures Synapse using the hostname as its Matrix server name.
7. Configures Element Web to use the HTTPS Matrix endpoint.
8. Deploys Caddy as the reverse proxy.
9. Obtains and renews the TLS certificate automatically.
10. Starts the complete stack using `nerdctl compose`.
11. Creates the configured demo Matrix user.

## Access Element

In browser open:

```text
https://svatrix.duckdns.org
```

## Demo account

Create a demo user:

```text
Username: martin

# Matrix ID is:
@martin:svatrix.duckdns.org
```

The password is configured in `matrix-server.yaml`.

Move the password and other secrets into Ansible Vault.

## TLS / HTTPS

```text
Browser
   │
   │ HTTPS
   ▼
Caddy :443
   │
   │ HTTP
   ▼
Synapse :8008
```

Synapse does not terminate TLS. 
Its HTTP listener is only reachable from the container network.

Intentional: 
- Caddy handles the public TLS connection while 
- Synapse receives the forwarded request internally.

## Container ports

The deployment does **not** expose Synapse or Element directly on the host.

| Component  | Container port |     Host port |
| ---------- | -------------: | ------------: |
| Caddy      |             80 |            80 |
| Caddy      |            443 |           443 |
| Element    |             80 | internal only |
| Synapse    |           8008 | internal only |
| PostgreSQL |           5432 | internal only |


## Matrix user IDs

Because Synapse uses the DNS hostname as its `server_name`, users are identified using that hostname:

```text
@martin:svatrix.duckdns.org
```

rather than using the server IP:

```text
@martin:46.224.69.212
```

This is important because the Matrix server name forms part of every user's permanent Matrix ID.

## Current POC limitations

Currently:

- One Synapse homeserver is deployed.
- PostgreSQL runs locally alongside Synapse.
- Element Web runs on the same server.
- Caddy provides public HTTPS.
- User registration is enabled.
- Matrix discovery endpoints are configured.
- Federation is configured at the Synapse listener level but has not yet been demonstrated with a second homeserver.
- The two VMs are currently independent Matrix deployments; they do not automatically form one Matrix server.

The next useful step for the POC is to deploy two independently named Matrix homeservers and demonstrate federation between them.

For example:

```text
@martin:svatrix.duckdns.org
             │
             │ Matrix federation
             ▼
@alice:other-matrix.example.org
```

## Secrets

The current deployment contains POC credentials such as:

```yaml
postgres_password: ...
registration_shared_secret: ...
matrix_test_password: ...
```

## Useful commands
Check the Matrix stack:

```bash
nerdctl compose -f ~/matrix/docker-compose.yaml ps
```

View Synapse logs:

```bash
nerdctl logs matrix-synapse
```

View Element logs:

```bash
nerdctl logs matrix-element
```

View Caddy logs:

```bash
nerdctl logs matrix-caddy
```

Check the Matrix API:

```bash
curl https://svatrix.duckdns.org/_matrix/client/versions
```

Check Matrix client discovery:

```bash
curl https://svatrix.duckdns.org/.well-known/matrix/client
```

Check Matrix server discovery:

```bash
curl https://svatrix.duckdns.org/.well-known/matrix/server
```


# LEARNING NOTES 

Matrix client API example:

```text
https://svatrix.duckdns.org/_matrix/client/versions
```

You should receive a JSON response containing the supported Matrix client API versions.

## Matrix discovery

Caddy serves the Matrix discovery endpoints:

```text
/.well-known/matrix/client
/.well-known/matrix/server
```

For example:

```bash
curl https://svatrix.duckdns.org/.well-known/matrix/client
```

returns the homeserver discovery information.

The server discovery endpoint allows Matrix clients and other Matrix homeservers to discover the Matrix server endpoint from the server name.

## NOTES
- Whitelist / Blacklist of servers?
- Video/Teams chat
- 

# LICENSE
EUPL-1.2 
