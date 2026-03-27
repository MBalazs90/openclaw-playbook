# OpenClaw Playbook

Ansible playbook that deploys [OpenClaw](https://github.com/openclaw/openclaw) to a DigitalOcean droplet with hardened security and Tailscale-only access.

## Architecture

```
                    Internet
                       |
              +--------+--------+
              | DigitalOcean    |
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

- A DigitalOcean droplet running Ubuntu 24.04 with your SSH key
- Ansible installed locally
- A [Tailscale auth key](https://login.tailscale.com/admin/settings/keys)
- At least one model provider API key (Moonshot, Anthropic, or OpenAI)

## Quick start

1. Update `ansible/inventory/hosts.yml` with your droplet IP and set `ansible_user: root` / `ansible_port: 22` for first run.

2. Run the playbook:

```bash
cd ansible

ansible-playbook playbook.yml \
  -e "openclaw_gateway_token=$(openssl rand -hex 32)" \
  -e "moonshot_api_key=sk-..." \
  -e "tailscale_auth_key=tskey-auth-..."
```

Or use an Ansible Vault file:

```bash
ansible-playbook playbook.yml \
  --extra-vars @secrets.yml --ask-vault-pass
```

See `secrets.yml.example` for the expected format.

3. After the first run, update `hosts.yml` with the randomized SSH port (printed during deploy) and change `ansible_user` to `openclaw`.

4. Access the dashboard at `https://<tailscale-hostname>` from any device on your tailnet.

## Security

| Layer | Protection |
|-------|-----------|
| **Network** | UFW denies all incoming except SSH; gateway only on loopback |
| **SSH** | Key-only auth, root login disabled, randomized port, fail2ban |
| **Kernel** | ASLR, BPF hardening, ptrace restrictions, reverse path filtering |
| **Access** | OpenClaw reachable only via Tailscale; trusted-proxy auth delegates identity to Tailscale |
| **Updates** | Unattended security upgrades enabled |

## Configuration

All variables are in `ansible/group_vars/all.yml`. Secrets must be passed via `--extra-vars` or Vault -- they are never stored in config files.

## Project structure

```
ansible/
  ansible.cfg
  playbook.yml
  inventory/
    hosts.yml
  group_vars/
    all.yml
  roles/
    hardening/
      defaults/main.yml
      handlers/main.yml
      tasks/main.yml
    tailscale/
      defaults/main.yml
      handlers/main.yml
      tasks/main.yml
    openclaw/
      defaults/main.yml
      handlers/main.yml
      tasks/main.yml
      templates/
        env.j2
        openclaw.json.j2
  secrets.yml.example
```
