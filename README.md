# Safe Debian Lab VM (KVM/Libvirt)

A step‑by‑step, GUI + CLI guide for building a disposable, isolated Debian lab VM on Ubuntu using KVM/QEMU with libvirt/virt‑manager.

---

## What this document provides

* A **“why”** for key steps (so it’s not a black box).
* **GUI path** (virt‑manager) and **CLI equivalents** where useful.
* A **safe networking** profile + an optional **LAN‑block** firewall.
* A **base‑and‑overlay disk** workflow so experiments are disposable.

---

## 0) Overview — End State

* **Hypervisor:** KVM/QEMU managed by libvirt/virt‑manager
* **Guest OS:** Debian (stable, lightweight; Debian preferred)
* **Networking:** Default NAT (no inbound from your LAN) + optional rule to block guest from talking to your home LAN at all
* **Isolation:** No shared folders, no clipboard/drag‑drop, no USB passthrough
* **Revertability:** Snapshots + overlay “throwaway” disks

---
Helpful terms:
# KVM, Hypervisor, NAT, and Snapshots — What They Are and Why We’re Using Them

## Hypervisor (the short version)

A **hypervisor** runs and isolates virtual machines (VMs) on a host. Two common models:

* **Type‑1 (bare‑metal):** Runs directly on hardware (e.g., ESXi). Lowest overhead.
* **Type‑2 (hosted):** Runs on top of a host OS (e.g., VirtualBox). More overhead.

**Linux with KVM** blurs the line: the kernel exposes hardware virtualization to VMs (**Type‑1‑like**), while user‑space tools (QEMU/libvirt/virt‑manager) provide device emulation and management. Performance is close to bare‑metal with strong isolation.

## KVM (Kernel‑based Virtual Machine)

**KVM** is a Linux kernel module (`kvm`, `kvm_intel`/`kvm_amd`) that turns the Linux kernel into a hypervisor. It provides:

* **Hardware‑assisted virtualization** (VT‑x/AMD‑V) for fast CPU/memory virtualization.
* **Memory isolation**: the guest runs in its own address space; host/guest memory is separated by the kernel and hardware MMU.
* **Privilege separation**: the guest’s “kernel” runs in a virtualized CPU mode; it cannot directly execute privileged instructions on the host.

**QEMU** adds device emulation (disk, NIC, display), and **libvirt/virt‑manager** handle lifecycle, networks, and storage. Together, they give you an efficient, manageable VM stack.

## NAT (libvirt’s default virtual network)

**NAT** (Network Address Translation) places the VM behind a virtual router (`virbr0` + `dnsmasq`).

* **Outbound works**: the VM can reach the Internet; packets are SNAT’d to the host.
* **Inbound blocked**: the VM has no open ports reachable from your LAN unless you explicitly forward them.
* **LAN visibility reduced**: by default, your LAN can’t initiate connections to the VM. With the optional nftables rule, the VM **also cannot initiate** connections to RFC1918 ranges (10/8, 172.16/12, 192.168/16).

This sharply limits how the VM can interact with your home network.

## Snapshots & Overlays (disposable experiments)

* A **snapshot** captures VM state (disk, optionally memory) at a point in time so you can **revert**.
* A **qcow2 overlay** (copy‑on‑write) stacks a **child** disk on top of a **base** disk. Writes go to the child; the base remains pristine.
* **Reset** = delete the overlay and recreate it; you’re instantly back to a known‑good base.

This workflow prevents experiments from contaminating your golden image.

## Why we’re doing this (your safety goal)

We want: an OS separate from my main system to SSH into CTFs like OverTheWire and work with binaries without risking corruption of my main system.”

**How this design addresses that:**

* **Isolation boundary:** KVM runs guests in hardware‑assisted VMX/SVM modes with separate memory spaces. The guest can’t directly modify host files or processes.
* **Network containment:** NAT prevents unsolicited inbound. Optional nftables rule blocks the VM from contacting your LAN entirely.
* **No host integrations:** No shared folders, no clipboard/drag‑drop, no USB passthrough → fewer pathways from guest to host.
* **Disposability:** Snapshots + overlays let you revert or nuke changes quickly, eliminating persistence of bad states.
* **Principle of least privilege:** Use a non‑root guest user; only `sudo` when needed.
---
## Limits and realistic risk

