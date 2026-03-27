# Security Hardening

Overview of all hardening measures applied by the `hardening` role.

## User Isolation

- Dedicated `openclaw` system user (no password, no login shell access for others)
- SSH key-only access copied from root
- Passwordless sudo via `/etc/sudoers.d/openclaw` (validated with `visudo`)

## SSH Hardening

| Setting | Value |
|---------|-------|
| Port | Randomized per host (10000-65000, seeded from `machine-id`) |
| PermitRootLogin | no |
| PasswordAuthentication | no |
| PermitEmptyPasswords | no |
| X11Forwarding | no |
| AllowAgentForwarding | no |
| AllowTcpForwarding | no |
| MaxAuthTries | 3 |
| ClientAliveInterval | 300s |
| ClientAliveCountMax | 2 |

The SSH port is deterministic per machine (derived from `machine-id` hash) so re-runs are idempotent. Ubuntu 24.04's `ssh.socket` systemd unit is overridden to match the custom port.

## Brute-Force Protection (fail2ban)

- Monitors SSH authentication log
- Bans IP after **5 failed attempts** within 10 minutes
- Ban duration: **1 hour**
- Backend: systemd journal

## Firewall (UFW)

- Default policy: **deny all incoming**, allow all outgoing
- Only the randomized SSH port is allowed through
- Port 22 is temporarily opened during first deploy, then closed after SSH migrates to the new port
- OpenClaw gateway ports (18789, 18790) are bound to `127.0.0.1` only -- never exposed on the public interface

## Automatic Security Updates

- `unattended-upgrades` installed and enabled
- Daily package list updates
- Automatic security patch installation
- Weekly auto-clean of old packages

## Kernel Hardening (sysctl)

30+ kernel parameters hardened via `/etc/sysctl.d/99-hardening.conf`:

### Network

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `net.ipv4.conf.all.accept_source_route` | 0 | Disable IP source routing |
| `net.ipv6.conf.all.accept_source_route` | 0 | Disable IPv6 source routing |
| `net.ipv4.conf.all.accept_redirects` | 0 | Ignore ICMP redirects |
| `net.ipv6.conf.all.accept_redirects` | 0 | Ignore IPv6 ICMP redirects |
| `net.ipv4.conf.all.send_redirects` | 0 | Don't send ICMP redirects |
| `net.ipv4.tcp_syncookies` | 1 | SYN flood protection |
| `net.ipv4.conf.all.log_martians` | 1 | Log packets with impossible addresses |
| `net.ipv4.icmp_echo_ignore_broadcasts` | 1 | Ignore broadcast ICMP (smurf attack prevention) |
| `net.ipv4.icmp_ignore_bogus_error_responses` | 1 | Ignore bogus ICMP errors |
| `net.ipv4.conf.all.rp_filter` | 1 | Strict reverse path filtering |
| `net.ipv6.conf.all.accept_ra` | 0 | Disable IPv6 router advertisements |
| `net.ipv4.ip_forward` | 1 | Kept enabled for Docker networking |

Default variants (`net.ipv4.conf.default.*`) are also applied for completeness.

### Kernel

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `kernel.randomize_va_space` | 2 | Full ASLR (address space layout randomization) |
| `kernel.kptr_restrict` | 2 | Hide kernel pointers from unprivileged users |
| `kernel.dmesg_restrict` | 1 | Restrict dmesg to root only |
| `kernel.perf_event_paranoid` | 3 | Restrict perf events to root only |
| `kernel.unprivileged_bpf_disabled` | 1 | Disable unprivileged BPF |
| `net.core.bpf_jit_harden` | 2 | Harden BPF JIT compiler |
| `kernel.yama.ptrace_scope` | 2 | Restrict ptrace to parent processes only |
| `kernel.unprivileged_userns_clone` | 0 | Block unprivileged user namespaces (container escape prevention) |

### Filesystem

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `fs.suid_dumpable` | 0 | Restrict core dumps for SUID binaries |
| `fs.protected_hardlinks` | 1 | Prevent hardlink-based attacks |
| `fs.protected_symlinks` | 1 | Prevent symlink-based attacks |
| `fs.protected_fifos` | 2 | Protect FIFOs in world-writable directories |
| `fs.protected_regular` | 2 | Protect regular files in world-writable directories |

## Misc

- **Cron directories** (`/etc/cron.d`, `/etc/cron.daily`, etc.) restricted to root only (mode 0700)
- **`su` command** restricted to root via `pam_wheel.so`

## Network Architecture

The gateway is never directly exposed to the internet:

```
Internet --> UFW (deny all except SSH) --> Tailscale Serve (HTTPS) --> 127.0.0.1:18789 --> Docker container
```

All OpenClaw traffic flows through Tailscale's encrypted WireGuard mesh. The `trusted-proxy` auth mode delegates user identity to Tailscale headers, eliminating the need for application-level pairing or tokens.
