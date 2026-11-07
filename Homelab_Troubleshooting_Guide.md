# Proxmox VE Troubleshooting: Resolving Duplicate LVM VGs & HP BIOS Boot Issues

This guide details the diagnosis and resolution of duplicate LVM volume group (`pve`) warnings, partition table cleanup, and HP desktop BIOS boot device configuration following drive replacement or reconfiguration.

---

## Technical Overview

* **Node Name:** `HomeServer`
* **Environment:** Proxmox VE 9.2.24 (Kernel `6.14.8-2-pve`)
* **Hardware:** HP Desktop Workstation (Legacy BIOS / HP Setup Utility)
* **Storage Setup:**
  * **Drive 1 (`sdg` / `sda`):** 1.0 TB HDD – Primary OS installation (`pve-root`, `pve-swap`, `EFI`/`BIOS boot`).
  * **Drive 2 (`sdf` / `sdb`):** 500 GB HDD (`WDC WD5000AAKX`) – Former Proxmox OS drive containing legacy LVM metadata.

---

## Problem Statement

1. **LVM Metadata Conflicts:** During system startup, Proxmox threw warnings regarding duplicate Volume Group names (`pve`) present on both the primary 1 TB boot drive and an unformatted/legacy 500 GB drive.
2. **Boot Failure ("Non-System disk or disk error"):** After wiping the partition table of the secondary 500 GB drive (`sdf`), the host hardware attempted to boot from the empty 500 GB drive first, resulting in a BIOS boot failure error screen:
   ```text
   Non-System disk or disk error
   replace and strike any key when ready
   ```

---

## Resolution Steps

### Phase 1: Identifying Drives and Wiping Legacy Metadata

1. Open the Proxmox Shell (or SSH terminal) and check block devices:
   ```bash
   lsblk
   ```
   *Identify the primary drive mounted on `/` (`pve-root`) vs. the secondary target drive.*

2. Erase GPT, MBR, and LVM partition signatures from the target 500 GB drive:
   ```bash
   wipefs -a /dev/sdf
   ```
   *Output confirmation:*
   ```text
   /dev/sdf: 8 bytes were erased at offset 0x00000200 (gpt)
   /dev/sdf: 8 bytes were erased at offset 0x7470c05e00 (gpt)
   /dev/sdf: 2 bytes were erased at offset 0x000001fe (PMBR)
   /dev/sdf: calling ioctl to re-read partition table: Success
   ```

---

### Phase 2: Resolving BIOS Boot Order Errors

When encountering the `Non-System disk or disk error` message on HP hardware:

#### Option A: One-Time Manual Boot (Immediate Access)
1. Power on or restart the server.
2. Tap **F9** repeatedly at the HP logo screen to open **Boot Device Options**.
3. Select **Hard Drive** -> Choose the primary 1 TB drive -> Press **Enter**.

#### Option B: Permanent BIOS Priority Adjustment (HP Setup Utility)
1. Reboot and tap **F10** to enter **Hewlett-Packard Setup Utility**.
2. Navigate to **Storage** tab > **Boot Order**.
3. Select **Hard Drive** (or **Integrated SATA**) and press **Enter**.
4. Use **F6** (Move Up) or **F5** (Move Down) to move the 1 TB primary drive above the 500 GB drive.
5. Press **F10** to accept changes, navigate to **File** > **Save Changes and Exit**.

---

### Phase 3: Initializing Secondary Disk in Proxmox Web UI

1. Open the Proxmox Web UI (`https://<IP>:8006`).
2. Navigate to **Node (`HomeServer`)** > **Disks**.
3. Select `/dev/sdb` (500 GB drive) and click **Initialize Disk with GPT**.
4. Configure as primary VM/LXC storage:
   * **For VM/LXC Disks with Snapshot support:**
     Go to **Disks** > **LVM-Thin** > Click **Create: ThinPool** > Select `/dev/sdb` > Set pool name (`disk500`).
   * **For Backups, ISOs, or Templates:**
     Go to **Disks** > **Directory** > Click **Create: Directory** > Select `/dev/sdb`, select filesystem type (`ext4`), and set storage name.

---

## Visual Reference

Refer to the full illustrated chronicle with screenshots and diagnostic photos in:
👉 [`chronicles/01-proxmox-duplicate-lvm-bios-boot/README.md`](chronicles/01-proxmox-duplicate-lvm-bios-boot/README.md)

---

## Key Takeaways & Best Practices

* **LVM Naming:** Always erase old LVM member headers (`wipefs`) when transferring hard drives between Proxmox nodes to avoid LVM `pve` VG collision.
* **HP BIOS Boot Order:** HP motherboards may not auto-fallback if the first SATA drive is initialized without a bootloader. Manually fix boot priority in F10 BIOS setup.
* **Storage Choice:**
  * **LVM-Thin:** Best for low-overhead block storage with instantaneous copy-on-write snapshotting support for VMs/LXCs.
  * **Directory:** Best for generic filesystem storage (ISO files, VM backups, container templates).