* **Not an absolute sandbox:** A kernel/hypervisor 0‑day *could* enable a VM escape. This is rare but non‑zero. Mitigations: keep host/guest updated, avoid unnecessary device passthrough, and disable host‑integration features (we did).
* **User‑driven paths:** If you manually copy a malicious file from guest to host and run it, isolation doesn’t help. Prefer one‑way transfers (e.g., `scp` to a quarantined folder) and treat guest artifacts as untrusted.
* **Network trust:** With the RFC1918 drop rule, the VM can’t probe your LAN. Temporarily disable it only if you intentionally need LAN access.

## Bottom line: does this meet your criteria?

**Yes.** For OverTheWire SSH and binary work, this setup provides a strong isolation boundary, safe default networking, and quick rollback. It’s a pragmatic balance of **security, performance, and manageability** suitable for daily use. Keep the base image clean, operate from overlays, and avoid enabling host‑integration features; you’ll substantially reduce the chance of the lab impacting your main system.


---

## 1) Host Prep (Ubuntu)

### 1.1 Verify hardware virtualization

**Why:** KVM needs VT‑x/AMD‑V; if disabled in BIOS, performance tanks or won’t work.

```bash
lscpu | grep -E 'Virtualization|Vendor ID'
# Expect "Virtualization: VT-x" (Intel) or "AMD-V" (AMD). If missing, enable in BIOS/UEFI.

# Optional, more explicit:
sudo apt update
sudo apt install -y cpu-checker
kvm-ok
# Expect: "KVM acceleration can be used"
```

### 1.2 Install KVM + tools

**Why:** These are the hypervisor, management service, and GUI.

```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients \
  virt-manager bridge-utils qemu-utils dnsmasq-base

# Enable and check libvirt
sudo systemctl enable --now libvirtd
systemctl status libvirtd --no-pager
```

### 1.3 Put your user in the right groups

**Why:** Lets you manage VMs without sudo.

```bash
sudo usermod -aG libvirt,kvm "$USER"
newgrp libvirt
newgrp kvm
# (Or log out/in instead of newgrp.)
```

### 1.4 Confirm KVM modules

```bash
lsmod | grep kvm
# Expect kvm_intel or kvm_amd plus kvm
```

---

## 2) Networking (safe by default)

### 2.1 Use libvirt’s default **NAT** network

* Outbound from VM → Internet works.
* Inbound from LAN → VM is blocked by default (no port forwards).
* This already prevents random services on your LAN from reaching the guest.

```bash
virsh net-list --all
# Expect a "default" network, usually 192.168.122.0/24

sudo virsh net-start default
sudo virsh net-autostart default
```

### 2.2 (Optional, recommended) Block guest → home LAN

**Why:** Even with NAT, the guest can initiate connections to LAN devices (e.g., router UI). These nftables rules drop any VM traffic destined to RFC1918 space.

```bash
# Create table + forward chain (no-op if they already exist)
sudo nft add table inet libvirt_filter 2>/dev/null || true
sudo nft add chain inet libvirt_filter forward '{ type filter hook forward priority 0; }' 2>/dev/null || true

# Allow established first (don’t break return traffic you choose to allow)
sudo nft add rule inet libvirt_filter forward ct state established,related accept

# Find your libvirt bridge name (usually virbr0)
ip -brief link | grep virbr

# Drop guest→LAN (10/8, 172.16/12, 192.168/16) — replace virbr0 if different
sudo nft add rule inet libvirt_filter forward iifname "virbr0" ip daddr {10.0.0.0/8,172.16.0.0/12,192.168.0.0/16} drop

# Allow remaining forward traffic from VMs (i.e., to the Internet after NAT)
sudo nft add rule inet libvirt_filter forward iifname "virbr0" accept

# Persist across reboots:
sudo sh -c 'nft list ruleset > /etc/nftables.conf'
sudo systemctl enable --now nftables
```

**Result:** The VM can go out to the Internet for packages/CTFs, but cannot reach anything in your private LAN ranges. Temporarily remove the drop rule if you need LAN access.

---

## 3) Create a **base** VM (Debian)

