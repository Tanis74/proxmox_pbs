# proxmox_pbs

Install and configure Proxmox Backup Server on Debian.

## Requirements

- Debian 13 (Trixie) host with SSH access for the `ansible` user
- `os_bootstrap` and `os_common` roles applied (base system configured)
- **On the control machine:** `sops` installed, age key at `~/.config/sops/age/keys.txt`
- External collections: `ansible.posix`, `community.general`

## Role Variables

See `defaults/main.yml` for the full list. Key variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `proxmox_pbs_version` | `4.2.2-1` | Pinned PBS version. Set to `""` for latest. |
| `proxmox_pbs_datastore_name` | `pbs-datastore` | PBS datastore name |
| `proxmox_pbs_datastore_path` | `/mnt/pbs-datastore` | Mount point for datastore |
| `proxmox_pbs_manage_datastore_disk` | `true` | Auto-discover, format, and mount a disk |
| `proxmox_pbs_datastore_device` | `""` | Override disk device (empty = auto-discover) |
| `proxmox_pbs_prune_schedule` | `daily 03:00` | Prune schedule (calendar event syntax) |
| `proxmox_pbs_gc_schedule` | `daily 04:00` | GC schedule |
| `proxmox_pbs_verify_schedule` | `sun 05:00` | Verify schedule |
| `proxmox_pbs_swap_size_mb` | `2048` | Swap file size in MB (0 = skip) |
| `proxmox_pbs_sync_enabled` | `false` | Create sync job for USB HDD offsite |

## Idempotency

All tasks are idempotent. Running the role twice produces `changed=0` on the second run.

Key idempotency patterns:
- **APT repos:** Checked before writing
- **Package install:** Strict version pin via `proxmox-backup-server={{ version }}`
- **Version pre-flight:** `apt-cache policy` → assert — fails early if pinned version disappeared from the repo
- **Datastore:** `proxmox-backup-manager datastore list` before create
- **Users/tokens:** `user list --output-format json` before create
- **ACL:** `acl list` comparison before update
- **Disk:** Quick-exit if mount point already active (data_disk pattern)
- **Swap:** Quick-exit if swap already active (`swapfree_mb > 0`)

## Token Storage (Manual SOPS)

This role creates two PBS API tokens:
- `backup@pbs!pve-backup` — for Proxmox VE (Datastore.Backup + Datastore.Prune)
- `monitor@pbs!alloy-metrics` — for monitoring (Sys.Audit, read-only)

Token secrets are displayed **once** via `ansible.builtin.debug`. You **must** save them to SOPS immediately:

```bash
sops --encrypt --age <age-public-key> ... > inventory/lab/group_vars/pbs.sops.yml
```

Secrets cannot be retrieved later. If lost:
```bash
proxmox-backup-manager user delete-token backup@pbs pve-backup
proxmox-backup-manager user delete-token monitor@pbs alloy-metrics
# Then re-run the role to generate new tokens
```

## Rollback

To remove PBS:
```bash
apt purge proxmox-backup-server
rm -rf /etc/proxmox-backup /var/lib/proxmox-backup
```

The datastore data at `proxmox_pbs_datastore_path` is preserved unless manually deleted.

## Example Playbook

```yaml
---
- name: Configure Proxmox Backup Server
  hosts: pbs
  gather_facts: true
  roles:
    - role: devops.platform.proxmox_pbs
      tags: ['pbs']
```

## Recovery After NUC Loss

See `homelab/docs/backup.md` scenarios Г and Д:
1. New hardware → Debian 13 → this role → mount existing USB HDD → `datastore create` on existing data
2. Full disaster → restore critical data from S3 (resticprofile), IaC repos from GitHub/GitLab push-mirrors
