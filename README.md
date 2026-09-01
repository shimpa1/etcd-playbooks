# etcd-playbooks

Ansible playbooks for quick creation of multi-node etcd clusters, declaratively
and repeatably - node count, sizing, and network are all variables.

## Design

Two independent pieces:

- **`playbooks/etcd.yml`** (role `roles/etcd`) - the actual deliverable. Takes
  any Ansible inventory group (default name `etcd`) of already-reachable
  Ubuntu/Debian/RHEL-family hosts and turns them into a healthy etcd cluster.
  Cluster size is simply `len(groups['etcd'])` - add or remove hosts from the
  group to resize. Has **no knowledge of Proxmox or any other provisioner**;
  it works identically against bare metal, cloud instances, or hand-written
  inventory.
- **`playbooks/provision_proxmox_testbed.yml`** (role `roles/proxmox_testbed`)
  - optional. Stands up disposable Ubuntu VMs on a Proxmox host purely to give
  the generic playbook something to run against, and writes an inventory file
  the generic playbook can consume. Not a dependency of `roles/etcd` - if you
  already have hosts, skip this entirely.
- **`playbooks/teardown_proxmox_testbed.yml`** - declarative counterpart to
  the above. Destroys exactly the VMIDs the same variables would provision
  and removes the generated inventory file. Idempotent - safe to run even if
  the VMs are already gone.
- **`site.yml`** - convenience wrapper that chains the two for a one-command
  demo against your own Proxmox host.

## Prerequisites

- Ansible core >= 2.15 (tested with `ansible-core` 2.20 / `ansible` 13).
- Install collections: `ansible-galaxy collection install -r requirements.yml`
  (pulls in `community.crypto` for TLS certificate generation, and
  `community.proxmox`, only needed for the Proxmox test-bed path).
- The control node needs the `cryptography` Python package (a dependency of
  `community.crypto`) - already present if `pip install ansible` pulled it in.
- SSH access (with a key, not a password) from the control node to every
  target host, and to the Proxmox host's root account if using the test-bed
  playbook.
- Target hosts need Python 3 and a supported init system (systemd).

## Quick start: bring your own hosts

```bash
cp inventory/hosts.example.yml inventory/my-cluster.yml
# edit ansible_host / ansible_user / ansible_ssh_private_key_file for your hosts
ansible-playbook -i inventory/my-cluster.yml playbooks/etcd.yml
```

Resize the cluster by adding or removing hosts from the `etcd` group in your
inventory and re-running - no task/role changes needed.

## Quick start: demo on the Proxmox test bed

1. Copy the secrets template and fill it in (see **Secrets** below):
   ```bash
   cp group_vars/proxmox_testbed_vault.yml.example group_vars/proxmox_testbed_vault.yml
   ```
2. Review `group_vars/proxmox_testbed.yml` - in particular
   `proxmox_testbed_vm_count`, `proxmox_testbed_static_ips` (must have at
   least `proxmox_testbed_vm_count` entries, and they must be free on your
   network - this repo does not scan for you), and `proxmox_template_vmid`
   (must already exist as a Proxmox template with cloud-init enabled).
3. Run the whole thing end to end:
   ```bash
   ansible-playbook site.yml
   ```
   Or run the two stages separately:
   ```bash
   ansible-playbook playbooks/provision_proxmox_testbed.yml
   ansible-playbook -i inventory/proxmox_testbed/hosts.yml playbooks/etcd.yml
   ```

Re-running either playbook against an already-provisioned cluster is safe:
VMs aren't re-cloned or duplicated, cluster members aren't duplicated, and
config-only changes just trigger an in-place `etcdctl` restart of the
affected node.

To tear the test bed down completely and rebuild from scratch (useful for
proving the whole pipeline is genuinely repeatable, not just idempotent):
```bash
ansible-playbook playbooks/teardown_proxmox_testbed.yml
rm -rf pki/<cluster_name>/   # optional: forces brand-new certs instead of reusing existing ones
ansible-playbook site.yml
```

## Scaling the cluster

- **Fresh cluster**: node count is `proxmox_testbed_vm_count` (test bed) or
  simply how many hosts you list under the `etcd` inventory group (generic
  path). Change the variable/inventory and re-run.