You can do this entirely in virt‑manager (GUI). GUI first (recommended), then an equivalent CLI.

### 3.1 GUI (virt‑manager)

**Launch:** `virt-manager`

**Create a new VM** (icon with a monitor + star):

1. **Local install media (ISO)** → **Forward**
2. **Use ISO image:** browse to your Debian ISO (e.g., `debian-12.*-amd64-netinst.iso`)
3. **OS type:** Linux → Debian 12 (or autodetected) → **Forward**
4. **CPU & RAM:**

   * RAM: **4096 MB** (4 GB) to start (bump later if Ghidra/large toolchains)
   * CPUs: **2** (bump later if needed)
5. **Storage:**

   * Create a disk image now: **40 GB** (qcow2 sparse, grows as you use it)
   * Name: `debian-lab-base.qcow2`
6. **Network:** Use **NAT: default**.
7. **Customize configuration before install:** **check this** → **Finish**.

**In Customize (hardening & perf):**

* **Overview → Firmware:** BIOS (or UEFI if you need it; BIOS is simplest)
* **CPUs:** set CPU model to **host-passthrough** for performance
* **Memory:** leave as chosen
* **Disks:** bus = **virtio** (faster)
* **NIC:** model = **virtio**
* **Display:** change to **VNC server** (simple)
* **Channel devices (like spicevmc):** **remove** to avoid host/guest clipboard/file channels
* **Serial console:** optional
* **Video:** QXL or Virtio

**Begin installation** and complete Debian setup:

* Create a non‑root user (e.g., `labuser`) and a strong passphrase.
* Select “standard system utilities” + **“SSH server”** if you want SSH into VM.
* Do **not** enable auto‑login or guest additions that share clipboard.

**Post‑install (inside the VM):**

```bash
sudo apt update
sudo apt install -y qemu-guest-agent \
  build-essential gdb valgrind strace ltrace \
  binutils binutils-common binutils-x86-64-linux-gnu \
  curl wget git python3 python3-pip radare2 \
  net-tools iproute2 iputils-ping dnsutils \
  unzip zip make cmake pkg-config

# Optional: basic RE/pwn helpers
pip3 install --break-system-packages pwntools
```

**Why:**

* `qemu-guest-agent` lets the host gracefully shut down/query the VM.
* Tooling aligns with C/RE/CTF work.

**Harden guest → host isolation (virt‑manager):**

* Shut down the VM → open **Details**.
* Ensure **Display = VNC** (not SPICE), and no “agent clipboard”/file sharing is enabled.
* No shared folders (virtiofs/9p) → don’t add any.
* No USB redirection.
* In **Boot Options:** disable **Autostart** unless you want it.

**Snapshot the pristine base:**

* Right‑click VM → **Snapshots** → **Take Snapshot**
* Name: `clean-base`
* Note: “Fresh install + core tools.”

This base image is your **golden reference**.

### 3.2 CLI (equivalent, optional)

**Create storage:**

```bash
mkdir -p ~/VMs
qemu-img create -f qcow2 ~/VMs/debian-lab-base.qcow2 40G
```

**Install with `virt-install` (adjust ISO path/version):**

```bash
virt-install \
  --name debian-lab-base \
  --memory 4096 \
  --vcpus 2 \
  --cpu host-passthrough \
  --disk path=$HOME/VMs/debian-lab-base.qcow2,format=qcow2,bus=virtio \
  --cdrom /path/to/debian-12.x-amd64-netinst.iso \
  --os-variant debian12 \
  --network network=default,model=virtio \
  --graphics vnc \
  --noautoconsole
```

**Snapshot (after post‑install tooling):**

```bash
virsh snapshot-create-as debian-lab-base clean-base "Fresh install + tools"
```

---

## 4) “Throwaway” Overlay Workflow (do risky stuff safely)

**Why:** Keep `debian-lab-base.qcow2` pristine. Do experiments on an **overlay** disk; to revert, delete the overlay.

### 4.1 Create a working overlay

**Shutdown the base VM** if it’s running, then:

```bash
cd ~/VMs
qemu-img create -f qcow2 -F qcow2 -b debian-lab-base.qcow2 debian-lab-work.qcow2 20G
```

**Clone a new VM that uses the overlay (GUI path):**

