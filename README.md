# mbalazs90.openclaw

Ansible Galaxy collection that deploys [OpenClaw](https://github.com/openclaw/openclaw) to any Ubuntu 24.04 cloud server (Hetzner, DigitalOcean, etc.) with hardened security and Tailscale-only access.

## Installation

```bash
ansible-galaxy collection install mbalazs90.openclaw
```

Or from GitHub:

```bash
ansible-galaxy collection install git+https://github.com/MBalazs90/openclaw-playbook.git
```

## Architecture

```
                    Internet
                       |
              +--------+--------+
              | Cloud Server    |
              | Ubuntu 24.04   |
              |                 |
              |  UFW: deny all  |
              |  except SSH     |
              |  (random port)  |
              +--------+--------+
                       |
            +----------+----------+
            |    Tailscale Mesh   |
            |   (encrypted WG)   |
            +----------+----------+
                       |
              Tailscale Serve (HTTPS :443)
                       |
                 127.0.0.1:18789
                       |
              +--------+--------+
              |  Docker         |
              |  openclaw:local |
              |                 |
              |  Gateway        |
              |  (loopback)     |
              |  auth:          |
              |  trusted-proxy  |
              +--------+--------+
                       |
              /opt/openclaw-data/
              (config + workspace)
```

**Key design decisions:**
- Gateway binds to `127.0.0.1` only -- never exposed on the public interface
- Tailscale Serve provides HTTPS termination and proxies to localhost
- Authentication uses `trusted-proxy` mode -- Tailscale identifies users via headers, no device pairing needed
- Docker ports are patched at deploy time to prevent Docker from bypassing UFW
- SSH port is randomized per host (seeded from `machine-id` for idempotency)

## Roles

| Role | Purpose |
|------|---------|
| **hardening** | Creates `openclaw` user, hardens SSH (key-only, no root, random port), enables fail2ban, applies 30+ sysctl rules, configures UFW |
| **tailscale** | Installs Tailscale, authenticates with auth key, configures Serve as HTTPS reverse proxy |
| **openclaw** | Installs Docker, clones repo, builds image with BuildKit, deploys config, starts services via docker compose |

## Prerequisites

- A cloud server running Ubuntu 24.04 with your SSH key (Hetzner, DigitalOcean, or any provider)
- Ansible installed locally
- A [Tailscale auth key](https://login.tailscale.com/admin/settings/keys)
- At least one model provider API key (Moonshot, Anthropic, or OpenAI)

## Quick start

1. Install the collection:

```bash
ansible-galaxy collection install mbalazs90.openclaw
```

2. Create an inventory file with your server IP (use `ansible_user: root` / `ansible_port: 22` for first run):

```yaml
all:
  hosts:
    openclaw-server:
      ansible_host: <SERVER_IP>
      ansible_user: root
      ansible_port: 22
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519
      ansible_python_interpreter: /usr/bin/python3
```

3. Run the included playbook:

```bash
ansible-playbook mbalazs90.openclaw.deploy \
  -i inventory.yml \
  -e "openclaw_gateway_token=$(openssl rand -hex 32)" \
  -e "moonshot_api_key=sk-..." \
  -e "tailscale_auth_key=tskey-auth-..."
```

Or use an Ansible Vault file:

```bash
ansible-playbook mbalazs90.openclaw.deploy \
  -i inventory.yml \
  --extra-vars @secrets.yml --ask-vault-pass
```

See `playbooks/secrets.yml.example` for the expected format.

4. After the first run, update your inventory with the randomized SSH port (printed during deploy) and change `ansible_user` to `openclaw`.

5. Access the dashboard at `https://<tailscale-hostname>` from any device on your tailnet.

## Security

| Layer | Protection |
|-------|-----------|
| **Network** | UFW denies all incoming except SSH; gateway only on loopback |
| **SSH** | Key-only auth, root login disabled, randomized port, fail2ban |
| **Kernel** | ASLR, BPF hardening, ptrace restrictions, reverse path filtering |
| **Access** | OpenClaw reachable only via Tailscale; trusted-proxy auth delegates identity to Tailscale |
| **Updates** | Unattended security upgrades enabled |

## Configuration

All variables are in `playbooks/group_vars/all.yml`. Secrets must be passed via `--extra-vars` or Vault -- they are never stored in config files.

### Skills

Install ClawHub skills or git-based skills automatically during deployment:

```yaml
openclaw_skills:
  - self-improving-agent                                          # from ClawHub
  - playwright                                                    # from ClawHub
  - { name: my-skill, git: "https://github.com/user/skill.git" } # from git
```

## Collection structure

```
galaxy.yml                      # Collection metadata
LICENSE
README.md
roles/
  hardening/                    # SSH, fail2ban, UFW, sysctl
    defaults/main.yml
    handlers/main.yml
    tasks/main.yml
  tailscale/                    # Install, auth, Serve proxy
    defaults/main.yml
    handlers/main.yml
    tasks/main.yml
  openclaw/                     # Docker, build, deploy
    defaults/main.yml
    handlers/main.yml
    tasks/main.yml
    templates/
      env.j2
      openclaw.json.j2
playbooks/
  deploy.yml                    # Main deployment playbook
  ansible.cfg
  inventory/hosts.yml
  group_vars/all.yml
  secrets.yml.example
```
