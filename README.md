# How to install Arch Linux from scratch

## 1. Connect Wifi

```bash
iwctl
station wlan0 connect "SSID"
exit
```

## 2. Enable SSH

```bash
systemctl enable sshd
# Create root password
passwd
```

You can now connect to your Arch Linux machine using SSH:

```bash
ssh root@<ip>
```

## 3. Partition the disk

If Windows is already installed, EFI partition is already created. Only the root partition is needed.

Otherwise, create a new GPT partition table for EFI and root partitions.

```bash
gdisk /dev/nvme0n1
```

Inside of gdisk, you can print the table using the `p` command.

To create a new partition use the `n` command. The below table shows 
the disk setup I have for my primary drive

| partition | first sector | last sector | code |
|-----------|--------------|-------------|------|
| 1         | default      | +512M       | ef00 |
| 2         | default      | default     | 8309 |

## 4. Encryption

```bash
# Replace /dev/nvme0n1p2 with the root partition
cryptsetup luksFormat -v -s 512 -h sha512 /dev/nvme0n1p2
cryptsetup open /dev/nvme0n1p3 luks_lvm
```

## 5. Volume setup

```bash
pvcreate /dev/mapper/luks_lvm
vgcreate arch /dev/mapper/luks_lvm
```

Create a volume for your swap space. A good size for this is your disk space + 2GB. In my case 34G:
```bash
lvcreate -n swap -L 34G arch
```

Crate root volume:
```bash
lvcreate -n root -l +100%FREE arch
```

## 6. Format the partitions

If you are not in Dual Boot, you can skip the EFI partition formatting. Otherwise, format the EFI partition:
```bash
mkfs.fat -F32 /dev/nvme0n1p1 
```

Format the root partition:
```bash
mkfs.btrfs -L root /dev/mapper/arch-root
```

Setup Swap:
```bash
mkswap /dev/mapper/arch-swap
```

## 7. Mount the partitions

```bash
# Swap
swapon /dev/mapper/arch-swap
swapon -a

# Root
mount /dev/mapper/arch-root /mnt

# Create boot directory
mkdir -p /mnt/boot

# Mount EFI
mount /dev/nvme0n1p1 /mnt/boot
```

## 8. Install the base system

```bash
# Install the base system
pacstrap -K /mnt base base-devel linux linux-firmware

# Generate fstab
genfstab -U -p /mnt > /mnt/etc/fstab

# Chroot into the new system
arch-chroot /mnt
```

## 9. Install packages

```bash
pacman -S vim lvm2 curl zsh networkmanager gnome intel-ucode cups hplip bluez pipewire pipewire-alsa pipewire-pulse apparmor git alacritty zoxide stow python python-pip python-virtualenv python-virtualenvwrapper
systemctl enable NetworkManager
systemctl enable gdm
```

## 10. MKINITCPIO

edit `/etc/mkinitcpio.conf` and set the `HOOKS` and `MODULES` line to:

```
MODULES=(crc32c-intel)
HOOKS=(base systemd autodetect keyboard sd-vconsole modconf block sd-encrypt lvm2 filesystems resume fsck)
```

## 11. Bootloader

```bash
bootctl install
```

Copy UUID of the root partition:
```bash
echo blkid -s UUID -o value /dev/nvme0n1p2
```

Create a new file `/boot/loader/entries/arch.conf` with the following content:
```
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options cryptdevice=UUID=<UUID_OF_ROOT_PARTITION>:luks_lvm root=/dev/mapper/arch-root resume=/dev/mapper/arch-swap rw quiet lsm=lockdown,yama,apparmor,bpf
```

Create a new file `/boot/loader/loader.conf` with the following content:
```
default arch.conf
timeout 3
console-mode max
editor no
```

## 12. Configure the system

```bash
# Change root password
passwd

# Timezone
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime

# Hardware clock
hwclock --systohc

# Locale
sed -i 's/# en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/g' /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# Hostname
hostname=myhostname
echo $hostname > /etc/hostname
echo -e "127.0.0.1 localhost\n::1 localhost\n127.0.1.1 $hostname.localdomain $hostname" > /etc/hosts

# Keymap
echo "KEYMAP=fr" > /etc/vconsole.conf

# Create a new user
useradd -m -G wheel -s /bin/zsh user
passwd user

# Allow the wheel group to execute any sudo command using:
vim /etc/sudoers

# If you are using a dual boot:
timedatectl set-local-rtc 1 --adjust-system-clock

# Install yay
su user
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ..
rm -rf yay

# Reboot
sudo reboot
```

## 13. Secure Boot

Put your BIOS Secure Boot in "Setup Mode" and enroll the keys.

```bash
sbctl sign -s /boot/vmlinuz-linux
sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi
sbctl sign -s /boot/EFI/Boot/bootx64.efi
sbctl sign -s /boot/EFI/Microsoft/Boot/bootmgfw.efi # For dual boot
sbctl sign -s /boot/EFI/Microsoft/Boot/bootmgr.efi # For dual boot
sbctl list-files
sbctl enroll-keys --microsoft # For dual boot
sbctl enroll-keys # For Arch Linux only
sbctl status
sbctl verify
reboot
```

## 14. TPM2

Edit /etc/crypttab.initramfs
```
root UUID=<UUID_OF_ROOT_PARTITION> none tpm2-device=auto
```

Enroll key to the TPM2:
```bash
systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0,7 /dev/nvme0n1p2
reboot
```

## 15. Final steps

Install Hack Nerd Font from [Nerd Fonts](https://www.nerdfonts.com/font-downloads)

```bash
# Install homyzsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

#Install zsh theme and plugins
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-autosuggestions.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions

# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Install lsd
cargo install lsd

# Install .fzf
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install

# Install dotfiles
git clone git@github.com:lchagnoleau/.dotfiles.git
cd .dotfiles
stow --adopt .

# Install Docker
sudo pacman -S docker
sudo systemctl start docker.service
sudo systemctl enable docker.service
sudo usermod -aG docker $USER
```

## 16. Usefull links

https://blog.sergeantbiggs.net/posts/booting-arch-linux-like-its-2012/

https://lemmy.ml/post/61254

https://github.com/dreamsofautonomy/arch-from-scratch
