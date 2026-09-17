# SSH Port Changer

> A safer, guided way to change the SSH port on Linux servers.

**V5.1** · Bash · OpenSSH · MIT

[中文 README](README.md) · Chinese script `ssh-port` · English script `ssh-port-en`

---

## Why

Changing `Port` is easy. Avoiding a remote lockout caused by a firewall, invalid configuration, service restart, or cloud security group is the important part.

SSH Port Changer provides a guarded workflow:

**Detect → choose port → firewall check → backup → validate → restart SSH → verify listener → test a new login**

### Highlights

- 5-step interactive safety wizard
- Generates an unused random port in `20000-60000`
- Detects `ssh.service`, `sshd.service`, and `ssh.socket`
- Automatic configuration backup
- Validates with both `sshd -t` and `sshd -T`
- Supports UFW, firewalld, iptables, and conservatively handles custom nftables
- Supports SELinux `ssh_port_t`
- Automatically restores SSH configuration after critical failures
- Requires explicit cloud/upstream firewall confirmation in wizard mode
- Status, rollback, and old-port cleanup advice

> V5.1 completed a real remote end-to-end test on **Debian GNU/Linux 13 (trixie)**: `17247 → 24976 → successful SSH login → rollback → 17247`.

---

## Install

### English Edition

```bash
curl -fsSL https://raw.githubusercontent.com/treeslins/ssh-port-changer/main/ssh-port-en \
  -o /usr/local/sbin/ssh-port
chmod +x /usr/local/sbin/ssh-port
ssh-port
```

### Chinese Edition

```bash
curl -fsSL https://raw.githubusercontent.com/treeslins/ssh-port-changer/main/ssh-port \
  -o /usr/local/sbin/ssh-port
chmod +x /usr/local/sbin/ssh-port
ssh-port
```

Run as `root`.

---

## Wizard

```text
1) Change SSH port               [Recommended]
2) Show SSH status
3) Restore previous configuration
4) Show old-port cleanup advice
5) Exit
```

The port-change workflow is:

```text
1/5 Choose a new SSH port
2/5 Firewall and cloud ingress check
3/5 Pre-change safety checks
4/5 Final confirmation
5/5 Verify login on the new port
```

The cloud firewall step defaults to **pause**, so repeatedly pressing Enter will not silently bypass this protection.

---

## Commands

| Action | Command |
| --- | --- |
| Start wizard | `ssh-port` |
| Change to a specific port | `ssh-port 17247` |
| Skip the script confirmation | `ssh-port 17247 -y` |
| Show status | `ssh-port --status` |
| Restore previous SSH config | `ssh-port --rollback` |
| Show old-port cleanup advice | `ssh-port --cleanup` |
| Show version | `ssh-port --version` |
| Show help | `ssh-port --help` |

---

## Safe usage

On a cloud server, **allow the new TCP port in the security group/cloud firewall first**. The script cannot reliably manage every provider's upstream ingress policy from inside the server.

After the server-side change succeeds, keep the current SSH session open and test from another terminal:

```bash
ssh -p 17247 root@SERVER_IP
```

Only close the old session after the new login actually works.

Backups:

```text
/root/ssh-port-changer-backups/
```

Latest change state:

```text
/var/lib/ssh-port-changer/state
```

`--rollback` intentionally leaves newly added firewall/SELinux rules in place to reduce lockout risk. `--cleanup` currently provides conservative cleanup advice rather than deleting old rules automatically.

---

## Compatibility

Designed for common OpenSSH Server environments including Debian, Ubuntu, Rocky Linux, AlmaLinux, and RHEL. The fully verified end-to-end environment is currently **Debian 13**.

If an explicit port is found in `ListenAddress`, or a complex custom nftables setup is detected, the script stops conservatively instead of guessing how to rewrite the administrator's network topology.

CI checks both language editions with `bash -n` and ShellCheck.

---

## License

MIT License · Copyright © 2026 treeslins
