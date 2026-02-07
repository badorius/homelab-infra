# Backup System

## Technology
- BTRFS snapshots
- Incremental send
- systemd timers

## Flow
03:00 Snapshot  
04:00 Send  
05:00 Local Prune  
06:00 Remote Prune  

## Retention
Hosts: 7 snapshots  
OMV: 14 snapshots

## Restore Strategy
Reinstall host → Apply Ansible → Receive snapshot

