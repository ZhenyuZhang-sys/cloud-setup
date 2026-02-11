# Server Environment Setup Guide

This document instructs Claude to set up a new server environment step by step. Execute each step sequentially and verify success before proceeding to the next. All steps require root privileges unless otherwise noted. Ask for confirmation before executing destructive operations (e.g., removing logical volumes, rebooting).

---

## Step 1: Replace Logical Volume on /tdata with ext4 on /dev/sda4

1. Unmount `/tdata` if currently mounted:
   ```bash
   umount /tdata
   ```
2. Identify and remove the logical volume(s) on `/tdata`:
   ```bash
   lvdisplay  # find the LV path associated with /tdata
   lvremove <lv_path>
   ```
   Also remove the volume group and physical volume if they are no longer needed:
   ```bash
   vgremove <vg_name>
   pvremove /dev/sda4
   ```
3. Create a new ext4 filesystem on `/dev/sda4`:
   ```bash
   mkfs.ext4 /dev/sda4
   ```

## Step 2: Mount /dev/sda4 at /mnt on Boot

1. Create a mount point if `/mnt` does not exist:
   ```bash
   mkdir -p /mnt
   ```
2. Get the UUID of `/dev/sda4`:
   ```bash
   blkid /dev/sda4
   ```
3. Add an entry to `/etc/fstab`:
   ```
   UUID=<uuid-of-sda4>  /mnt  ext4  defaults  0  2
   ```
4. Test the mount:
   ```bash
   mount -a
   df -h /mnt
   ```

## Step 3: Clone and Compile GAPBS Workload

1. Clone the GAPBS repository into `/mnt`:
   ```bash
   git clone https://github.com/sbeamer/gapbs.git /mnt/gapbs
   ```
2. Read the README to understand build instructions:
   ```bash
   cat /mnt/gapbs/README.md
   ```
3. Compile with the `bench-graphs` target using 8 parallel jobs (this takes ~40 minutes):
   ```bash
   cd /mnt/gapbs && make bench-graphs -j8
   ```

## Step 4: Clone cxl_scripts to Home Directory

```bash
git clone git@github.com:ZhenyuZhang-sys/cxl_scripts.git ~/cxl_scripts
```

## Step 5: Clone memcg Kernel (zhenyu branch) into /mnt

```bash
git clone -b zhenyu git@github.com:MoatLab/memcgLinux.git /mnt/memcgLinux
```

## Step 6: Install Kernel Compilation Dependencies

Install the packages required to compile a Linux kernel:
```bash
apt-get update && apt-get install -y \
  build-essential bc kmod cpio flex libncurses5-dev \
  libelf-dev libssl-dev dwarves bison rsync zstd debhelper
```

## Step 7: Configure the Kernel

1. Start with a minimal config based on currently loaded modules:
   ```bash
   cd /mnt/memcgLinux && make localmodconfig
   ```
2. Disable `LRU_GEN`:
   ```bash
   scripts/config --disable CONFIG_LRU_GEN
   ```
3. Enable `CONFIG_MMCTL`:
   ```bash
   scripts/config --enable CONFIG_MMCTL
   ```
4. Verify the config:
   ```bash
   grep -E 'LRU_GEN|MMCTL' .config
   ```

## Step 8: Compile and Install the Kernel

1. Compile the kernel with 40 parallel jobs:
   ```bash
   cd /mnt/memcgLinux && make -j40
   ```
2. Install kernel modules:
   ```bash
   make modules_install
   ```
3. Install the kernel:
   ```bash
   make install
   ```

## Step 9: Set GRUB to Boot the mmctl Kernel by Default

1. List available kernels in GRUB:
   ```bash
   grep -E "menuentry " /boot/grub/grub.cfg | head -20
   ```
2. Edit `/etc/default/grub` and set `GRUB_DEFAULT` to the menu entry of the mmctl kernel. Use the full menuentry string or the numeric index (e.g., `"1>2"` for a submenu entry):
   ```
   GRUB_DEFAULT="<appropriate-entry>"
   ```
3. Update GRUB:
   ```bash
   update-grub
   ```
4. Verify:
   ```bash
   grub-editenv list
   ```

## Step 10: Clone Colloid (skx branch) into /mnt

```bash
git clone -b skx git@github.com:MoatLab/colloid.git /mnt/colloid
```

## Step 11: Clone pact-runtime (zhenyu branch) into /mnt

```bash
git clone -b zhenyu git@github.com:MoatLab/pact-runtime.git /mnt/pact-runtime
```

## Step 12: Reboot the Server

**Ask for user confirmation before rebooting.**

```bash
reboot
```

---

## Notes

- Steps 3 (GAPBS compile) and 8 (kernel compile) are long-running. Run them in the background or use `tmux`/`screen` if needed.
- Step 9 requires careful identification of the correct GRUB menu entry. Verify the kernel version string matches the one just compiled.
- After reboot, verify the running kernel with `uname -r` to confirm the mmctl kernel is active.
