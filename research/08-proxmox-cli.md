---
tags: [research, proxmox, cli, cheatsheet, pveam]
created: 2026-09-17 21:31:54 +05
modified: 2026-09-17 21:42:45 +05
---

# Proxmox CLI Cheatsheet

Research date: 2026-09-17. Source: official pve-docs man pages.

## Tool map

| Tool | Purpose | Key subcommands |
|------|---------|-----------------|
| qm | QEMU/KVM VMs | create, clone, set, start, stop, shutdown, reset, suspend, migrate, template, importdisk, resize, cloudinit, guest exec |
| pct | LXC containers | create, destroy, start, stop, reboot, migrate, set, resize, exec, enter, console, snapshot, status |
| pvesh | raw REST-API shell (root) | get, set, create, delete, ls, usage |
| pvesm | storage manager | add, set, remove, status, list, alloc, free, path, scan |
| pvenode | node tasks / SSL / ACME | config get/set, task list/log, startall/stopall, migrateall, acme, cert set |
| pveum | users / roles / ACLs / tokens | user add/token add, role add, aclmod, group add, realm add, acl list |
| pveam | container images | available, update, download, list, remove |
| pve-firewall | firewall (iptables pvefw) | start, stop, status, compile, log; proxmox-firewall = nftables |
| pvesr | storage replication | create-local-job, list, update, disable, delete, status |
| proxmox-backup-client | PBS host/file backups | backup, restore, snapshot list, catalog dump, key import |
| pvecm | cluster | status, add, create, expected, nodes |

Notes: vzdump = backup engine; qmrestore / pct restore = restores; pveversion -v = package list; pveupgrade = release checklist.

## Everyday commands

VMs (qm):

```
qm list
qm create 100 --name web --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0 \
   --scsi0 local-lvm:16 --ostype l26 --ide2 local:iso/ubuntu.iso,media=cdrom \
   --boot order=scsi0
qm set 100 --cores 4 --memory 4096
qm start 100 ; qm stop 100 ; qm shutdown 100 ; qm reboot 100 ; qm suspend 100
qm clone 9000 101 --name web2 --full
qm template 9000
qm disk resize 100 scsi0 +8G
qm guest exec 100 -- apt-get update       # needs qemu-guest-agent
```

Containers (pct):

```
pct list ; pct start 200 ; pct stop 200 ; pct reboot 200
pct set 200 --cores 2 --memory 2048 --mp0 local-lvm:8,mp=/srv/data
pct resize 200 rootfs +4G
pct exec 200 -- apt-get update
pct enter 200                              # shell inside container
pct snapshot 200 pre-upgrade ; pct rollback 200 pre-upgrade
```

Images (pveam):

```
pveam update
pveam available --section system          # mail | system | turnkeylinux
pveam download local ubuntu-24.04-standard_24.04-1_amd64.tar.gz
pveam list local
pveam remove local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst
```

Storage (pvesm):

```
pvesm status
pvesm add nfs nas --path /mnt/nas --server 192.168.1.10 --export /srv --content backup
pvesm set nas --content backup,images
pvesm list local --content iso
pvesm free local:vztmpl/old.tar.gz
```

Users/tokens (pveum):

```
pveum user add alice@pve --password secret --email a@x.io
pveum role add VMAdmin --privs "VM.Allocate,VM.Clone,VM.Config.CPU,VM.Start,VM.Stop"
pveum aclmod /vms/100 --users alice@pve --roles VMAdmin
pveum user token add alice@pve ci --privsep 1    # value shown ONCE - store in Bitwarden
pveum user list --full ; pveum acl list ; pveum passwd alice@pve
```

Node + replication:

```
pvenode config get ; pvenode config set --description "lab node"
pvenode task list --errors
pvenode startall --vms 100,200 --force
pvesr create-local-job 100-0 pve2 --schedule "*/5" --rate 10
pvesr list ; pvesr delete 100-0
```

## Workflows

Ubuntu LXC:

```
pveam update
pveam available --section system
pveam download local ubuntu-24.04-standard_24.04-1_amd64.tar.gz
pct create 200 local:vztmpl/ubuntu-...tar.gz --hostname ct01 --password secret \
   --storage local-lvm --rootfs local-lvm:8 --cores 2 --memory 1024 \
   --net0 name=eth0,bridge=vmbr0,ip=dhcp --unprivileged 1 --features nesting=1 \
   --timezone host --start
```

VM from cloud image (cloud-init):

```
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
qm create 9000 --memory 2048 --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci
qm set 9000 --scsi0 local-lvm:0,import-from=/var/lib/vz/template/iso/noble-server-cloudimg-amd64.img
qm set 9000 --ide2 local-lvm:cloudinit --boot order=scsi0 --serial0 socket --vga serial0
qm set 9000 --ciuser ubuntu --sshkeys ~/.ssh/id_rsa.pub --ipconfig0 ip=dhcp
qm template 9000
qm clone 9000 101 --name app1 ; qm start 101
```

## pvesh <-> REST API

pvesh mirrors https://host:8006/api2/json/... :

```
pvesh get /nodes
pvesh get /nodes/pve/qemu
pvesh create /nodes/pve/qemu/100/clone --newid 101
pvesh set /nodes/pve/qemu/100/status/start
pvesh get /cluster/resources --type vm --output-format json-pretty
pvesh usage /nodes/pve/qemu -v
```

## One-liners

```
qm list
pvesh get /cluster/resources --type vm
zpool status ; zpool list
pvesm status
pvesh get /cluster/backup
pveversion -v
pveupgrade
apt update && apt list --upgradable
pvenode task list --errors
```

## Sources

- https://pve.proxmox.com/pve-docs/qm.1.html
- https://pve.proxmox.com/pve-docs/pct.1.html
- https://pve.proxmox.com/pve-docs/pvesh.1.html
- https://pve.proxmox.com/pve-docs/pveum.1.html
- https://pve.proxmox.com/pve-docs/pveam.1.html
- https://pve.proxmox.com/wiki/Cloud-Init_Support