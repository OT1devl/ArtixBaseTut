# ArtixBaseTut

A minimal **Artix Linux** installation with **LUKS full-disk encryption**, **OpenRC**, and a command-line-only setup.

This guide documents the exact installation process used to create the provided virtual machine.

---

# Requirements

* Artix Linux installation media
* UEFI firmware
* Internet connection
* Basic Linux command-line knowledge

---

# 1. Boot the Installer

Boot from the Artix USB image.

Login as:

```bash
root
```

Password:

```text
artix
```

Load your keyboard layout if necessary.

---

# 2. Partition the Disk

Open the partition manager:

```bash
cfdisk /dev/sda
```

Create a GPT partition table and two partitions:

| Partition | Size            | Purpose                |
| --------- | --------------- | ---------------------- |
| /dev/sda1 | 512 MB          | EFI System Partition   |
| /dev/sda2 | Remaining Space | Encrypted Linux System |

Write changes and quit.

Verify:

```bash
lsblk
```

---

# 3. Encrypt the System

Format the Linux partition with LUKS:

```bash
cryptsetup -v luksFormat /dev/sda2
```

Confirm with:

```text
YES
```

Open the encrypted container:

```bash
cryptsetup open /dev/sda2 cryptroot
```

Verify:

```bash
lsblk
```

You should now see:

```text
cryptroot
```

inside `/dev/mapper`.

---

# 4. Create Filesystems

Format the encrypted root partition:

```bash
mkfs.ext4 /dev/mapper/cryptroot
```

Format the EFI partition:

```bash
mkfs.fat -F32 /dev/sda1
```

Mount everything:

```bash
mount /dev/mapper/cryptroot /mnt

mkdir /mnt/boot

mount /dev/sda1 /mnt/boot
```

Verify:

```bash
lsblk
```

---

# 5. Install Artix

Install the base system:

```bash
basestrap /mnt base base-devel openrc elogind-openrc
```

Install kernel and utilities:

```bash
basestrap /mnt linux linux-firmware sof-firmware
```

Install bootloader and networking:

```bash
basestrap /mnt grub efibootmgr
basestrap /mnt networkmanager networkmanager-openrc
```

Install encryption tools:

```bash
basestrap /mnt cryptsetup lvm2
```

Install useful tools:

```bash
basestrap /mnt vim neofetch
```

---

# 6. Generate fstab

Create the filesystem table:

```bash
fstabgen -U /mnt >> /mnt/etc/fstab
```

Verify:

```bash
cat /mnt/etc/fstab
```

---

# 7. Enter the New System

```bash
artix-chroot /mnt
```

From now on, every command runs inside the installed system.

---

# 8. Configure Time

Set timezone:

```bash
ln -sf /usr/share/zoneinfo/Europe/Madrid /etc/localtime
```

Synchronize hardware clock:

```bash
hwclock --systohc
```

Verify:

```bash
date
```

---

# 9. Configure Locale

Edit:

```bash
vim /etc/locale.gen
```

Uncomment:

```text
en_US.UTF-8 UTF-8
```

Generate locales:

```bash
locale-gen
```

Create locale configuration:

```bash
echo 'LANG=en_US.UTF-8' > /etc/locale.conf
```

Verify:

```bash
cat /etc/locale.conf
```

---

# 10. Configure Hostname

Choose a hostname:

```bash
echo artix-base > /etc/hostname
```

Edit hosts:

```bash
vim /etc/hosts
```

Add:

```text
127.0.0.1 localhost
::1 localhost
127.0.1.1 artix-base.localdomain artix-base
```

---

# 11. Create Users

Set root password:

```bash
passwd
```

Create a normal user:

```bash
useradd -m -G wheel -s /bin/bash artix-user
```

Set user password:

```bash
passwd artix-user
```

Enable sudo:

```bash
EDITOR=vim visudo
```

Uncomment:

```text
%wheel ALL=(ALL:ALL) ALL
```

---

# 12. Configure Encryption Support

Edit:

```bash
vim /etc/mkinitcpio.conf
```

Locate the HOOKS line and add:

```text
encrypt lvm2
```

after:

```text
block
```

Generate initramfs:

```bash
mkinitcpio -P
```

---

# 13. Install GRUB

Install GRUB:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Artix
```

Edit:

```bash
vim /etc/default/grub
```

Set:

```text
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=LUKS_UUID_HERE:cryptroot root=UUID=ROOT_UUID_HERE"
```

Automatically replace UUIDs:

```bash
sed -i "s|LUKS_UUID_HERE|$(blkid -o value -s UUID /dev/sda2)|" /etc/default/grub

sed -i "s|ROOT_UUID_HERE|$(blkid -o value -s UUID /dev/mapper/cryptroot)|" /etc/default/grub
```

Generate configuration:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

---

# 14. Enable Services

```bash
rc-update add NetworkManager default

rc-update add elogind default
```

---

# 15. Reboot

Exit chroot:

```bash
exit
```

Unmount:

```bash
umount -R /mnt
```

Reboot:

```bash
reboot
```

Remove the installation media.

---

# 16. Install a Graphical Environment (Optional)

Login as your user:

```bash
su artix-user
```

Update the system:

```bash
sudo pacman -Syu
```

Install Xorg, Firefox and Git:

```bash
sudo pacman -S xorg xorg-xinit firefox git
```
