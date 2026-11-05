---
layout: default
title: Arch Linux Installation on Proxmox
---

# Arch Linux Installation on Proxmox

**Course:** CYB 3353 — System Administration  
**Purpose:** A short, repeatable record of my manual Arch Linux installation, including commands, decisions, and troubleshooting.  
**Reference:** [Official Arch Linux installation guide](https://wiki.archlinux.org/title/Installation_guide)

> **Before repeating this guide:** All disk commands below target a disposable **32 GiB VM disk `/dev/sda`** and can destroy its contents. Confirm the disk name with `lsblk` first. The IP configuration shown is specific to my course lab; don't reuse it on another network without permission. Commands under *Installation* run from the Arch live ISO unless otherwise noted.

## 1. VM setup and choices

I installed Arch Linux manually inside a VM on the course Proxmox cluster. I used the Arch Linux x86-64 installation ISO, a 32 GiB QEMU virtual disk, and a VirtIO network adapter (`ens18`). The VM booted in **Legacy BIOS** mode rather than UEFI, so I used **GPT with a small BIOS boot partition** and the **GRUB BIOS bootloader**. I did not use `archinstall`.

| Setting | My choice / observation |
| --- | --- |
| Platform | Course Proxmox VM |
| Installation method | Manual Arch Linux ISO |
| Disk | `/dev/sda`, 32 GiB |
| Firmware | Legacy BIOS |
| Partition table | GPT |
| Filesystem | ext4 on `/dev/sda2` |
| Bootloader | GRUB, `i386-pc` target |
| Network interface | `ens18` (VirtIO) |
| Lab static IP | `10.30.39.10/24` |
| Gateway | `10.30.39.1` |
| DNS | `1.1.1.1` |
| Hostname | `archlinux` |

**Check before publishing:** Add your actual CPU/RAM settings if required; they were not confirmed in the installation screenshots. Do not publish login credentials or other private lab details.

## 2. Boot and check the live installer

After attaching the ISO in Proxmox, I started the VM and selected the Arch Linux installation medium. The prompt was `root@archiso`.

```bash
ls /sys/firmware/efi          # No directory: live ISO was booted in Legacy BIOS mode
lsblk                        # Find the target disk; mine was /dev/sda, 32G
ip a                         # Inspect network interfaces; mine was ens18
ping -c 3 archlinux.org      # Check DNS and connectivity
```

The first ping failed with `Temporary failure in name resolution`. `ip a` showed `ens18` was UP but had no IPv4 address, and `ip route` showed no route. The Proxmox network did not provide DHCP for this VM.

### Live installer: temporary static networking

```bash
ip addr add 10.30.39.10/24 dev ens18  # Assign my course-lab IP to ens18
ip link set ens18 up                  # Bring up the interface
ip route replace default via 10.30.39.1 dev ens18  # Configure the default gateway
resolvectl dns ens18 1.1.1.1          # Set the DNS resolver for the live session
resolvectl domain ens18 '~.'          # Route DNS lookups through this interface
ping -c 3 1.1.1.1                    # Test IP connectivity without DNS
ping -c 3 archlinux.org              # Test DNS and internet connectivity
```

**Lesson:** A working network adapter does not mean it has an IP address, route, or DNS. The static settings used in the live ISO do **not** automatically carry over to the installed system.

## 3. Partition the VM disk

I confirmed `/dev/sda` was the empty 32 GiB VM disk. **This procedure destroys any previous partition table on that disk.**

```bash
fdisk /dev/sda    # Open the partition editor for the verified VM disk
```

Inside `fdisk`, I used the following keys:

| Input | What it did |
| --- | --- |
| `g` | Created a new GPT partition table |
| `n`, default partition/start, `+2M` | Created a 2 MiB first partition |
| `t`, `4` | Changed partition 1 to **BIOS boot** (verify type in `p` output) |
| `n`, defaults for remaining prompts | Created partition 2 using the rest of the disk |
| `p` | Reviewed the planned layout before writing |
| `w` | Wrote the changes to disk |

The `p` output showed `/dev/sda1` (2 MiB, BIOS boot) and `/dev/sda2` (approximately 32 GiB, Linux filesystem).

```bash
mkfs.ext4 /dev/sda2       # Create an ext4 filesystem on the root partition
mount /dev/sda2 /mnt      # Mount the target filesystem for installation
lsblk                     # Confirm /dev/sda2 is mounted at /mnt
```

**Important:** I did **not** format `/dev/sda1`. GRUB uses that BIOS boot partition for embedding boot code on BIOS/GPT systems.

## 4. Install the base system

```bash
pacstrap -K /mnt base linux linux-firmware grub networkmanager nano sudo
# Install the base system, kernel, firmware, GRUB, and administration utilities

genfstab -U /mnt > /mnt/etc/fstab  # Write persistent mount entries using filesystem UUIDs
arch-chroot /mnt                   # Enter the installed Arch system to configure it
```

### Mistake: first pacstrap attempt filled the live environment

My first attempt returned `Partition / too full` and `failed to install packages to new root`. I then tried `genfstab` and `arch-chroot`, but they failed because **`/mnt` was not mounted**. I corrected the problem by mounting `/dev/sda2` at `/mnt` and running `pacstrap` again. The second installation succeeded, and the kernel's initramfs generated successfully.

```bash
lsblk -f               # Check filesystems and mount points
mount /dev/sda2 /mnt   # Recovery: mount the intended root filesystem
 df -h /mnt            # Check available space on the actual installation target
```

After the successful installation, `/dev/sda2` showed approximately **28 GiB free**. I generated `fstab` and entered `arch-chroot` only after `pacstrap` completed without errors.

## 5. Configure the installed system (inside arch-chroot)

```bash
ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime  # Set Central time zone
hwclock --systohc                                         # Write system time to hardware clock
nano /etc/locale.gen                                      # Uncomment: en_US.UTF-8 UTF-8
locale-gen                                                # Generate the selected locale
echo 'LANG=en_US.UTF-8' > /etc/locale.conf                # Set the default language
echo 'archlinux' > /etc/hostname                          # Set the hostname
nano /etc/hosts                                           # Add local hostname mappings
passwd                                                    # Set the root password interactively
```

The `/etc/hosts` entries used were:

```text
127.0.0.1 localhost
::1       localhost
127.0.1.1 archlinux.localdomain archlinux
```

I configured persistent networking using `systemd-networkd`, not NetworkManager, even though NetworkManager was included in the original `pacstrap` package list. **Only one network manager should control this interface.**

```bash
nano /etc/systemd/network/20-wired.network  # Create the static network profile
```

Contents:

```ini
[Match]
Name=ens18

[Network]
Address=10.30.39.10/24
Gateway=10.30.39.1
DNS=1.1.1.1
```

```bash
systemctl enable systemd-networkd    # Start static network configuration at boot
systemctl enable systemd-resolved    # Start DNS resolution at boot
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf  # Use resolved for DNS
```

## 6. Install the BIOS GRUB bootloader

Because the VM booted in BIOS mode and the disk used GPT, I installed GRUB to the **disk**, not a partition:

```bash
grub-install --target=i386-pc /dev/sda    # Install GRUB's BIOS bootloader
grub-mkconfig -o /boot/grub/grub.cfg     # Generate GRUB menu entries
```

I then exited the chroot and rebooted:

```bash
exit                 # Leave the installed system's chroot
umount -R /mnt       # Unmount installation filesystems
reboot               # Restart into the installed OS
```

In Proxmox, I made sure the virtual hard disk was ahead of the ISO in the boot order. The VM successfully booted into installed Arch Linux and displayed a login prompt.

## 7. Post-installation and troubleshooting

After reboot, I checked the network and attempted to update packages and install OpenSSH.

```bash
ip -4 addr show ens18        # Confirm the static address is present
ip route                     # Confirm the default gateway
resolvectl status            # Inspect DNS configuration
ping -c 3 archlinux.org      # Confirm name resolution
pacman -Syu                  # Synchronize repositories and upgrade packages
```

### Issue: DNS errors after reboot

Package downloads initially failed with `Temporary failure in name resolution`. I checked the IP address and route, enabled `systemd-resolved`, set DNS to `1.1.1.1`, and linked `/etc/resolv.conf` to the systemd-resolved stub. This was a reminder that live-ISO networking and installed-system networking are separate configurations.

```bash
systemctl enable --now systemd-resolved     # Enable and start the resolver
resolvectl dns ens18 1.1.1.1               # Set DNS for the running interface
resolvectl domain ens18 '~.'               # Prefer this interface for DNS
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf  # Fix resolver link
```

### Issue: Pacman reported invalid/corrupted packages (PGP signature)

When installing `openssh`, Pacman reported signatures from **Christian Hesse** and **Levente Polyak** as `unknown trust`. It also refused to reinstall `archlinux-keyring` for the same reason. The packages were not necessarily damaged; the local keyring did not trust their signing keys.

I corrected the keyring with:

```bash
pacman-key --init               # Initialize the local Pacman GPG keyring
pacman-key --populate archlinux # Import and trust the distribution's packaged key set
```

After this, package installation worked. I kept signature verification enabled rather than bypassing package security.

### OpenSSH and optional administration tools

```bash
pacman -Syu                                  # Update all installed packages
pacman -S openssh                            # Install the SSH server and client
systemctl enable --now sshd                  # Start SSH and enable it on future boots
systemctl status sshd                        # Check that the SSH daemon is active
```

**Verify before claiming completion:** The conversation confirms the keyring fix and successful installation, but I should still capture a screenshot of `systemctl is-active sshd` and a successful SSH login if I use those as evidence.

Optional tools used or planned for system administration:

```bash
pacman -S git curl wget vim htop tree tmux  # Useful command-line utilities
```

Optional regular user setup (run only if the account does not already exist):

```bash
id jjp                                      # Check whether the account already exists
useradd -m -G wheel -s /bin/bash jjp       # Create a non-root user with wheel group membership
passwd jjp                                 # Set the user's password
EDITOR=nano visudo                         # Uncomment: %wheel ALL=(ALL:ALL) ALL
```

From a Windows computer on the authorized course VPN, the SSH command is:

```powershell
ssh jjp@10.30.39.10   # Connect to the VM with the created non-root account
```

**Only mark optional commands as completed if you actually ran them successfully.**

## 8. Final checks and evidence

These are useful verification commands and screenshots for the walkthrough video and assignment:

```bash
cat /etc/os-release                 # Show that this is Arch Linux
uname -r                            # Show the installed kernel version
hostnamectl                         # Show the system hostname and platform
lsblk -f                            # Show disk partitions, filesystems, and mounts
ip -4 addr show ens18               # Show the static IPv4 address
ip route                            # Show the default route
resolvectl status                   # Show the DNS resolver
ping -c 3 archlinux.org             # Verify network and DNS
pacman -Q linux grub openssh        # Show key installed packages
systemctl is-active sshd            # Verify SSH is running
```

**Evidence checklist:** [ ] Installed Arch login; [ ] `lsblk -f`; [ ] static IP and route; [ ] successful `ping`; [ ] GRUB boot; [ ] `pacman -Q`; [ ] SSH status and remote login (if completed).

## 9. Lessons learned

The biggest lessons were to check the target mount before `pacstrap`, identify BIOS versus UEFI before choosing a bootloader, distinguish the live installer network from the installed system's network, and use Pacman's official keyring repair process when package signatures show `unknown trust`. Recording both mistakes and recovery commands makes this guide useful if I need to rebuild the VM for a future lab or walkthrough video.

## References

- [Arch Linux installation guide](https://wiki.archlinux.org/title/Installation_guide)
- [ArchWiki: GRUB](https://wiki.archlinux.org/title/GRUB)
- [GitHub Docs: Configuring a publishing source for GitHub Pages](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site)
