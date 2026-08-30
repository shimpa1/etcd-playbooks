# Project agent memory

This file is the project's committed home for project-intrinsic agent knowledge: build, test, release, architecture, and sharp-edge notes that should travel with the code.

- Architecture and full variable reference: see `README.md`. In short:
  `roles/etcd` is generic and inventory-driven (no Proxmox awareness);
  `roles/proxmox_testbed` is an optional, independent VM provisioner used
  only to stand up disposable hosts to validate `roles/etcd` against.
  `roles/etcd` must never import or depend on `roles/proxmox_testbed`.
- Known Proxmox template gotchas (stale cloud-init instance cache,
  `ciupgrade`, `cicustom` on node-restricted storage) and how this repo
  works around each: see README.md "Known Proxmox template gotchas".
- `ansible-playbook`/`--syntax-check` in a non-interactive shell can fail with
  `ERROR: Ansible requires blocking IO on stdin/stdout/stderr` - wrap the
  command in `script -q /dev/null <cmd>` to force blocking I/O.
- Validated end-to-end against a real host: Proxmox VE 9.2.4, node `pve-i2`
  at `10.4.0.13`, template VMID 9000 (`ubuntu-26.04-template`). A 4-node TLS
  cluster (including a live scale-out from 3->4 via the runtime
  `etcdctl member add` join path) was provisioned and confirmed healthy
  there, with unauthenticated/plaintext access confirmed rejected; VMIDs
  3201-3204 were left running afterward as a demonstrated working example.
  A harmless orphaned thin LV `pve/vm-3101-disk-0` (device-mapper wouldn't
  release it, ~12GB, not attached to any VM config) was left over from a
  debugging cycle - safe to `lvremove -f` after it clears on its own, or
  ignore.
- TLS is mandatory, not a toggle (see README.md "TLS"). `roles/etcd`
  generates its own CA/peer/server/client certs via `community.crypto` -
  never assume plaintext etcd URLs when reading or editing this role.
- A `delegate_to` target that references a `set_fact` value set only inside
  a conditional loop (e.g. `etcd_existing_member_host`) must have a
  `| default(...)` fallback: Jinja resolves `delegate_to` before the task's
  own `when` is checked, so an undefined var there fails even on hosts that
  would have skipped the task.

## Maintaining this file

Keep this file for knowledge useful to almost every future agent session in this project.
Do not repeat what the codebase already shows; point to the authoritative file or command instead.
Prefer rewriting or pruning existing entries over appending new ones.
When updating this file, preserve this bar for all agents and keep entries concise.