- **Growing an already-running cluster**: `roles/etcd` detects per-node
  whether etcd has already bootstrapped (checks for `<data-dir>/member`) and,
  for a genuinely new host being added to an existing cluster, first runs
  `etcdctl member add` against a healthy existing member before starting the
  new node with `initial-cluster-state: existing`. This is what makes "add a
  host to inventory, re-run" a real scale-out rather than only working for a
  simultaneous first bootstrap. Existing members are left running throughout.
- **Removing a node**: not automated. Run `etcdctl member remove <id>` on a
  remaining member yourself, then drop the host from inventory/decommission
  the VM. Automating safe member removal (which requires care around quorum)
  was out of scope here.

## TLS

The cluster is always TLS-secured - there is no plaintext mode. `roles/etcd`
generates everything itself; nothing is brought in by the operator:

- One cluster **CA** (self-signed).
- A **peer certificate** per node (mutual TLS: `--peer-client-cert-auth`), SAN
  covering the node's `ansible_host` and inventory hostname.
- A **server certificate** per node for client-facing traffic
  (`--client-cert-auth`), SAN additionally covering `127.0.0.1` so local
  `etcdctl` calls verify cleanly.
- One shared **client certificate**, deployed to every node, used by this
  role's own health checks and by `etcdctl member add` during scale-out.

Generation uses `community.crypto` (`openssl_privatekey` / `openssl_csr` /
`x509_certificate`), which is idempotent by design - certs are only
regenerated when their parameters (SANs, validity, key type/size) actually
change. The CA private key and all generated certs/keys are written to
`etcd_pki_dir` (default `pki/<cluster_name>/` at the repo root - already in
`.gitignore`) on the control node, then pushed out to each target host's
`etcd_remote_pki_dir` (default `/etc/etcd/pki`) over the existing SSH
connection. **Back up or otherwise protect `etcd_pki_dir` yourself** - if you
lose the CA key, every node needs new certs.

To use `etcdctl` yourself against a running cluster (from the control node,
using the generated client cert):
```bash
etcdctl --endpoints=https://<node-ip>:2379 \
  --cacert=pki/etcd-cluster/ca.pem \
  --cert=pki/etcd-cluster/client.pem \
  --key=pki/etcd-cluster/client-key.pem \
  endpoint health
```

Relevant variables (`roles/etcd/defaults/main.yml`): `etcd_pki_dir`,
`etcd_remote_pki_dir`, `etcd_key_type`, `etcd_key_size`,
`etcd_ca_validity_days`, `etcd_cert_validity_days`. Certificate rotation
before expiry is not automated - re-running the playbook after bumping the
validity/key variables (or just deleting the relevant file(s) under
`etcd_pki_dir`) regenerates and redeploys them, followed by an automatic
`etcdctl`-triggered restart of the affected node(s).

## Variables reference

### `roles/etcd` (generic - `roles/etcd/defaults/main.yml`, `group_vars/etcd.yml`)

| Variable | Default | Purpose |
|---|---|---|
| `etcd_group_name` | `etcd` | Inventory group that forms the cluster |
| `etcd_cluster_name` | `etcd-cluster` | `initial-cluster-token` / unit description |
| `etcd_version` | `v3.7.1` | Pinned etcd release to install |
| `etcd_client_port` / `etcd_peer_port` | `2379` / `2380` | etcd listen ports |
| `etcd_data_dir` / `etcd_config_dir` / `etcd_install_dir` | `/var/lib/etcd`, `/etc/etcd`, `/usr/local/bin` | Filesystem layout |
| `etcd_system_user` / `etcd_system_group` | `etcd` | Service account |
| `etcd_health_check_retries` / `_delay` | `12` / `5` | Health-check polling |
| `etcd_pki_dir` | `pki/<cluster_name>/` | Control-node CA/cert working directory (gitignored) |
| `etcd_remote_pki_dir` | `/etc/etcd/pki` | Where certs/keys are deployed on each node |
| `etcd_key_type` / `etcd_key_size` | `RSA` / `2048` | Generated key algorithm/size |
| `etcd_ca_validity_days` / `etcd_cert_validity_days` | `3650` / `825` | CA / leaf certificate validity |

### `roles/proxmox_testbed` (optional - `roles/proxmox_testbed/defaults/main.yml`, `group_vars/proxmox_testbed.yml`)

