# Linux mitigations: CopyFail and Dirty Frag (Ansible)

This repository ships Ansible **roles** (and playbooks) to apply or roll back:

- **CopyFail** (**CVE-2026-31431**) — `initcall_blacklist=algif_aead_init`, modprobe policy, bootloader updates (**`roles/copyfail_mitigation/`**).
- **Dirty Frag / Copy.Fail2** (**CVE-2026-43284**, **CVE-2026-43500**) — modprobe policy for **`esp4`**, **`esp6`**, **`rxrpc`**, optional sysctl, reboot when needed (**`roles/dirtyfrag_mitigation/`**).
- **Coalesced reboot** — **`roles/mitigation_coalesce_reboot/`** plus **`coalesce_*.yaml`** so both mitigations can share **one** reboot when desired.

`ansible.cfg` sets **`roles_path = ./roles`**. Run playbooks from this **repository root** so role names (`copyfail_mitigation`, `dirtyfrag_mitigation`, …) resolve.

### Repository playbooks

Six entry points at the repo root (run with **`ansible-playbook -i … <file>.yaml`**):

| Scope | Mitigate (apply) | Rollback |
|-------|------------------|----------|
| **CopyFail** only | **`copyfail_mitigate.yaml`** | **`copyfail_rollback.yaml`** |
| **Dirty Frag** only | **`dirtyfrag_mitigate.yaml`** | **`dirtyfrag_rollback.yaml`** |
| **Both** (one coalesced reboot) | **`coalesce_mitigate.yaml`** | **`coalesce_rollback.yaml`** |

Single-role plays use one role each; **`coalesce_*`** sets **`mitigation_coalesce_reboot: true`**, lists **`mitigation_coalesce_reboot`** first, then both mitigation roles, then **`post_tasks: meta: flush_handlers`**.

### Dirty Frag / Copy.Fail2 (`dirtyfrag_mitigation`)

A sibling role under **`roles/dirtyfrag_mitigation/`** applies the same *class* of operational mitigations for **Dirty Frag** / **Copy.Fail2** (**CVE-2026-43284**, **CVE-2026-43500**): **`modprobe.d`** policy for **`esp4`**, **`esp6`**, and **`rxrpc`**, optional **`user.max_user_namespaces`** sysctl (when the kernel exposes it), **`modprobe -r`** attempts, and **`reboot`** only when needed. The support matrix explicitly includes **RHEL 6 and 7** (with EL6 caveats documented in the role README). Use **`dirtyfrag_mitigate.yaml`** / **`dirtyfrag_rollback.yaml`** in the table above. Rollback mirrors CopyFail: if **`esp4`/`esp6`/`rxrpc` stay loaded** after removing persistence, the role requests a **reboot** (when allowed), then optionally verifies **`modprobe -n -v`** the same way CopyFail does after rollback.

### One reboot for CopyFail + Dirty Frag (`mitigation_coalesce_reboot`)

By default each role calls **`meta: flush_handlers`** at the end, so a play with **both** roles could **reboot twice**. **`coalesce_mitigate.yaml`** and **`coalesce_rollback.yaml`** set **`mitigation_coalesce_reboot: true`**, list **`mitigation_coalesce_reboot`** **first**, run **`copyfail_mitigation`** then **`dirtyfrag_mitigation`**, and end with **`post_tasks: meta: flush_handlers`**.

This applies to **apply and rollback**: both use the same **`notify`** topics, so one **`Mitigation reboot coalesced`** still produces **one** reboot, then **both** roles’ post-reboot checks (present *or* absent verification) run in handler order.

#### Role order (required for coalescing)

Ansible registers handlers **as each role is loaded**. The **`mitigation_coalesce_reboot`** role defines the **`ansible.builtin.reboot`** handler that listens on **`Mitigation reboot coalesced`**. If you omit it or list it **after** the mitigation roles, **`notify: Mitigation reboot coalesced`** may have **no reboot handler** (Ansible warns; you can get **no reboot** while per-role reboot handlers are suppressed). Always use this order:

1. **`mitigation_coalesce_reboot`**
2. **`copyfail_mitigation`**
3. **`dirtyfrag_mitigation`**
4. **`post_tasks`**: **`meta: flush_handlers`** when **`mitigation_coalesce_reboot`** is true

