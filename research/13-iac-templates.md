---
tags: [research, iac, terraform, tofu, ansible, cloudinit, gitops]
created: 2026-09-17 21:38:22 +05
modified: 2026-09-17 21:42:45 +05
---

# IaC & Templates (Terraform / OpenTofu / Ansible / cloud-init)

Research date: 2026-09-17.

## Terraform + bpg/proxmox (the standard)

- `bpg/proxmox` (fork of abandoned danitso provider) is de-facto standard: ~14M downloads, v0.111.x (mid-2026), targets PVE 9.x (8.x limited, 7.x unsupported). Needs Terraform 1.5+ / OpenTofu 1.6+.
- Auth via API token (preferred) or username+password; an `ssh` block is required for disk-import/snippet ops.

```hcl
terraform {
  required_providers {
    proxmox = { source = "bpg/proxmox", version = "~> 0.111" }
  }
}
provider "proxmox" {
  endpoint  = "https://pve.lan:8006/"
  username  = "terraform@pve"
  api_token = var.api_token
  ssh { agent = true }
}
```

## Core flow: cloud image -> template -> clone

```hcl
resource "proxmox_virtual_environment_download_file" "ubuntu" {
  content_type = "import"            # enable "Import" content on storage
  datastore_id = "local"
  node_name    = "pve"
  url          = "https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img"
  file_name    = "noble-server-cloudimg-amd64.qcow2"
}

resource "proxmox_virtual_environment_vm" "tmpl" {
  name = "ubuntu-noble-tmpl"; node_name = "pve"; vm_id = 9000
  on_boot = false
  agent { enabled = true }
  disk {
    datastore_id = "local-lvm"
    file_id      = proxmox_virtual_environment_download_file.ubuntu.id
    interface    = "scsi0"; size = 10
  }
  initialization {
    ip_config { ipv4 { address = "dhcp" } }
    user_account { username = "ubuntu"; keys = [var.ssh_pubkey] }
  }
  clone { retries = 1 }
  template = true
}

resource "proxmox_virtual_environment_vm" "guest" {
  for_each = { web = { vm_id = 200, cores = 2, mem = 2048 } }
  name = each.key; node_name = "pve"; vm_id = each.value.vm_id
  clone { vm_id = proxmox_virtual_environment_vm.tmpl.vm_id }
  cpu { cores = each.value.cores }
  memory { dedicated = each.value.mem }
  initialization {
    ip_config { ipv4 { address = "dhcp" } }
    user_account { username = "ubuntu"; keys = [var.ssh_pubkey] }
  }
}
```

LXC variant uses `proxmox_virtual_environment_download_file` with content_type=vztmpl, then `proxmox_virtual_environment_container` resource (unprivileged, features.nesting).

Filtering with data sources: `proxmox_virtual_environment_vms` supports filter by name/template/status regex to discover existing VMs.

## OpenTofu (2026 status)

- Drop-in replacement, same HCL and provider protocol (runs bpg/proxmox unchanged). OpenTofu 1.12 (May 2026), MPL-2.0, Linux Foundation + CNCF. Terraform 1.15.x under BUSL/IBM.
- For homelab: OpenTofu is the safe default. Adds state encryption.

## Ansible + division of labor

- Old community.general proxmox modules deprecated -> use `community.proxmox` collection (v2.x). Modules run on the controller; proxmoxer+requests needed there.

```yaml
- hosts: localhost
  connection: local
  tasks:
    - community.proxmox.proxmox_kvm:
        api_host: pve.lan
        api_user: terraform@pve
        api_token_id: ansible
        api_token_secret: "{{ pve_token }}"
        node: pve
        name: web
        clone: ubuntu-noble-tmpl
        cores: 2
        memory: 2048
        net: { net0: 'virtio,bridge=vmbr0' }
        ciuser: ubuntu
        state: present
```

- `proxmox_inventory` plugin = dynamic inventory straight from the cluster API.
- Division of labor: Terraform/OpenTofu = provisioning (VMs, disks, networks, templates); Ansible = configuration (packages, services, user state). Keep the inventory as code.

## cloud-init in PVE

- Guest gets a nocloud drive rendered from qm config: --ciuser --cipassword --sshkeys --ipconfig0 --nameserver --searchdomain.
- Custom user-data via `qm set <vmid> --cicustom "user=local:snippets/user.yaml"` (needs "Snippets" content type; files under /var/lib/vz/snippets/).
- Vendordata set on template is retained by clones; user-data regenerated per guest. cicustom replaces the auto config entirely (GUI/qm user/ssh/ip ignored). Packages belong in the snippet (packages:, runcmd:).

## Version control / GitOps

- /etc/pve is pmxcfs (database) - cannot just git-init it. Options: `proxmox-backup-client backup etc.pxar:/etc` for config snapshots, or a git bare repo on the node committing /etc/pve + /etc/network/interfaces to a private remote (config only, never secrets).
- Guests are reproducible via IaC in a git repo (state + HCL). Keep state in the repo (or s3 backend).

## Best practices (single-node homelab)

- One VM template per distro; clone for guests, never full provisioning.
- QEMU guest agent on all VMs; stop_on_destroy to avoid hangs.
- Restrict Terraform to guest workload; manage node stuff (users, networks, storage) by hand.
- Prefer scoped API tokens over root@pam; keep tokens out of git (env vars / sops / vault).
- Use OpenTofu, pin provider, keep state in git.

## Sources

- https://registry.terraform.io/providers/bpg/proxmox/latest
- https://github.com/bpg/terraform-provider-proxmox/blob/main/docs/guides/cloud-init.md
- https://pve.proxmox.com/wiki/Cloud-Init_Support
- https://galaxy.ansible.com/ui/repo/published/community/proxmox/content/module/proxmox_kvm
- https://docs.ansible.com/projects/ansible/latest/collections/community/proxmox/proxmox_inventory.html
- https://github.com/opentofu/opentofu