| Variable | Purpose |
|---|---|
| `proxmox_api_host` / `proxmox_api_user` / `proxmox_node` | Proxmox API/host target |
| `proxmox_template_vmid` | Source template to clone (must have cloud-init + `agent: enabled=1`) |
| `proxmox_vmid_start` | First VMID to use; must be clear of existing VMIDs on your host |
| `proxmox_storage` / `proxmox_bridge` / `proxmox_vlan_tag` | Disk storage / network bridge / optional VLAN tag for cloned VMs |
| `proxmox_snippet_storage` / `proxmox_snippet_storage_path` | Storage (with "snippets" content enabled) used for the cloud-init vendor-data workaround below |
| `proxmox_testbed_vm_count` | **Node count** - how many VMs to clone |
| `proxmox_testbed_vm_cores` / `_sockets` / `_memory_mb` | VM sizing |
| `proxmox_testbed_static_ips` / `_network_prefix` / `_gateway` | Static addressing for cloned VMs (must have >= `proxmox_testbed_vm_count` free IPs) |
| `proxmox_ci_user` | cloud-init user created on each VM (SSH key comes from the vault file) |

## Secrets

Never commit Proxmox API credentials or SSH private keys. Two supported
conventions (both satisfy this repo's "no secrets committed" rule):

1. **Untracked vars file (default, simplest)**: copy
   `group_vars/proxmox_testbed_vault.yml.example` to
   `group_vars/proxmox_testbed_vault.yml` and fill in real values. That exact
   filename is already in `.gitignore`.
2. **Ansible Vault**: `ansible-vault encrypt group_vars/proxmox_testbed_vault.yml`
   after filling it in - an encrypted file is safe to commit. Pass
   `--ask-vault-pass` or `--vault-password-file` to `ansible-playbook` when
   running.

Create the Proxmox API token referenced by `proxmox_api_token_id` with, e.g.:
```bash
pveum user token add root@pam ansible-etcd --privsep 0
```
(`--privsep 0` gives the token the same privileges as `root@pam`; scope it
down with a dedicated role/user for anything beyond a personal lab.)

## Serial console + VGA

Every VM this repo creates that enables a serial console (`serial0: socket`,
useful for `qm terminal` / automation observability) is always given a
standard VGA display (`vga: std`) in the same task - see
`roles/proxmox_testbed/tasks/provision_one_vm.yml`. A VM definition is never
shipped serial-only.

## Known Proxmox template gotchas this repo works around

- **Stale cloud-init instance cache**: a template not cleaned with
  `cloud-init clean` before being converted causes every clone to skip
  re-applying network config on boot (cloud-init logs
  `Event Denied: scopes=['network'] EventType=boot-legacy`). Worked around
  with a small cloud-init vendor-data snippet
  (`roles/proxmox_testbed/templates/vendor-data.yaml.j2`) that forces
  `netplan apply` via `runcmd` on every boot, without touching the shared
  template. The proper long-term fix is running `cloud-init clean --logs` on
  the template itself.
- **`ciupgrade` inherited from the template**: if the template has automatic
  package upgrade on first boot enabled, every clone runs a full
  `apt dist-upgrade` (often including a kernel/grub update) before the
  network even comes up, adding many minutes to provisioning. The test-bed
  role explicitly sets `ciupgrade: false` on its clones.
- **`cicustom` pointing at node-restricted storage**: if the template's
  cloud-init vendor-data reference lives on storage restricted to a different
  cluster node, clones on other nodes inherit a broken reference. The
  test-bed role overrides `cicustom` with its own snippet on
  `proxmox_snippet_storage` instead of relying on the inherited value.

## Idempotency

- VM provisioning: `community.proxmox.proxmox_kvm` clone/update calls are
  keyed by VMID, so re-running skips re-cloning an existing VM.
- etcd install/config: version-checked before re-downloading; config/unit
  changes only trigger a restart via Ansible handlers, not on every run.
- Cluster bootstrap: `initial-cluster-state` is only consulted by etcd on a
  node's genuinely first start (empty data-dir) - re-running against an
  already-formed cluster is safe regardless of what's rendered into the
  config file.
- TLS: `community.crypto` only regenerates a key/CSR/certificate when its own
  parameters change, so re-running with no variable changes leaves existing
  certs untouched (no restart triggered).