See also **`roles/mitigation_coalesce_reboot/README.md`**.

#### Reboot policy must align on each host

The combined reboot runs only when **`copyfail_mitigation_reboot_allowed`** and **`dirtyfrag_mitigation_reboot_allowed`** are **both** true (inventory groups / vars per role). If **one** role allows auto-reboot and the **other** does not, **no coalesced reboot** runs; each role skips post-reboot verification that assumes a reboot, emits a **debug** explaining the partner role blocked coalesce, and adds the host to that role’s **manual reboot** summary group (`*_report_manual_reboot_required` or rollback equivalent). Put hosts in **`copyfail_safe_to_reboot`** and **`dirtyfrag_safe_to_reboot`** (or equivalent) when you intend a **single** automatic reboot for both.

#### Mixed `present` / `absent` in one play

Running **CopyFail** **`present`** and **Dirty Frag** **`absent`** (or the reverse) in the **same** coalesced play is **unusual**: one side applies mitigation while the other rolls back, so operator intent, reboot messaging, and summaries are harder to reason about. The roles support it mechanically (each role uses its own `*_state`), but **document and test** before relying on it in production.

#### Collection mirror (`bu3ny.mitigation`)

If you install **`bu3ny.mitigation`** from Galaxy or a built tarball, the **same six-way layout** lives under **`examples/`** with FQCN roles — see **`ansible_collections/bu3ny/mitigation/README.md`** (`playbook-copyfail-*`, `playbook-dirtyfrag-*`, `playbook-coalesce-*`).

### Role path (why rollback can “do nothing” for `disable-algif_aead.conf`)

Ansible picks **one** `copyfail_mitigation` directory from **`roles_path`** (and
similar). If your log lines say `included: …/some-other-folder/roles/copyfail_mitigation/…`,
that is the tree that ran — **not** necessarily the repo you edit. An older copy will
**not** contain newer rollback tasks (CVE drop-in `rm`, `stat` assert, etc.), so files
such as **`/etc/modprobe.d/disable-algif_aead.conf`** will remain.

**Fix:** put this repo’s `roles/` **first** in `roles_path`, delete or rename the duplicate
role, or reference the role by **absolute path** in the play. This role’s **`include_tasks`**
paths use **`{{ role_path }}/tasks/…`** so they still resolve when the play lives in another
directory (e.g. only `role:` points at Copy-Fail). Run **`ansible-playbook -v`** once: you
should see **`copyfail_mitigation role revision 2026.05.03+cve-dropin`** early in the play;
if the revision is missing or different, you are not running this checkout.

## CopyFail role behavior (`copyfail_mitigation`)

The bullets below describe **`copyfail_mitigation`** only. For **Dirty Frag**, see **`roles/dirtyfrag_mitigation/README.md`** (modprobe/sysctl/reboot flow, EL6 caveats, summaries).

- Applies when the host matches an allowlisted **`ansible_facts['distribution']`**
  (see `defaults/main.yml`: RHEL/CentOS/Fedora/Amazon Linux/Alma/Rocky/Oracle/Scientific,
  Ubuntu/Debian, SLES/openSUSE, …). **Red Hat Enterprise Linux** additionally
  requires **`distribution_major_version`** in `copyfail_mitigation_supported_rhel_major_versions`
  (`8`, `9`, `10` by default); other Red Hat-like distros are not major-gated.
- **Preflight:** the role checks that **`modprobe`** and the **bootloader tools**
  for this OS exist before changing anything (see table below).
- `copyfail_mitigation_state: present` persists the kernel argument (**grubby**
  on Red Hat-like hosts, or **`/etc/default/grub`** edits on Debian/SUSE),
  writes a modprobe block file, and runs `modprobe -r algif_aead` **only when**
  the mitigation string is **not** already present on the running kernel’s
  command line (otherwise unload is skipped). If the mitigation is **already**
  on the cmdline, the host is treated as **protected without reboot** even if
  `/sys/module/algif_aead` still exists (e.g. builtin or sysfs quirks)
