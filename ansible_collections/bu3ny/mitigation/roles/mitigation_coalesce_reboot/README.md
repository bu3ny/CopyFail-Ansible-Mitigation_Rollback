# `mitigation_coalesce_reboot`

Meta-role: defines **one** `ansible.builtin.reboot` handler listening to **`Mitigation reboot coalesced`**.

Use it when a play runs **`copyfail_mitigation`** and **`dirtyfrag_mitigation`** together with **`mitigation_coalesce_reboot: true`** so a single reboot can satisfy both roles’ handler chains — for **`present`** (apply) **or** **`absent`** (rollback). Rollback notifies use the same **`Mitigation reboot coalesced`** topic when coalescing.

## Playbook order

1. **`mitigation_coalesce_reboot`** (this role — registers the combined reboot handler **first**)
2. **`copyfail_mitigation`**
3. **`dirtyfrag_mitigation`**
4. **`post_tasks`**: `meta: flush_handlers` when `mitigation_coalesce_reboot` (the mitigation roles skip their own flush in that mode)

**Why first?** Handlers are registered as roles load. Listing this role **after** the mitigation roles can leave **`Mitigation reboot coalesced`** without a matching **`reboot`** handler (Ansible warns). Per-role reboot handlers are disabled when coalescing, so the play may **not reboot** at all.

See **`coalesce_mitigate.yaml`** and **`coalesce_rollback.yaml`** in the repository root (third row of the **Repository playbooks** table in the top-level **`README.md`**).

## Mixed `present` / `absent`

A single play may set **`copyfail_mitigation_state`** and **`dirtyfrag_mitigation_state`** independently (e.g. one **`present`**, one **`absent`**). That is **atypical** operationally; verify reboot need, notify topics, and summary groups in a lab before depending on it.

## Reboot policy alignment

The combined reboot runs only when **both** **`copyfail_mitigation_reboot_allowed`** and **`dirtyfrag_mitigation_reboot_allowed`** are true. If they disagree, **no** coalesced reboot runs; each role’s post-reboot checks that require a reboot are skipped, and the role that **did** allow auto-reboot records **manual reboot** groups plus an explanatory **debug**. Align inventory groups (or vars) for both roles on each host when you want one automatic reboot.

## Ansible Galaxy collection

The same role is published as **`bu3ny.mitigation.mitigation_coalesce_reboot`** alongside **`bu3ny.mitigation.copyfail_mitigation`** and **`bu3ny.mitigation.dirtyfrag_mitigation`**; see the collection **`README.md`** and the six **`examples/playbook-*.yml`** entry points (CopyFail, Dirty Frag, coalesce × mitigate/rollback).

## Variables

| Variable | Purpose |
|----------|---------|
| `mitigation_coalesce_reboot` | Must be `true` (set on the play or in inventory). |
| `mitigation_coalesced_reboot_message` | Optional custom reboot message. |
| `mitigation_coalesced_reboot_timeout` | Optional seconds; default is the **max** of each role’s computed reboot timeout on the host. |

Combined reboot runs only if **both** `copyfail_mitigation_reboot_allowed` and `dirtyfrag_mitigation_reboot_allowed` are true (defaults apply if a role did not run). Use the **same** inventory reboot groups for both roles when coalescing.
