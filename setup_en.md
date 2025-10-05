# Arch Linux Installation

<br />

## 0. Flash ISO

Let’s get started quickly with the installation.
First, download the official **Arch Linux installation image** from the [Arch Linux download page](https://archlinux.org/download/).
Look for a file named something like:

```
archlinux-[date]-x86_64.iso
```

After downloading, flash the ISO to your USB stick



## 1. Wipe the disk and create a new GPT table

```bash
sudo gdisk /dev/nvme0n1
```

Inside `gdisk`, type:

```
o   # Create a new empty GPT partition table
y   # Confirm wiping existing table

--- 

n   # New partition → root partition
1   # Partition number
    # Accept default first sector
+100G  # Size = 100 GB for root
8304   # Type code for Linux x86-64 root (or just press Enter)

---

n   # New partition → swap
2   # Partition number
    # Accept default first sector
+16G  # Size = 16 GB swap (adjust as needed)
8200   # Type code for Linux swap

---

n   # New partition → EFI system
3   # Partition number
    # Accept default first sector
+512M  # Size = 512 MB EFI
ef00   # Type code for EFI System Partition

---

w   # Write changes to disk
y   # Confirm
```

✅ This creates:

| Partition | Size   | Type             | Mountpoint |
| :-------- | :----- | :--------------- | :--------- |
| p1        | 100 GB | Linux filesystem | `/`        |
| p2        | 16 GB  | Linux swap       | swap       |
| p3        | 512 MB | EFI System       | `/boot`    |

---

## 2. Format partitions

```bash
sudo mkfs.ext4 /dev/nvme0n1p1
```

→ Formats your root partition as `ext4`.

```bash
sudo mkswap /dev/nvme0n1p2
```

→ Formats your swap area.

```bash
sudo mkfs.fat -F32 /dev/nvme0n1p3
```

→ Formats the EFI partition as FAT32 (required by UEFI).

---

## 3. Mount partitions

```bash
sudo mount /dev/nvme0n1p1 /mnt
```

→ Mounts root partition to `/mnt` (install target).

```bash
sudo mkdir /mnt/boot
sudo mount /dev/nvme0n1p3 /mnt/boot
```

→ Mounts the EFI partition at `/boot`.

```bash
sudo swapon /dev/nvme0n1p2
```

→ Activates swap.

---

## 4. Install base system packages

```bash
sudo pacstrap -K /mnt base linux linux-firmware nano networkmanager grub efibootmgr
```

→ Installs core Arch system, Linux kernel, firmware, text editor, networking, and bootloader tools.

---

## 5. Generate `fstab`

```bash
sudo genfstab -U /mnt | sudo tee /mnt/etc/fstab
```

→ Automatically detects your partitions and writes mount info into `/etc/fstab`, which defines what gets mounted at boot.

---

## 6. Enter the new system (chroot)

```bash
sudo arch-chroot /mnt
```

→ Enters your new Arch system’s root as if you had booted into it — crucial for configuring inside the new OS.

---

## 7. Configure timezone, locale, hostname

### Set timezone

```bash
ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
hwclock --systohc
```

→ Sets system clock and timezone to Tokyo.

### Generate locales

```bash
nano /etc/locale.gen
```

→ Uncomment:

```
en_US.UTF-8 UTF-8
ja_JP.UTF-8 UTF-8
```

Then:

```bash
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

→ Sets language environment.

### Set hostname and hosts

```bash
echo "arch-pc" > /etc/hostname
nano /etc/hosts
```

Add:

```
127.0.0.1   localhost
::1         localhost
127.0.1.1   arch-pc.localdomain   arch-pc
```

---

## 8. Create user and enable `sudo`

```bash
useradd -m -G wheel -s /bin/bash ruki
passwd ruki
```

→ Creates your main user and adds it to the `wheel` group.

Enable `sudo`:

```bash
EDITOR=nano visudo
```

→ Uncomment:

```
%wheel ALL=(ALL) ALL
```

---

## 9. Enable networking and time sync

```bash
systemctl enable NetworkManager
systemctl enable systemd-timesyncd
```

→ Enables internet auto-connect and automatic clock sync at boot.

---

## 10. Install GRUB bootloader (UEFI)

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
grub-mkconfig -o /boot/grub/grub.cfg
```

→ Installs GRUB to the EFI partition and generates the boot menu.

---

## 11. Finish up

```bash
exit
umount -R /mnt
swapoff -a
reboot
```

→ Leaves the chroot, unmounts, disables swap, and reboots.

---

✅ After reboot:

* Log in as your `ruki` user.
* You’ll have working internet (NetworkManager).
* Use `sudo` for administrative commands.
* System clock and locale will be correct.

---

## Optional: Essential utilities

```bash
sudo pacman -Syu git base-devel vim htop man-db wget curl
```

→ Updates packages and installs commonly used tools.

---

### Ref

* [Arch Wiki: Installation Guide](https://wiki.archlinux.org/title/Installation_guide)
* [GRUB UEFI Setup](https://wiki.archlinux.org/title/GRUB)