1. In **virt‑manager** → **Create new** → **Import existing disk image**
2. Point to `~/VMs/debian-lab-work.qcow2`
3. OS: **Debian 12**
4. CPU/RAM same as base
5. Network: **NAT (default)**
6. Ensure **no** shared folders/USB/clipboard
7. **Name:** `debian-lab-work`

**CLI alternative:**

```bash
virt-install \
  --name debian-lab-work \
  --memory 4096 \
  --vcpus 2 \
  --cpu host-passthrough \
  --disk path=$HOME/VMs/debian-lab-work.qcow2,format=qcow2,bus=virtio \
  --os-variant debian12 \
  --network network=default,model=virtio \
  --graphics vnc \
  --import
```

**You now have:**

* **Base VM** (rarely boot — only for updates + snapshot)
* **Work VM** (burnable overlay). Break it as much as you want; delete the overlay to reset.

---

## 5) Day‑to‑Day “Safe Operation” Checklist

### Before risky testing

* Confirm **no shared folders** and **no clipboard/drag‑drop** enabled.
* Confirm **nftables LAN‑block** rule is active (if you chose it):

```bash
sudo nft list ruleset | grep -A2 libvirt_filter
```

* Take a snapshot of `debian-lab-work`:

  * virt‑manager → **VM** → **Snapshots** → name `pre-test-<date>`

### During testing

* Run as **non‑root user** (`labuser`), only sudo when necessary.
* Prefer a dedicated working directory:

```bash
mkdir -p ~/sandboxes/test-$(date +%F)
cd ~/sandboxes/test-$(date +%F)
```

* You can **disconnect VM network** mid‑session in virt‑manager (NIC → “Inactive”) for an air‑gap.

### File transfer (safer options)

* Use **scp** to/from the VM (no shared folder):

```bash
# From host to VM
scp ./somefile labuser@<vm-ip>:/home/labuser/transfer/

# Find VM IP in virt‑manager NIC details or inside VM:
ip -4 addr show
```

* Avoid drag‑drop/clipboard integrations.

### Revert

* If anything feels off: **power off** `debian-lab-work`, delete `~/VMs/debian-lab-work.qcow2`, recreate the overlay (Section 4.1).

---

## 6) Extra Hardening (optional)

* Separate host account for VM administration (limits credentials exposure).
* **No host USB passthrough** to the VM (don’t bridge unknown devices).
* **Disable autostart** for VMs (you choose when the lab runs).
* Keep host & guest updated:

```bash
# Host
sudo apt update && sudo apt upgrade -y
# Guest
sudo apt update && sudo apt upgrade -y
```

* Consider **AppArmor** confinement (Ubuntu default confines libvirt/qemu well).
* Use an **Isolated** libvirt network (no Internet) for certain exercises:

  * virt‑manager → **Edit → Connection Details → Virtual Networks → +**
  * Type **Isolated** → subnet e.g., `192.168.200.0/24`.
  * Attach the VM to that network for a full air‑gap.

---

## 7) Inside‑VM Starter Setup (C / RE track)

```bash
# Basics
sudo apt update
sudo apt install -y build-essential gdb cgdb valgrind strace ltrace \
  binutils gcc-multilib g++-multilib \
  make cmake pkg-config \
  curl wget git ripgrep

# RE-ish
sudo apt install -y radare2 file nasm

# Useful runtime tools
sudo apt install -y htop tmux tree

# Python + pwntools
sudo apt install -y python3 python3-pip
pip3 install --break-system-packages pwntools
```

---

## 8) Quick “Start a Session” Recipe

1. Open **virt‑manager**.
2. Start **`debian-lab-work`**.
3. Take a snapshot: `pre-<task>`.
4. Work as **non‑root**. Keep network NAT on, **RFC1918‑blocked** unless you need LAN.
5. Done? To reset:

   * Shut down VM → delete `debian-lab-work.qcow2` → recreate overlay (Section 4.1).

---

**Notes**

* Replace interface names (`virbr0`) as appropriate on your host.
* Keep the **base** image idle; perform updates occasionally, then snapshot again.
* For advanced isolation, consider additional MAC filtering, separate libvirt networks, or nested virtualization constraints depending on your threat model.
