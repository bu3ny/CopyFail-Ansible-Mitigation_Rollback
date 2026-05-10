# `copyfail_mitigation`

Ansible role that **applies** or **rolls back** the Linux **CopyFail** mitigation for
**[CVE-2026-31431](https://nvd.nist.gov/vuln/detail/CVE-2026-31431)** on supported hosts.

It persists `initcall_blacklist=algif_aead_init` via the distro bootloader path (**grubby** on
Red Hat-like systems, **`/etc/default/grub`** plus **`update-grub`** / **`grub2-mkconfig`** on
Debian-like and SUSE), manages a **modprobe** drop-in, and can **reboot** when variables and
inventory groups allow it.

## Requirements

- **Ansible** `>= 2.13` (see `meta/main.yml`).
- **Supported distributions** are listed in `defaults/main.yml` (Red Hat-like including gated
  RHEL major versions, Debian-like, SUSE family). Other OS families skip role tasks.
- **Bootloader tools** for the target family must be present (preflight fails early if not).

## Role variables

All variables are prefixed with `copyfail_mitigation_`. Important defaults include:

| Variable | Default | Meaning |
|----------|---------|---------|
| `copyfail_mitigation_state` | `present` | `present` to apply mitigation, `absent` to roll back. |
| `copyfail_mitigation_reboot` | `true` | Whether the role may initiate reboot (subject to inventory groups). |
| `copyfail_mitigation_reboot_allowed_group` | `copyfail_safe_to_reboot` | Hosts in this group may auto-reboot when `copyfail_mitigation_reboot` is true. |
| `copyfail_mitigation_manual_reboot_group` | `copyfail_manual_reboot` | Hosts in this group never auto-reboot. |

See **`defaults/main.yml`** for the full list (timeouts, modprobe paths, optional NVD gating,
SUSE `grub2-mkconfig` output path, rollback extras, etc.).

## Inventory groups

Typical pattern:

- **`copyfail_safe_to_reboot`** — hosts that may receive changes and optional reboot.
- **`copyfail_manual_reboot`** — hosts where the role must not reboot automatically.

## Example (collection FQCN)

```yaml
- hosts: copyfail_safe_to_reboot
  become: true
  collections:
    - bu3ny.mitigation
  roles:
    - role: bu3ny.mitigation.copyfail_mitigation
      vars:
        copyfail_mitigation_state: present
```

## Example (standalone role name)

```yaml
- hosts: copyfail_safe_to_reboot
  become: true
  roles:
    - role: copyfail_mitigation
      vars:
        copyfail_mitigation_state: absent
```

## License

GPL-3.0-only (see role `meta/main.yml`).
