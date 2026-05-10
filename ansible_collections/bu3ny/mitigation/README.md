# Ansible collection: `bu3ny.mitigation`

Ansible collection for **CopyFail** (**CVE-2026-31431**) and **Dirty Frag / Copy.Fail2**
(**CVE-2026-43284**, **CVE-2026-43500**) Linux mitigations: bootloader / GRUB, **`modprobe.d`**
policy, optional sysctl, reboot orchestration, and **optional coalesced reboot** when both
mitigations run in one play.

Published namespace: [bu3ny on Ansible Galaxy](https://galaxy.ansible.com/ui/namespaces/bu3ny/).

## Layout

| Path | Role |
|------|------|
| `roles/copyfail_mitigation/` | CopyFail mitigation / rollback (mirrors the standalone repo `roles/copyfail_mitigation/`). |
| `roles/dirtyfrag_mitigation/` | Dirty Frag mitigation / rollback (`esp4` / `esp6` / `rxrpc` modprobe policy, etc.). |
| `roles/mitigation_coalesce_reboot/` | Meta-role: single **`ansible.builtin.reboot`** handler for **`Mitigation reboot coalesced`**. |
| `examples/` | Sample inventory, `ansible.cfg`, and playbooks (single-role and coalesced). |

The standalone **Copy-Fail** repository root **`README.md`** documents behaviour in depth; this file focuses on **installing and referencing the collection**.

## Build

From the **collection root** (this directory):

```bash
ansible-galaxy collection build .
```

Produces `bu3ny-mitigation-1.0.5.tar.gz` in the current directory.

## Install (local tarball)

```bash
ansible-galaxy collection install bu3ny-mitigation-1.0.5.tar.gz -p collections
```

Or install into the default path with `-p ~/.ansible/collections`.

From Galaxy after publish:

```bash
ansible-galaxy collection install bu3ny.mitigation:>=1.0.5
```

## Use in a playbook (FQCN)

Declare the collection, then reference roles by **fully qualified collection name** (`namespace.collection.role_name`):

```yaml
---
- hosts: all
  gather_facts: false
  become: true
  collections:
    - bu3ny.mitigation
  roles:
    - role: bu3ny.mitigation.copyfail_mitigation
      vars:
        copyfail_mitigation_state: absent
```

Variables and behaviour match the standalone roles; see each role’s **`defaults/main.yml`** and the upstream repository **`README.md`**. Prefer the six **`examples/playbook-*.yml`** files (table below) over pasting plays by hand.

### Coalesced reboot (CopyFail + Dirty Frag)

To reboot **once** for both roles, set **`mitigation_coalesce_reboot: true`**, list **`bu3ny.mitigation.mitigation_coalesce_reboot` first** (so the coalesced reboot handler is registered), then the two mitigation roles, then **`post_tasks: meta: flush_handlers`**.

**Role order matters:** if **`mitigation_coalesce_reboot`** is omitted or listed **after** the mitigation roles, **`notify: Mitigation reboot coalesced`** may have **no** reboot handler while per-role reboot handlers are suppressed under coalesce — Ansible warns and the host may **not** reboot.

The combined reboot runs only when **`copyfail_mitigation_reboot_allowed`** and **`dirtyfrag_mitigation_reboot_allowed`** are **both** true on the host. If only one role allows auto-reboot, **no** coalesced reboot runs; each role skips post-reboot checks that assume a reboot and adds **manual reboot** summary groups. Align **`copyfail_*`** and **`dirtyfrag_*`** inventory groups (or equivalent vars) when you intend one automatic reboot.

Running **mixed** **`present`** / **`absent`** between the two roles in one play is **unusual**; test in a lab before relying on it in production.

## Example playbooks and inventory

Under **`examples/`**, six playbooks mirror the standalone repo layout:

| Scope | Mitigate (apply) | Rollback |
|-------|------------------|----------|
| CopyFail only | **`playbook-copyfail-mitigate.yml`** | **`playbook-copyfail-rollback.yml`** |
| Dirty Frag only | **`playbook-dirtyfrag-mitigate.yml`** | **`playbook-dirtyfrag-rollback.yml`** |
| Both (one coalesced reboot) | **`playbook-coalesce-mitigate.yml`** | **`playbook-coalesce-rollback.yml`** |

Supporting files:

| File | Purpose |
|------|---------|
| `inventory.ini` | INI inventory with CopyFail and Dirty Frag reboot groups (RFC 5737 placeholders). |
| `inventory.yml` | YAML inventory with the same groups. |
| `group_vars/copyfail_safe_to_reboot.yml` | Example CopyFail vars for lab defaults (`reboot: false`). |
| `group_vars/dirtyfrag_safe_to_reboot.yml` | Example Dirty Frag vars for lab defaults (`reboot: false`). |
| `ansible.cfg` | Sets `collections_path` so `bu3ny.mitigation` resolves when this tree lives under `.../ansible_collections/bu3ny/mitigation/`. |

**Edit the inventory** (host names, `ansible_host`, `ansible_user`, SSH keys, `become` method) before running anything. Mitigation changes bootloader and modprobe configuration; use disposable VMs or approved change windows.

### Run from the unpacked collection (source tree)

```bash
cd examples
for p in playbook-copyfail-mitigate.yml playbook-copyfail-rollback.yml \
         playbook-dirtyfrag-mitigate.yml playbook-dirtyfrag-rollback.yml \
         playbook-coalesce-mitigate.yml playbook-coalesce-rollback.yml; do
  ansible-playbook -i inventory.yml "$p" --syntax-check
done
# When ready (lab only):
# ansible-playbook -i inventory.yml playbook-copyfail-mitigate.yml
```

`ansible.cfg` in `examples/` sets `collections_path` four levels up to the directory that contains `ansible_collections/` (the parent of `ansible_collections`). If you moved only the `mitigation/` folder without that layout, install the built collection into a `collections/` tree and either remove the local `examples/ansible.cfg` `collections_path` line or set `ANSIBLE_COLLECTIONS_PATH` to the directory **above** `ansible_collections`.

### Run after installing the tarball

Install the collection, then run the playbooks from any project directory **without** relying on the bundled `examples/ansible.cfg` collection path (or point `collections_path` / `ANSIBLE_COLLECTIONS_PATH` at your install prefix). Example:

```bash
ansible-galaxy collection install bu3ny-mitigation-1.0.5.tar.gz -p ./collections
ANSIBLE_COLLECTIONS_PATH="$PWD/collections" ansible-playbook \
  -i /path/to/mitigation/examples/inventory.yml \
  /path/to/mitigation/examples/playbook-coalesce-mitigate.yml
```

Adjust `-i` and playbook paths to where you copied `examples/` on your control node.

## Metadata

`galaxy.yml` lists the **bu3ny** Galaxy namespace for `repository` / `documentation` / `homepage` / `issues`. Point `repository` at a real Git repo URL when you have one; Galaxy may require a valid SCM link for some checks.
