# Runbook: Backup & Restore

---

## Backup Overview

There are two independent backup systems:

| System | What | Where | Retention |
|--------|------|-------|-----------|
| BTRFS incremental | Arch Linux root filesystems | OMV NAS | 7 local / 14 NAS |
| K3s ETCD | Cluster state database | NFS at `/mnt/nfs/backups/k3s/` | K3s default (last 5) |

---

## BTRFS Backups

### How It Works

Each Arch Linux host runs these systemd timers:

| Service | Time | Action |
|---------|------|--------|
| `btrfs-snapshot.timer` | 03:00 daily | `btrfs subvolume snapshot -r / /.snapshots/YYYY-MM-DD_HH-MM` |
| `btrfs-send.timer` | 04:00 daily | Incremental `btrfs send \| btrfs receive` → OMV NAS |
| `btrfs-prune.timer` | 05:00 daily | Delete local snapshots older than 7 days |
| Remote prune (on OMV) | 06:00 daily | Delete NAS snapshots older than 14 days |

### Check Backup Status

```bash
# On an arch host (SSH in)
systemctl status btrfs-snapshot.timer
systemctl list-timers | grep btrfs
journalctl -u btrfs-send.service --since today
ls /.snapshots/   # local snapshots

# On OMV NAS (via SSH)
ls /srv/dev-disk-by-uuid-*/btrfs-backups/arch01/
```

### Manual Snapshot

```bash
# On arch01 (as root)
/usr/local/bin/btrfs-snapshot.sh
```

### Manual Send to NAS

```bash
# On arch01 (as root)
/usr/local/bin/btrfs-send.sh
```

---

## K3s ETCD Backups

### How It Works

K3s automatically snapshots the embedded ETCD database. The `k3s-backup` Ansible role configures this to save to NFS.

### Check ETCD Backup Status

```bash
# On vm01 (K3s master)
ls /mnt/nfs/backups/k3s/

# Or via kubectl
kubectl get --raw /api/v1/namespaces/kube-system | grep etcd
```

### Manual ETCD Snapshot

```bash
# On vm01 (as root)
k3s etcd-snapshot save --name manual-backup-$(date +%Y%m%d)
```

---

## Restore Procedures

### Scenario 1: Single Arch Host Recovery (OS corruption)

The host filesystem is on BTRFS. If the OS is corrupted but the disk is intact:

```bash
# Boot from Arch Linux live USB
# Mount the BTRFS root
mount -o subvol=/ /dev/sda2 /mnt

# List available snapshots
ls /mnt/.snapshots/

# Roll back to the latest good snapshot
# Find the snapshot name (e.g., 2026-04-24_03-00)
SNAPSHOT="2026-04-24_03-00"

# Create a new writable snapshot as the new root
btrfs subvolume snapshot /mnt/.snapshots/$SNAPSHOT /mnt/restored-root

# Swap the default subvolume
SUBVOL_ID=$(btrfs subvolume list /mnt | grep restored-root | awk '{print $2}')
btrfs subvolume set-default $SUBVOL_ID /mnt

# Reboot
```

### Scenario 2: Full Arch Host Rebuild (hardware failure)

1. Install fresh Arch Linux on new hardware (BTRFS root)
2. Add the new host to `ansible/inventory/hosts.yml` with the same IP
3. Run Ansible to configure:
   ```bash
   ansible-playbook playbooks/setup_hosts.yml -l arch01
   ```
4. Restore data from NAS:
   ```bash
   # On the new arch01 (as root)
   # Mount the OMV NAS
   mount 192.168.8.15:/share /mnt/nfs
   
   # Receive the latest NAS snapshot
   LATEST=$(ls /mnt/nfs/btrfs-backups/arch01/ | sort | tail -1)
   btrfs receive / < /mnt/nfs/btrfs-backups/arch01/$LATEST
   ```
5. Recreate the VMs:
   ```bash
   ansible-playbook playbooks/setup_vms.yml -l arch01
   ```

### Scenario 3: VM Recovery (vm01/02/03 corrupted)

VMs are disposable. Destroy and recreate:

```bash
# On the host (e.g., arch01)
virsh destroy vm01
virsh undefine vm01
rm /var/lib/libvirt/images/vm01.qcow2
rm /var/lib/libvirt/seed/vm01.iso

# Recreate via Ansible
ansible-playbook playbooks/setup_vms.yml -l arch01
ansible-playbook playbooks/setup_k3s.yml -l vm01
```

The K3s cluster state is in ETCD. After joining a new worker, ArgoCD will redeploy all workloads.

### Scenario 4: K3s Cluster State Recovery (ETCD corruption)

If the K3s cluster database is corrupted:

```bash
# Stop K3s on master
ssh darthv@192.168.8.248 "sudo systemctl stop k3s"

# List available ETCD snapshots
ssh darthv@192.168.8.248 "ls /mnt/nfs/backups/k3s/"

# Restore from a snapshot
SNAPSHOT="etcd-snapshot-2026-04-24"
ssh darthv@192.168.8.248 "sudo k3s server --cluster-reset --cluster-reset-restore-path=/mnt/nfs/backups/k3s/${SNAPSHOT}"

# Restart K3s
ssh darthv@192.168.8.248 "sudo systemctl start k3s"
```

After ETCD restore, workers may need to rejoin. Rerun:
```bash
ansible-playbook playbooks/setup_k3s.yml -l k3s_worker
```

### Scenario 5: Full Cluster Rebuild from Scratch

If everything is lost (disks dead, etc.):

1. Follow the [Fresh Deploy Runbook](fresh-deploy.md)
2. Application data (PVCs) is on the OMV NAS — it survives if the NAS is intact
3. After ArgoCD is running, it will redeploy all services and they will reattach to existing PVCs

**The NFS path for PVC data**: `/share/<namespace>-<pvcname>-pvc-<uid>/` on the OMV NAS.

---

## Backup Verification

Run this monthly to verify backups are working:

```bash
# Check latest snapshot dates on all arch hosts
for HOST in arch01 arch02 arch03; do
  echo "=== $HOST ==="
  ssh darthv@$HOST "ls -lt /.snapshots/ | head -3"
done

# Check NAS has recent snapshots
for HOST in arch01 arch02 arch03; do
  echo "=== $HOST on NAS ==="
  ssh darthv@192.168.8.15 "ls -lt /srv/dev-disk-by-uuid-c1739375-47b6-4818-80ec-5048a25ecc0a/btrfs-backups/$HOST/ | head -3" 2>/dev/null
done

# Check K3s ETCD snapshots
ssh darthv@192.168.8.248 "ls -lt /mnt/nfs/backups/k3s/ | head -5"
```

---

## Re-Apply Backup Configuration

If backup timers stop working, re-apply the Ansible role:

```bash
cd ansible
ansible-playbook playbooks/setup_backups.yml
```
