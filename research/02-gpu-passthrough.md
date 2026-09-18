---
tags: [research, homelab, proxmox, gpu, passthrough, nvidia, ai, llm]
created: 2026-09-17 21:24:01 +05
modified: 2026-09-18 18:25:03 +05
---

# GPU Passthrough - RTX 50-series on Proxmox for AI/LLM

Research date: 2026-09-17. Target GPU: NVIDIA GeForce RTX 5060 Max-Q / Mobile (GB206M, Blackwell, 8 GB GDDR7, sm_120).

## Options for consumer RTX 50-series

Consumer GeForce has NO SR-IOV and NO MIG (vGPU is Pro/Server only). Two realistic paths:

| Method | Isolation | Overhead | Fits |
|--------|-----------|----------|------|
| VM passthrough (VFIO) | Full | low | isolated guests, Windows gaming |
| LXC via /dev/nvidia | none | near zero | local Ollama/vLLM/text-gen - RECOMMENDED |

LXC path: install NVIDIA driver on the host, pass `/dev/nvidia0 nvidiactl nvidia-modeset nvidia-uvm*` (+ nvidia-caps) into the container; install same driver build in LXC with `--no-kernel-modules` or bind host libcuda/ML .so. Avoids VFIO + reset bug entirely. Windows gaming VM = must use VFIO.

## IOMMU / VFIO setup (PVE 9.x, Intel)

```
# BIOS: enable VT-d, Above 4G Decoding/Resizable BAR, disable CSM
# GRUB /etc/default/grub - PVE host. NOTE: for Blackwell VFIO do NOT set
# iommu=pt (device demands a 1:1 IOMMU mapping and the kernel rejects it at
# VM start; research/38). intel_iommu=on is already the kernel default.
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"
update-initramfs -u -k all && proxmox-boot-tool refresh && reboot

# /etc/modules-load.d/vfio.conf
vfio
vfio_pci
vfio_iommu_type1

# /etc/modprobe.d/vfio.conf
options vfio-pci ids=10de:2d59,10de:22eb disable_vga=1

# /etc/modprobe.d/blacklist.conf
blacklist nouveau
blacklist nvidia
blacklist nvidiafb
blacklist nvidia-gpu
softdep snd_hda_intel pre: vfio-pci
```

Verify: `dmesg | grep -i -e DMAR -e IOMMU`, `find /sys/kernel/iommu_groups/* -maxdepth 0`, `lspci -nnk | grep -A3 vfio`. Add `pcie_acs_override=downstream,multifunction` only if GPU shares an IOMMU group.

## Blackwell-specific issues (2026)

- Driver/CUDA: needs NVIDIA R570+ (CUDA 12.8+ for sm_120). Current branches 580/590/595. Open kernel modules recommended/default. PyTorch cu128+ wheels.
- Hypervisor check: consumer Blackwell driver refuses VMs (`osVerifySystemEnvironment failed`). VM CPU: `host,hidden=1,flags=+pcid`, `args: -cpu host,kvm=off,hv_vendor_id=...,-hypervisor`, `rombar=0`, guest `clearcpuid=514`. Windows = Error 43.
- Reset bug (WPR2): GSP firmware keeps Write-Protected Region 2 across PCIe FLR -> GPU dead on 2nd VM start. Fix: host reboot or re-bind nvidia <-> vfio-pci before start. Hits most of 50-series.
- iommu=pt MUST NOT be set for Blackwell VFIO (the device requests a 1:1
  IOMMU mapping and the kernel rejects it at VM start). Omit it; plain
  `intel_iommu=on` suffices (kernel default since 6.8). Also add
  `vfio-pci.disable_idle_d3=1` + a udev D0 rule to avoid D3cold freezes.
  Full 2026 findings: research/38-rtx5060-blackwell-vfio.md.
- PVE kernel regressions: 9.1 (6.17) breaks RTX 5080 VFIO (bugzilla 7374); 9.2 kernel 7.0 `nova_core` module conflicts with VFIO and can hard-hang - check bug reports before `apt full-upgrade`; keep a pinned known-good kernel (6.14/6.17).
- Mobile Max-Q: hybrid/Optimus dGPU can vanish after host reboot; may share IOMMU group with WiFi on 275HX platforms; `pcie_aspm=off` if stalls.

## VRAM -> model size (RTX 5060 = 8 GB, sm_120)

| Model | Q4_K_M | Q8_0 | Fits 8 GB? |
|-------|--------|------|------------|
| 1.5-3B | ~1-2.5 GB | ~2-4 GB | yes (Q8 too) |
| 7B | ~4.2-5.5 GB | ~7.5-8.5 GB | Q4 yes (tight long ctx) |
| 8B (Llama-3) | ~4.9-6 GB | ~9.5 GB | Q4 yes / Q8 no |
| 13B | ~7.5-9 GB | ~14.5 GB | Q3/Q2 borderline |
| 32B+ | ~22 GB | ~36 GB | no |
| 70B | ~42 GB | ~75 GB | no |

Sweet spot: 1-8B quantized models with 1-2 GB KV/context overhead. Audio (HDMI) subdevice: omit for headless AI; pass only for gaming VM.

## Recommended for OmaLaptop

LXC container with host NVIDIA driver + Ollama (no VFIO). Only consider VM passthrough later for isolation or a Windows gaming VM, and only with a pinned safe PVE kernel + reset-bug mitigation.

## Sources

- https://forum.proxmox.com/threads/gpu-passthrough-hang-lockup-after-kernel-update-rtx-5070-ti-gb203-blackwell-7-0-6-2-pve-regression.184197
- https://github.com/siddhant-rajhans/blackwell-proxmox-passthrough
- https://github.com/thanhan92-f1/proxmox-gpu-passthrough/blob/main/docs/TROUBLESHOOTING.md
- https://www.tomshardware.com/pc-components/gpus/rtx-5090-pro-6000-bug-forces-host-reboot
- https://pve.proxmox.com/wiki/NVIDIA_vGPU_on_Proxmox_VE