- `copyfail_mitigation_state: absent` removes this role’s modprobe file
  (`copyfail_mitigation_modprobe_path`, default **`/etc/modprobe.d/copyfail-algif_aead.conf`**)
  and the mitigation kernel argument from the bootloader (**grubby** on Red Hat-like
  systems, or **`/etc/default/grub`** edits plus **`update-grub`** /
  **`grub2-mkconfig`** on Debian / SUSE). When **`copyfail_mitigation_absent_remove_disable_algif_aead_conffile`**
  is **true** (default), rollback also deletes **`copyfail_mitigation_absent_disable_algifaead_conffile_path`**
  (default   **`/etc/modprobe.d/disable-algif_aead.conf`**, the usual Ubuntu CVE drop-in name)
  in a **dedicated** task (`rm -f -- path`, then a `stat` assert) so removal matches the shell
  behaviour operators expect. If the path still exists after that, the play **fails** (permissions /
  become / wrong role path). Set the flag **false** to keep
  that path. **`copyfail_mitigation_absent_extra_modprobe_paths`** lists additional absolute
  paths under **`/etc/modprobe.d/`** to remove. After rollback the role runs
  **`modprobe -n -v algif_aead`** and **warns** if a block policy is still visible.
- Reboot is enabled by default and controlled by a variable
- Inventory groups can override that default per host:
  `copyfail_safe_to_reboot` forces reboot allowed, and
  `copyfail_manual_reboot` forces manual reboot handling
- The role waits up to `300` seconds for virtual guests and `1800` seconds for
  physical hosts to come back after reboot
- If required facts are missing because `gather_facts: false` was used, the role
  gathers the minimal facts it needs on its own
- Before any change, the role performs a preflight check of the current kernel
  command line, whether `algif_aead` is loaded, and whether the modprobe block
  file already exists
- After `grubby --info=ALL`, the role **fails** if it cannot parse any
  `args="..."` lines (so bootloader state is never guessed blindly). Set
  `copyfail_mitigation_allow_unparsed_grubby: true` only as a temporary escape
  hatch if Red Hat changes `grubby` output and you accept the risk
