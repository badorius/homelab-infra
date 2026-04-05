# Restore Procedures

This document outlines the steps to restore your infrastructure from the BTRFS snapshots stored on your OpenMediaVault NAS.

## 1. Physical Host Recovery (Bare-metal Arch)

If an Arch host (`arch01-03`) suffers a disk failure:

1.  **Reinstall Arch Linux**: Follow the standard installation or use your existing automated PXE/ISO.
2.  **Mount NAS**: Ensure the OMV NFS share is mounted at `/mnt/nfs`.
3.  **Restore Snapshot**:
    ```bash
    # On the target host
    mount -o subvolid=5 /dev/sda2 /mnt/new-root
    btrfs receive /mnt/new-root < /mnt/nfs/badorius/btrfs-backups/arch01/root-YYYY-MM-DD
    ```
4.  **Update Bootloader**: Re-run `grub-mkconfig` or your bootloader setup to point to the new snapshot.

## 2. VM Recovery (KVM/Libvirt)

Since VM disks are stored in `/var/lib/libvirt/images`, they are included in every host snapshot.

### To restore a specific VM disk image:
1.  Navigate to your latest snapshot on the host: `/.snapshots/root-YYYY-MM-DD/`.
2.  Copy the `.qcow2` file back to the live path:
    ```bash
    cp /.snapshots/root-YYYY-MM-DD/var/lib/libvirt/images/vm01.qcow2 /var/lib/libvirt/images/
    ```
3.  Restart the VM: `virsh start vm01`.

## 3. Kubernetes Cluster Recovery (K3s)

### Full VM Restore (Preferred)
Restore the entire VM using the procedure above. This reverts the OS, ETCD, and cluster state to the snapshot time.

### ETCD Snapshot Restore (Advanced)
If only the Kubernetes database is corrupted:
1.  Locate the latest ETCD snapshot in `/mnt/nfs/backups/k3s/`.
2.  Follow the official K3s restore command:
    ```bash
    k3s server --cluster-reset --cluster-reset-restore-path=/path/to/snapshot
    ```

> [!CAUTION]
> Always stop the `k3s` service on all nodes before performing an ETCD reset.

---

## 4. Persistent Application Data (NFS)

Since your applications (qBittorrent, Grafana, etc.) store their data directly on the OMV NAS via NFS:
- **Case 1: Application pod failure**: Just delete the pod; Kubernetes will remount the NFS volume.
- **Case 2: NAS failure**: You must restore the OMV `/share/` folder from your external/offsite backup.
