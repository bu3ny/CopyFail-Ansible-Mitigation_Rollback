# `dirtyfrag_mitigation`

Ansible role for **Dirty Frag / Copy.Fail2** kernel mitigations tied to **CVE-2026-43284** (IPsec ESP **`esp4`/`esp6`**) and **CVE-2026-43500** (**`rxrpc`** / AFS-related use cases).

It mirrors the **operational pattern** of the `copyfail_mitigation` role: install a **`modprobe.d`** policy, try **`modprobe -r`**, and use the **`reboot`** module only when the host still needs a cycle to converge—gated by inventory groups.

## What it does

- **`present`**: writes `blacklist` + `install … /bin/false` for **`esp4`**, **`esp6`**, **`rxrpc`**, attempts immediate unload, may trigger reboot.
- **`absent`**: removes this role’s drop-ins and warns if other **`modprobe.d`** files still block those modules. If **`esp4`/`esp6`/`rxrpc` are still loaded** after persistence is removed (same idea as CopyFail when the boot cmdline still carries mitigation), the role **notifies a reboot**; after an allowed reboot, optional **`modprobe -n -v`** checks use **`dirtyfrag_mitigation_verify_modprobe_policy_after_rollback`** (default **true**), same idea as CopyFail’s rollback verification.
- **Optional** (off by default): Red Hat–style **`user.max_user_namespaces=0`** sysctl drop-in **only if** `/proc/sys/user/max_user_namespaces` exists (typically **not** on **EL6**).

## EL6 / EL7

**EL6** targets often lack `user.max_user_namespaces`; the role **skips** the sysctl branch and relies on **modprobe** only. Controllers should run a current Ansible; managed EL6 hosts may need **`ansible_python_interpreter`** and a usable Python stack—treat EL6 as **best-effort** and validate in a lab.

## Requirements

- Ansible **>= 2.13**
- Remote: **`modprobe`** / **kmod** (or **`module-init-tools`** on very old systems)

## Variables (see `defaults/main.yml`)

| Variable | Default | Meaning |
|----------|---------|---------|
| `dirtyfrag_mitigation_state` | `present` | `present` / `absent` |
| `dirtyfrag_mitigation_modprobe_path` | `/etc/modprobe.d/dirtyfrag-mitigation.conf` | Managed drop-in |
| `dirtyfrag_mitigation_unprivileged_userns_sysctl` | `false` | Set `true` to also apply sysctl mitigation (vendor-specific tradeoffs) |
| `dirtyfrag_mitigation_reboot_allowed_group` | `dirtyfrag_safe_to_reboot` | Auto-reboot allowed when in this group |
| `dirtyfrag_mitigation_manual_reboot_group` | `dirtyfrag_manual_reboot` | Never auto-reboot |
| `dirtyfrag_mitigation_verify_modprobe_policy_after_rollback` | `true` | After rollback reboot, fail if `modprobe -n -v` still shows block rules for esp4/esp6/rxrpc |

## Example

```yaml
- hosts: dirtyfrag_safe_to_reboot
  become: true
  roles:
    - role: dirtyfrag_mitigation
      vars:
        dirtyfrag_mitigation_state: present
```

## Tradeoffs

Blocking **`esp4`/`esp6`** breaks **IPsec ESP**; blocking **`rxrpc`** breaks **AFS** client scenarios. Confirm with `lsmod` / architecture review before fleet rollout.

## License

GPL-3.0-only (`meta/main.yml`).