- **Optional NVD gating** (`copyfail_mitigation_nvd_upstream_vulnerable_check`):
  when **true**, `present` compares the leading numeric release parsed from
  `ansible_facts['kernel']` (e.g. `6.8.0` from `6.8.0-60-generic`) to the **NVD
  `linux:linux_kernel` version bands** for [CVE-2026-31431](https://nvd.nist.gov/vuln/detail/CVE-2026-31431).
  If the fact does not yield a clean **`N.N` or `N.N.N`** triplet, the role **warns**
  and treats the host as **still subject to the NVD check** (fail-closed: present
  mitigation applies rather than being skipped on a bad parse). Hosts **outside**
  the bands **skip** grubby/modprobe/reboot for `present` (and appear under
  `nvd_skipped_kernel_not_vulnerable` in the summary). **Rollback (`absent`)
  always runs** so old mitigations can be removed after upgrades.
  **Caveat:** RHEL kernels often keep a stable `5.14.x`-style prefix while
  backports fix CVEs, so the heuristic can stay “vulnerable” after an RHSA;
  keep the flag **false** on EL unless you accept that or add a separate
  vendor-NVR policy later. The numeric **bands in** `nvd_kernel_gate.yml` **should
  be refreshed** when NVD’s `linux:linux_kernel` affected ranges for CVE-2026-31431
  change (there is no live NVD fetch at runtime).
- On **Debian/SUSE**, the role evaluates both **`GRUB_CMDLINE_LINUX_DEFAULT`**
  and **`GRUB_CMDLINE_LINUX`** when they exist. Values are read from
  double-quoted, single-quoted, **unquoted**, or **empty** (`VAR=` / `VAR=\s*`)
  assignments. `present_on_all_kernels` means every defined GRUB cmdline
  variable carries the mitigation string, which is the closest generic
  equivalent to checking all boot entries with `grubby --info=ALL` on RHEL.
  If the same variable appears on **multiple lines**, the role emits a warning
  and models only the **first** match.
- **`update-grub`** / **`grub2-mkconfig`** are reported as **not** changing Ansible’s
  `changed` counter (`changed_when: false`) because the **`replace`** / **`copy`**
  tasks already reflect file mutations; regen is a follow-up only.
- On `present`, reboot is triggered only when the host is **not** considered
  runtime-protected: if the mitigation is already on the cmdline, no reboot; if
  `modprobe -r` ran and `algif_aead` is still visible under `/sys/module`, or
  `modprobe -r` stderr indicates a **builtin** module while the mitigation is
  still **missing** from the cmdline, a reboot is required to converge
- After a reboot on `present`, verification uses **two gates** (both are
  required): **(1)** the booted **`/proc/cmdline`** includes the mitigation
  string (e.g. `initcall_blacklist=…` — kernel boot-time disable path); **(2)**
  **`modprobe -n -v algif_aead`** reflects effective **modprobe.d** policy
  (an **`install algif_aead … /bin/false`**-style line in output,
  **`algif_aead` near `is builtin`** in the same output to avoid unrelated “builtin”
  text, or on some **`kmod`** builds **exit 0** with empty stdout/stderr). Cmdline and modprobe can each disable exposure
  without the other; together they match how Linux actually applies boot args vs
  module configuration.
- After a reboot on `absent`, verification uses **(1)** mitigation removed from
  **`/proc/cmdline`**, and **(2)** by default **`modprobe -n -v algif_aead`** —
  combined stdout/stderr must **not** match **`install algif_aead … /bin/false`**
  nor **`blacklist algif_aead`** (case-insensitive), catching leftover **modprobe.d**
  rules beyond bare `/bin/false` substrings. Set
  **`copyfail_mitigation_verify_modprobe_policy_after_rollback: false`** for
  cmdline-only rollback verification (also use that if you **intentionally**
  blacklist `algif_aead` elsewhere and accept that post-reboot dry-run output).
- If reboot is disabled, the role emits a reminder only for hosts that still
  need reboot to finish convergence
- At the end of a run on supported hosts, the role prints a one-shot summary on
  the controller (a small dict): for `present`, keys include
  `manual_mitigation_reboot_required` and (when NVD gating is on)
  `nvd_skipped_kernel_not_vulnerable`; for `absent`, keys include
  `rollback_noop_target_clean` (already had no modprobe drop-in and no mitigation
  in bootloader state), `rolled_back_without_reboot` (removed persistence while
  the running kernel had no mitigation arg), `rebooted_and_rolled_back`, and
  `manual_rollback_reboot_required` (host lists from `group_by`)
- Use linear strategy (default) for that summary to stay accurate; with
  `strategy: free`, some hosts might not have finished `group_by` before the
  summary task runs. With `serial`, the summary runs once per batch, so lists
  only reflect hosts in that batch unless you move reporting to play `post_tasks`

## Required packages / binaries (preflight)

**CopyFail** needs bootloader tooling per OS family (below) plus **`modprobe`**. **Dirty Frag** does **not** edit GRUB; it needs **`modprobe`** (and **`sysctl`** if you enable the optional userns sysctl). See each role’s README for full preflight lists.

| OS family | Bootloader path (CopyFail) | Typical packages / binaries |
|-----------|-----------------------------|-------------------------------|
| Red Hat-like (RHEL, CentOS, Fedora, Amazon Linux, Alma, Rocky, Oracle, Scientific, …) | `grubby` | `grubby` (often `dnf install grubby` / `yum install grubby`) |
| Debian-like (Ubuntu, Debian, …) | `/etc/default/grub` + `update-grub` | `grub-common`; BIOS/UEFI metapackages (`grub-pc`, `grub-efi-amd64`, …) |
| SUSE (SLES, openSUSE) | `/etc/default/grub` + `grub2-mkconfig` | `grub2` |

`modprobe` is required on all paths (kernel `kmod` stack).

## Variables

### CopyFail (`copyfail_mitigation`)

```yaml
copyfail_mitigation_state: present
copyfail_mitigation_reboot: true
copyfail_mitigation_virtual_reboot_timeout: 300
copyfail_mitigation_physical_reboot_timeout: 1800
copyfail_mitigation_reboot_message: "Reboot initiated by the CopyFail mitigation role"
copyfail_mitigation_reboot_allowed_group: copyfail_safe_to_reboot
copyfail_mitigation_manual_reboot_group: copyfail_manual_reboot
copyfail_mitigation_kernel_arg: initcall_blacklist=algif_aead_init
copyfail_mitigation_modprobe_path: /etc/modprobe.d/copyfail-algif_aead.conf
copyfail_mitigation_default_grub_path: /etc/default/grub
copyfail_mitigation_suse_grub_mkconfig_output: /boot/grub2/grub.cfg
copyfail_mitigation_verify_modprobe_policy_after_rollback: true
copyfail_mitigation_allow_unparsed_grubby: false
copyfail_mitigation_nvd_upstream_vulnerable_check: false
copyfail_mitigation_supported_rhel_major_versions:
  - 8
  - 9
  - 10
# Extend allowlists if your facts use a different distribution string:
# copyfail_mitigation_supported_redhat_like_distributions: [...]
# copyfail_mitigation_supported_debian_like_distributions: [...]
# copyfail_mitigation_supported_suse_distributions: [...]
```

### Dirty Frag (`dirtyfrag_mitigation`)

Defaults and full list: **`roles/dirtyfrag_mitigation/defaults/main.yml`**. Common knobs:

```yaml
dirtyfrag_mitigation_state: present
dirtyfrag_mitigation_reboot: true
dirtyfrag_mitigation_reboot_allowed_group: dirtyfrag_safe_to_reboot
dirtyfrag_mitigation_manual_reboot_group: dirtyfrag_manual_reboot
dirtyfrag_mitigation_modprobe_path: /etc/modprobe.d/dirtyfrag-mitigation.conf
dirtyfrag_mitigation_verify_modprobe_policy_after_rollback: true
dirtyfrag_mitigation_unprivileged_userns_sysctl: false
mitigation_coalesce_reboot: false
# dirtyfrag_mitigation_modules: [esp4, esp6, rxrpc]  # default list
```

Coalesced plays also need **`mitigation_coalesce_reboot: true`** on the play (see **`coalesce_*.yaml`**). Example **`group_vars`** for both roles’ reboot groups lives under **`ansible_collections/bu3ny/mitigation/examples/group_vars/`** (`copyfail_safe_to_reboot.yml`, `dirtyfrag_safe_to_reboot.yml`).

## Example usage

Use the **Repository playbooks** table at the top (`copyfail_*`, `dirtyfrag_*`, `coalesce_*`). Inline equivalents:

**CopyFail** (needs root for bootloader, `modprobe`, `/etc/modprobe.d`):

```yaml
- hosts: all
  become: true
  roles:
    - role: copyfail_mitigation
```

**Dirty Frag** (needs root for `modprobe`, `/etc/modprobe.d`, optional sysctl):

```yaml
- hosts: all
  gather_facts: false
  become: true
  roles:
    - role: dirtyfrag_mitigation
```

**Reboot groups:** each role has its own allowed / manual group names (defaults: `copyfail_safe_to_reboot` / `copyfail_manual_reboot`, `dirtyfrag_safe_to_reboot` / `dirtyfrag_manual_reboot`). Hosts in `*_safe_to_reboot` allow automated reboot even when `*_mitigation_reboot` is false; `*_manual_reboot` forces no auto-reboot from that role. If a host is in **both** allowed and manual for **the same** role, **manual wins**.

Use names that resolve from your controller (`getent hosts …` / `ping`), or set **`ansible_host`**. Example: **`ansible-playbook -i 'root@203.0.113.10,' copyfail_rollback.yaml --ask-pass --user root`**.

```ini
[copyfail_safe_to_reboot]
rhel-vm-01

[copyfail_manual_reboot]
rhel-db-01

[dirtyfrag_safe_to_reboot]
rhel-vm-01

[dirtyfrag_manual_reboot]
rhel-db-01
```

Example **`group_vars`** for lab-style defaults: **`ansible_collections/bu3ny/mitigation/examples/group_vars/`**.

## Verification

**CopyFail** — after reboot, confirm the mitigation is on the running kernel cmdline:

```bash
grep -E 'initcall_blacklist|algif_aead' /proc/cmdline
```

**Dirty Frag** — confirm modprobe policy (no load path for blocked modules), e.g.:

```bash
modprobe -n -v esp4 esp6 rxrpc
```
