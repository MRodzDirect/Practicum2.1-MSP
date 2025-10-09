
***

# Arch Linux Installation Guide for Dell OptiPlex 7010 SFF

**Complete installation with LUKS2 encryption, LVM, Btrfs subvolumes, LXQt, BlackArch tools, AI/ML development stack, and virtualization support**

## System Specifications

- **Model**: Dell OptiPlex 7010 Small Form Factor
- **CPU**: Intel Core i5-3570 (Ivy Bridge, 3rd Gen)
- **Integrated GPU**: Intel HD Graphics (Xeon E3-1200 v2/3rd Gen Core processor Graphics Controller)
- **RAM**: 8 GB (upgradeable to 16 GB)
- **Storage**: /dev/sda (previously Windows 10 Pro, will be completely erased)
- **Original Partitions**: sda1 (100 MB), sda2 (16 MB), sda3 (120 GB), sda4 (530 MB)

## Table of Contents

1. [Pre-Installation](#pre-installation)
2. [Disk Partitioning Strategy](#disk-partitioning-strategy)
3. [LUKS2 Encryption Setup](#luks2-encryption-setup)
4. [LVM Configuration](#lvm-configuration)
5. [Btrfs Filesystems and Subvolumes](#btrfs-filesystems-and-subvolumes)
6. [Base System Installation](#base-system-installation)
7. [System Configuration](#system-configuration)
8. [Initramfs Configuration](#initramfs-configuration)
9. [GRUB Bootloader](#grub-bootloader)
10. [LXQt Desktop Environment](#lxqt-desktop-environment)
11. [Intel Graphics Drivers](#intel-graphics-drivers)
12. [BlackArch Repository and Tools](#blackarch-repository-and-tools)
13. [AI/ML/LLM Development Tools](#aimlllm-development-tools)
14. [Virtualization with QEMU/KVM](#virtualization-with-qemukvm)
15. [Additional Optimizations](#additional-optimizations)
16. [Post-Installation](#post-installation)
17. [Security Considerations](#security-considerations)
18. [Optional GPU Upgrade](#optional-gpu-upgrade)

***

## Pre-Installation

### Boot into Arch Linux Installer

1. Download the latest Arch Linux ISO from [archlinux.org](https://archlinux.org)
2. Create a bootable USB drive using `dd` or a tool like Rufus/Etcher
3. Boot the Dell OptiPlex 7010 from the USB drive
4. Ensure the system boots in UEFI mode (not legacy BIOS)

### Initial Setup Commands

```bash
# Verify UEFI boot mode (directory should exist)
ls /sys/firmware/efi/efivars

# Set keyboard layout (default is US)
loadkeys us

# Connect to the internet
# Wired connection should auto-connect via DHCP
ping -c 3 archlinux.org

# Update system clock
timedatectl set-ntp true

# Verify time synchronization
timedatectl status
```

***

## Disk Partitioning Strategy

### Partition Layout Decision

For a 120–128 GB disk with security, development, and virtualization requirements:

- **EFI System Partition (ESP)**: 512 MB (ef00)
- **/boot**: 1 GB (ef00 for UEFI compatibility)
- **LUKS Container**: Remaining space (~117 GB) containing LVM with:
  - **Root (/)**: 50 GB Btrfs with subvolumes
  - **Swap**: 16 GB (for hibernation with 16 GB RAM upgrade)
  - **Home (/home)**: ~51 GB Btrfs with subvolumes

### Partitioning Commands

```bash
# Ensure dm-crypt modules are loaded (recommended check)
modprobe dm-crypt
modprobe dm-mod

# Wipe all existing signatures and partition table
wipefs -a /dev/sda
sgdisk --zap-all /dev/sda

# Create new GPT partition table with three partitions
# Partition 1: EFI System Partition (512 MB, type ef00)
sgdisk -n1:0:+512M -t1:ef00 -c1:"EFI" /dev/sda

# Partition 2: Boot partition (1 GB, type ef00 for UEFI compatibility)
sgdisk -n2:0:+1G -t2:ef00 -c2:"BOOT" /dev/sda

# Partition 3: LUKS container (remaining space, type 8309 for Linux LUKS)
sgdisk -n3:0:0 -t3:8309 -c3:"PV" /dev/sda

# Verify partition layout
sgdisk -p /dev/sda

# Format EFI and boot partitions
mkfs.fat -F32 /dev/sda1
mkfs.ext4 -L BOOT /dev/sda2
```

**Note**: Type `8309` is the official GPT type GUID for Linux LUKS partitions, ensuring proper identification by system tools and partition managers.[1][2][3]

***

## LUKS2 Encryption Setup

### Create Encrypted Container

```bash
# Encrypt /dev/sda3 with LUKS2 (uses argon2id PBKDF by default)
cryptsetup luksFormat --type luks2 /dev/sda3

# Confirm with uppercase YES and enter a strong passphrase
# This passphrase will be required at every boot

# Open the encrypted container
cryptsetup open /dev/sda3 cryptpv

# Verify the mapping exists
ls /dev/mapper/cryptpv
```

**Security Note**: LUKS2 with argon2id provides strong protection against brute-force attacks. Back up the LUKS header for disaster recovery:

```bash
cryptsetup luksHeaderBackup /dev/sda3 --header-backup-file luks-header-backup.img
# Store this file on external media in a secure location
```

***

## LVM Configuration

### Create Physical Volume, Volume Group, and Logical Volumes

```bash
# Initialize the encrypted device as an LVM Physical Volume
pvcreate /dev/mapper/cryptpv

# Create Volume Group named 'vg0'
vgcreate vg0 /dev/mapper/cryptpv

# Create Logical Volumes
# Root: 50 GB (adjust for larger disks: 80-100 GB recommended for 256+ GB drives)
lvcreate -L 50G -n lv_root vg0

# Swap: 16 GB (matches planned RAM upgrade for hibernate support)
lvcreate -L 16G -n lv_swap vg0

# Home: remaining space
lvcreate -l 100%FREE -n lv_home vg0

# Verify LVM structure
pvdisplay
vgdisplay
lvdisplay
```

***

## Btrfs Filesystems and Subvolumes

### Format Logical Volumes with Btrfs

```bash
# Format root and home as Btrfs
mkfs.btrfs -L ROOT /dev/vg0/lv_root
mkfs.btrfs -L HOME /dev/vg0/lv_home

# Initialize swap
mkswap -L SWAP /dev/vg0/lv_swap
```

### Create Btrfs Subvolumes

Btrfs subvolumes enable independent snapshots, rollbacks, and flexible filesystem management.[4][5][6]

```bash
# Create root subvolume (@)
mount /dev/vg0/lv_root /mnt
btrfs subvolume create /mnt/@
umount /mnt

# Create home subvolume (@home)
mount /dev/vg0/lv_home /mnt
btrfs subvolume create /mnt/@home
umount /mnt
```

### Mount Filesystems with Optimized Options

```bash
# Mount root subvolume with compression and noatime
mount -o subvol=@,compress=zstd,noatime /dev/vg0/lv_root /mnt

# Create mount points
mkdir -p /mnt/{boot,home}

# Mount boot and EFI partitions
mount /dev/sda2 /mnt/boot
mkdir /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi

# Mount home subvolume
mount -o subvol=@home,compress=zstd,noatime /dev/vg0/lv_home /mnt/home

# Activate swap
swapon /dev/vg0/lv_swap

# Verify mount points
lsblk
```

**Mount Options Explained**:
- `subvol=@`: Mounts the @ subvolume as root
- `compress=zstd`: Transparent compression for space savings and performance
- `noatime`: Reduces write operations by not updating access times

***

## Base System Installation

### Install Base Packages

```bash
# Update mirror list for optimal download speeds
reflector --latest 20 --protocol https --sort rate --save /etc/pacman.d/mirrorlist

# Install base system with essential packages
pacstrap -K /mnt base base-devel linux linux-firmware intel-ucode \
  vim nano networkmanager grub efibootmgr btrfs-progs lvm2 \
  git wget curl man-db man-pages sudo

# Generate fstab
genfstab -U /mnt >> /mnt/etc/fstab

# Verify fstab entries
cat /mnt/etc/fstab

# Chroot into the new system
arch-chroot /mnt
```

**Key Packages**:
- `intel-ucode`: Microcode updates for Intel i5-3570 (Ivy Bridge) security and stability
- `btrfs-progs`: Btrfs filesystem utilities
- `lvm2`: LVM management tools
- `grub` + `efibootmgr`: UEFI bootloader

***

## System Configuration

### Timezone, Locale, and Hostname

```bash
# Set timezone (adjust for your location, e.g., America/Guayaquil)
ln -sf /usr/share/zoneinfo/UTC /etc/localtime
hwclock --systohc

# Configure locale
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# Set hostname
echo "arch-7010" > /etc/hostname

# Configure hosts file
cat >> /etc/hosts << EOF
127.0.0.1   localhost
::1         localhost
127.0.1.1   arch-7010.localdomain arch-7010
EOF
```

### User and Password Configuration

```bash
# Set root password
passwd

# Create user account
useradd -m -G wheel,video,audio,storage,power -s /bin/bash user
passwd user

# Enable sudo for wheel group
EDITOR=vim visudo
# Uncomment the line: %wheel ALL=(ALL:ALL) ALL

# Enable NetworkManager for boot
systemctl enable NetworkManager
```

***

## Initramfs Configuration

### Configure mkinitcpio Hooks

The initramfs must include hooks to unlock LUKS, activate LVM, and resume from encrypted swap.[7][8]

```bash
# Edit /etc/mkinitcpio.conf
nano /etc/mkinitcpio.conf
```

Modify the `HOOKS` line:

```ini
HOOKS=(base udev autodetect microcode modconf block encrypt lvm2 resume filesystems fsck)
```

**Hook Order Explanation**:
- `encrypt`: Unlocks LUKS container at boot
- `lvm2`: Activates LVM volumes
- `resume`: Enables hibernation from encrypted swap

```bash
# Regenerate initramfs
mkinitcpio -P
```

***

## GRUB Bootloader

### Install GRUB to EFI Partition

```bash
# Install GRUB for UEFI
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ARCH

# Get UUID of the LUKS partition
CRYPTUUID=$(blkid -s UUID -o value /dev/sda3)
echo "LUKS UUID: $CRYPTUUID"
```

### Configure GRUB Kernel Parameters

```bash
# Edit /etc/default/grub
nano /etc/default/grub
```

Modify the following lines:

```bash
# Set kernel parameters for LUKS, LVM, and resume
GRUB_CMDLINE_LINUX="cryptdevice=UUID=YOUR-UUID-HERE:cryptpv root=/dev/mapper/vg0-lv_root resume=/dev/mapper/vg0-lv_swap"

# Enable os-prober for future multi-boot setups
GRUB_DISABLE_OS_PROBER=false
```

Replace `YOUR-UUID-HERE` with the actual UUID from the `blkid` command output.

**Additional Parameters** (optional for Intel i5-3570):
- `intel_iommu=on`: Required for IOMMU/VT-d support (GPU passthrough)

### Generate GRUB Configuration (Dual Build)

```bash
# Primary GRUB config
grub-mkconfig -o /boot/grub/grub.cfg

# Secondary GRUB config (EFI fallback path)
grub-mkconfig -o /boot/efi/EFI/arch/grub.cfg
```

**Why Dual Configuration?**: Ensures GRUB configuration is available in both standard and EFI-specific paths for maximum boot reliability and future multi-OS support.[9][10][11]

***

## LXQt Desktop Environment

### Install LXQt with X11 and RDP Support

```bash
# Install Xorg, LXQt, SDDM display manager, and RDP stack
pacman -S xorg-server xorg-xinit lxqt sddm xrdp xorgxrdp

# Enable SDDM (display manager)
systemctl enable sddm

# Enable xrdp for remote desktop access
systemctl enable xrdp
```

### Configure RDP Session

```bash
# Set LXQt as default X session for user
echo "exec startlxqt" > /home/user/.xinitrc
chown user:user /home/user/.xinitrc

# Optional: Configure xrdp for better performance
# Edit /etc/xrdp/xrdp.ini and set:
# max_bpp=24
# crypt_level=high
```

### Firewall Configuration for RDP (Optional)

```bash
# Install and configure UFW firewall
pacman -S ufw
systemctl enable ufw

# Allow RDP port 3389
ufw allow 3389/tcp

# Allow SSH (if needed)
ufw allow 22/tcp

# Enable firewall
ufw enable
```

**RDP Access**: Connect from any RDP client (Windows Remote Desktop, Remmina, etc.) using the Arch system's IP address on port 3389.[12]

***

## Intel Graphics Drivers

### Install Intel GPU Stack for Ivy Bridge (3rd Gen i5-3570)

The Intel HD Graphics (Xeon E3-1200 v2/3rd Gen Core) uses the in-kernel `i915` driver with Mesa for 3D acceleration and `libva-intel-driver` (i965 VA-API stack) for hardware video decoding on Ivy Bridge architecture.[13][14][15]

```bash
# Install Mesa and Intel video acceleration
pacman -S mesa libva-intel-driver libvdpau-va-gl intel-gpu-tools

# Optional: Vulkan support (limited on Ivy Bridge but available)
pacman -S vulkan-intel vulkan-tools

# Verify VA-API driver installation
# After reboot, run: vainfo
# Expected output: i965 driver for Ivy Bridge
```

**Driver Notes**:
- **i915**: In-kernel DRM/KMS driver (automatically loaded)
- **libva-intel-driver**: Provides VA-API (i965) for Ivy Bridge hardware video acceleration
- **No xf86-video-intel needed**: The Xorg modesetting driver is recommended and enabled by default

### Set VA-API Environment Variable (Optional)

```bash
# Force i965 driver for VA-API (if needed)
echo "export LIBVA_DRIVER_NAME=i965" >> /etc/environment
```

***

## BlackArch Repository and Tools

### Add BlackArch Repository

```bash
# Download and verify the BlackArch setup script
curl -O https://blackarch.org/strap.sh

# Verify checksum (recommended)
sha1sum strap.sh
# Expected: See https://blackarch.org/downloads.html for current hash

# Make executable and run
chmod +x strap.sh
./strap.sh

# Update package database
pacman -Syy
```

### Install Beginner-Friendly Security Tools

```bash
# Install curated tool categories
pacman -S blackarch-networking blackarch-scanner blackarch-recon \
  blackarch-forensic blackarch-crypto

# Essential beginner penetration testing tools
pacman -S nmap wireshark-qt metasploit john hashcat aircrack-ng \
  hydra sqlmap burpsuite nikto dirb gobuster binwalk autopsy

# Additional useful tools
pacman -S tcpdump net-tools traceroute whois dnsutils
```

**Tool Categories**:[16][17][18]
- **blackarch-networking**: Network analysis and monitoring
- **blackarch-scanner**: Vulnerability scanners
- **blackarch-recon**: Reconnaissance tools
- **blackarch-forensic**: Digital forensics utilities
- **blackarch-crypto**: Cryptography and hashing tools

***

## AI/ML/LLM Development Tools

### Install Python and Machine Learning Frameworks

```bash
# Install Python and scientific computing stack
pacman -S python python-pip python-numpy python-scipy python-pandas \
  python-matplotlib python-scikit-learn jupyter-notebook

# Install development toolchain
pacman -S gcc gdb cmake make git docker docker-compose

# Enable Docker service
systemctl enable docker
usermod -aG docker user
```

### Install PyTorch and Deep Learning Libraries

```bash
# Install PyTorch (CPU-only for integrated graphics)
pacman -S python-pytorch python-torchvision python-torchaudio

# Alternative: Install TensorFlow
pacman -S python-tensorflow

# Install transformers and LLM tools via pip
pip install --user transformers datasets accelerate peft trl \
  llama-cpp-python langchain openai sentence-transformers
```

### Install Ollama for Local LLM Inference

```bash
# Install Ollama from AUR (requires yay helper, installed later)
# Manual install:
curl -fsSL https://ollama.com/install.sh | sh

# Start Ollama service
systemctl enable ollama
systemctl start ollama

# Pull a model (example: Llama 3)
ollama pull llama3
```

### Install Additional AI Development Tools

```bash
# Install AUR helper (yay)
cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si

# Install VS Code from AUR
yay -S visual-studio-code-bin

# Install Anaconda (optional for isolated environments)
yay -S anaconda
```

**AI/ML Workflow Tools**:[19][20][21]
- **Jupyter Notebook**: Interactive computing environment
- **Docker**: Containerized ML workflows
- **PyTorch/TensorFlow**: Deep learning frameworks
- **Ollama**: Local LLM inference engine
- **Transformers**: Hugging Face library for NLP/LLM tasks

***

## Virtualization with QEMU/KVM

### Install QEMU, KVM, and Management Tools

```bash
# Install virtualization packages
pacman -S qemu-full virt-manager libvirt edk2-ovmf bridge-utils \
  dnsmasq iptables-nft dmidecode

# Enable and start libvirt service
systemctl enable libvirtd
systemctl start libvirtd

# Add user to virtualization groups
usermod -aG libvirt user
usermod -aG kvm user
```

### Enable Nested Virtualization

```bash
# Enable nested virtualization for Intel CPUs
echo "options kvm_intel nested=1" | tee /etc/modprobe.d/kvm.conf
```

### Configure Default Network

```bash
# Start and autostart the default network
virsh net-autostart default
virsh net-start default
```

### Verify IOMMU Support (for GPU Passthrough)

```bash
# Check if IOMMU is enabled (after reboot)
dmesg | grep -e DMAR -e IOMMU

# Script to list IOMMU groups
#!/bin/bash
shopt -s nullglob
for d in /sys/kernel/iommu_groups/*/devices/*; do
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done
```

**Note**: The Dell OptiPlex 7010 supports VT-d (IOMMU), but GPU passthrough requires a discrete GPU in addition to the integrated Intel graphics. See the [Optional GPU Upgrade](#optional-gpu-upgrade) section for recommendations.[22][23][24]

***

## Additional Optimizations

### Install Useful Utilities

```bash
# System monitoring and utilities
pacman -S htop btop neofetch tmux zsh

# GUI applications
pacman -S firefox chromium vlc gimp libreoffice-fresh

# Fonts
pacman -S ttf-dejavu ttf-liberation noto-fonts noto-fonts-emoji ttf-hack
```

### Enable SSD TRIM (if applicable)

```bash
# Enable periodic TRIM for SSD longevity
systemctl enable fstrim.timer
```

### CPU Power Management (Optional)

```bash
# Install CPU frequency scaling tools
pacman -S cpupower

# Enable cpupower service
systemctl enable cpupower.service

# Set performance governor for AI/ML workloads
echo 'GOVERNOR="performance"' > /etc/default/cpupower

# Alternative: Use ondemand for balanced power/performance
# echo 'GOVERNOR="ondemand"' > /etc/default/cpupower
```

***

## Post-Installation

### Exit Chroot and Reboot

```bash
# Exit the chroot environment
exit

# Unmount all partitions
umount -R /mnt

# Turn off swap
swapoff -a

# Reboot
reboot
```

### First Boot

1. Remove the USB installer
2. System will boot and prompt for the LUKS passphrase
3. After unlocking, GRUB will load the kernel
4. SDDM login screen will appear
5. Log in with the user credentials created earlier
6. LXQt desktop environment will start

### Post-Boot Verification

```bash
# Check microcode is loaded
dmesg | grep microcode

# Verify Intel graphics driver
lspci -k | grep -A 3 VGA

# Check VA-API driver
vainfo

# Verify LUKS/LVM status
lsblk
lvs
vgs
pvs

# Check Btrfs subvolumes
btrfs subvolume list /
btrfs subvolume list /home

# Test RDP connection
# From another computer: rdp://YOUR_ARCH_IP:3389
```

***

## Security Considerations

### Strong Passphrase Strategy

- **LUKS Passphrase**: Use a strong passphrase (20+ characters, mixed case, numbers, symbols)
- **Avoid Keyfiles**: The current setup does not rely on keyfiles stored on disk, preventing unauthorized auto-unlock if the drive is stolen
- **Header Backup**: Store the LUKS header backup on external encrypted media

### Optional: TPM2-Assisted Unlock

The Dell OptiPlex 7010 may have TPM 1.2 or TPM 2.0 depending on BIOS revisions. If TPM 2.0 is available, LUKS can be enrolled with TPM for PIN-gated unlock:

```bash
# Check TPM version
cat /sys/class/tpm/tpm0/tpm_version_major

# If TPM 2.0, install systemd-cryptenroll
pacman -S tpm2-tools

# Enroll LUKS with TPM2 (requires unlocked system)
systemd-cryptenroll --tpm2-device=auto /dev/sda3

# Add a recovery passphrase
systemd-cryptenroll --recovery-key /dev/sda3
```

**Benefits**:
- Single PIN entry instead of long passphrase at boot
- TPM binding prevents unlock on different hardware
- Recovery passphrase available if TPM fails

### Optional: Secure Boot

To enable Secure Boot with custom keys:

1. Generate MOK (Machine Owner Keys)
2. Sign GRUB and kernel with your keys
3. Enroll keys in UEFI firmware
4. Enable Secure Boot in BIOS

**Resources**: [Arch Wiki - Secure Boot](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot)

### Additional Hardening

```bash
# Install AppArmor for mandatory access control
pacman -S apparmor

# Enable AppArmor
systemctl enable apparmor

# Install audit daemon
pacman -S audit
systemctl enable auditd

# Configure automatic updates (optional)
# Note: Not recommended for rolling release without testing
```

***

## Optional GPU Upgrade

### Best GPU for Intel i5-3570 (Ivy Bridge)

The Dell OptiPlex 7010 SFF has limited physical space and a 240W PSU. Suitable GPUs for this system include:

#### Budget Option: AMD Radeon RX 6400
- **TDP**: 53W (no external power connector required)
- **Performance**: 1080p gaming, light ML workloads
- **Driver**: mesa, xf86-video-amdgpu
- **Compatibility**: Low-profile bracket required for SFF

#### Mid-Range Option: NVIDIA GTX 1650 (Low Profile)
- **TDP**: 75W (no external power connector required)
- **Performance**: 1080p gaming, CUDA support for ML
- **Driver**: nvidia-dkms
- **Compatibility**: Low-profile available, excellent CUDA support

#### High-End Option: NVIDIA GTX 1660 Super (requires PSU upgrade)
- **TDP**: 125W (requires 6-pin PCIe power)
- **Performance**: 1440p capable, strong CUDA performance
- **Driver**: nvidia-dkms
- **Compatibility**: Requires PSU upgrade to 400W+

### Installing NVIDIA Drivers

```bash
# Install NVIDIA proprietary drivers
pacman -S nvidia-dkms nvidia-utils lib32-nvidia-utils

# Install CUDA toolkit for ML workloads
pacman -S cuda cudnn

# Rebuild initramfs with nvidia module
mkinitcpio -P

# Add nvidia modules to mkinitcpio
nano /etc/mkinitcpio.conf
# MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)

# Rebuild again
mkinitcpio -P

# Reboot
reboot
```

### Installing AMD Drivers

```bash
# Install AMD open-source drivers (usually pre-installed)
pacman -S mesa xf86-video-amdgpu vulkan-radeon lib32-vulkan-radeon

# Install ROCm for AMD compute (AI/ML workloads)
pacman -S rocm-hip-sdk rocm-opencl-sdk

# Reboot
reboot
```

### Configuring GPU Passthrough

For GPU passthrough to VMs, a discrete GPU is required in addition to integrated graphics:

```bash
# Identify GPU PCI IDs
lspci -nn | grep -i vga

# Create vfio configuration (replace IDs with your GPU)
echo "options vfio-pci ids=10de:xxxx,10de:yyyy" > /etc/modprobe.d/vfio.conf

# Update mkinitcpio to load vfio early
nano /etc/mkinitcpio.conf
# MODULES=(vfio_pci vfio vfio_iommu_type1 vfio_virqfd nvidia nvidia_modeset nvidia_uvm nvidia_drm)

# Regenerate initramfs
mkinitcpio -P

# Reboot
reboot
```

**Note**: GPU passthrough requires the discrete GPU to be isolated from the host and assigned to a VM. The integrated Intel GPU will handle the host display.[23][24][22]

***

## Troubleshooting

### GRUB Not Detecting Other OSes

If os-prober doesn't detect a second OS after installation:

```bash
# Ensure os-prober is installed
pacman -S os-prober

# Mount the other OS's partition
mkdir /mnt/other
mount /dev/sdX /mnt/other

# Regenerate GRUB config
grub-mkconfig -o /boot/grub/grub.cfg
grub-mkconfig -o /boot/efi/EFI/arch/grub.cfg

# Unmount
umount /mnt/other
```

### RDP Login Issues

If xrdp doesn't start properly:

```bash
# Check xrdp service status
systemctl status xrdp

# View xrdp logs
journalctl -u xrdp -xe

# Ensure .xinitrc is correct
cat ~/.xinitrc
# Should contain: exec startlxqt

# Restart xrdp service
systemctl restart xrdp
```

### Intel Graphics Driver Issues

If graphics performance is poor:

```bash
# Verify i915 module is loaded
lsmod | grep i915

# Check Xorg is using modesetting driver
grep LoadModule /var/log/Xorg.0.log

# Force i965 VA-API driver
export LIBVA_DRIVER_NAME=i965
vainfo
```

***

## References

- [Arch Linux Installation Guide](https://wiki.archlinux.org/title/Installation_guide)[25]
- [dm-crypt/Encrypting an entire system](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system)[7]
- [LVM on LUKS](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS)[8]
- [Btrfs Subvolumes](https://wiki.archlinux.org/title/Btrfs#Subvolumes)[5][4]
- [Intel Graphics](https://wiki.archlinux.org/title/Intel_graphics)[15][13]
- [BlackArch Linux](https://blackarch.org)[16]
- [PCI Passthrough via OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)[23]
- [GRUB os-prober](https://wiki.archlinux.org/title/GRUB#Detecting_other_operating_systems)[10][9]

***

## License

This guide is provided as-is for educational purposes. Follow at your own risk.

## Contributing

Feel free to submit issues or pull requests to improve this guide.

***

**Last Updated**: October 2025  
**Tested Hardware**: Dell OptiPlex 7010 SFF (Intel Core i5-3570, 8GB RAM, 128GB SSD)  
**Arch Linux Kernel**: 6.x series

[1](https://bbs.archlinux.org/viewtopic.php?id=256115)
[2](https://dev.to/ivanmoreno/how-to-encrypt-an-existing-archlinux-lvm-installation-lvm-on-luks-1800)
[3](https://wiki.archlinux.org/title/GPT_fdisk)
[4](https://www.reddit.com/r/btrfs/comments/psmxbb/creating_subvolume_for_home_directory_best/)
[5](https://fedoramagazine.org/working-with-btrfs-subvolumes/)
[6](https://www.jwillikers.com/btrfs-layout)
[7](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system)
[8](https://wiki.archlinux.org/title/LVM)
[9](https://wiki.archlinux.org/title/GRUB)
[10](https://forum.endeavouros.com/t/warning-os-prober-will-not-be-executed-to-detect-other-bootable-partitions-systems-on-them-will-not-be-added-to-the-grub-boot-configuration-check-grub-disable-os-prober-documentation-entry/13998)
[11](https://plug-world.com/posts/grub-and-os-prober-in-arch-linux/)
[12](https://wiki.archlinux.org/title/Xrdp)
[13](https://wiki.archlinux.org/title/Hardware_video_acceleration)
[14](https://forum.manjaro.org/t/va-api-driver-broken/30529)
[15](https://www.siberoloji.com/how-to-install-and-configure-intel-graphics-drivers-on-arch-linux/)
[16](https://blackarch.org)
[17](https://blackarch.org/tools.html)
[18](https://linuxsecurity.com/howtos/secure-my-network/how-to-install-blackarch-tools-on-arch-linux)
[19](https://dev.to/kurealnum/an-actually-productive-arch-linux-setup-2d62)
[20](https://www.unixmen.com/ai-software-for-linux-which-linux-ai-tools-are-best-in-2025/)
[21](https://www.tecmint.com/linux-tools-for-ai-development/)
[22](https://gist.github.com/MaxXor/e24094f2b0624cf702f534f1a0dea0be)
[23](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)
[24](https://github.com/chikobara/guide-QEMU-KVM-GPU-Passthrough)
[25](https://wiki.archlinux.org/title/Installation_guide)
[26](https://forum.garudalinux.org/t/in-gdisk-what-exactly-does-8308-8309-do/13556)
[27](https://www.reddit.com/r/linuxquestions/comments/18y10kl/do_partition_types_actually_do_anything/)
[28](https://forums.freebsd.org/threads/very-odd-installation-question.85505/)
[29](https://discourse.maas.io/t/handling-multiple-uefi-esp-partition/7830)
[30](https://bbs.archlinux.org/viewtopic.php?id=267225)
[31](https://bbs.archlinux.org/viewtopic.php?id=189296)
[32](https://community.clearlinux.org/t/clear-linux-install-lvm-on-luks/4441)
[33](https://docs.hetzner.com/robot/dedicated-server/operating-systems/efi-system-partition/)
[34](https://discussion.fedoraproject.org/t/i-want-to-create-a-partition-subvolume-to-hold-my-personal-data/147246)
[35](https://www.reddit.com/r/archlinux/comments/12xwvdp/should_partition_type_be_linux_filesystem_or/)
[36](https://forum.manjaro.org/t/btrfs-with-home-on-separate-disk/157639)
[37](https://forums.opensuse.org/t/dual-booting-in-uefi-systems-with-two-different-hard-drives/138624)
[38](https://discuss.cachyos.org/t/howto-preserve-btrfs-home-volume-at-installation/315)
[39](https://fitzcarraldoblog.wordpress.com/2017/02/10/partitioning-hard-disk-drives-for-bios-mbr-bios-gpt-and-uefi-gpt-in-linux/)
