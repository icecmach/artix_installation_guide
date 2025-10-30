###### 01 August 2025

# Artix Install [btrfs + encryption + dinit + X11Libre + i3]

These are the installation instructions used for the Artix system that I use on my own machine.
Always consult the official Artix wiki [install guide](https://wiki.artixlinux.org/Main/Installation) as well the Arch [install guide](https://wiki.archlinux.org/title/Installation_guide),
sometimes you may find your preferences deviating from the the official guide, and so my intention here is to
provide a walkthrough on setting up your own system with the following:

- [btrfs](https://wiki.archlinux.org/title/Btrfs): modern copy on write (COW) file system
- [encryption](https://gitlab.com/cryptsetup/cryptsetup/): LUKS disk encryption based on the dm-crypt kernel module
- [dinit](https://github.com/davmac314/dinit/blob/master/doc/getting_started.md): service manager and init system
- [X11Libre](https://github.com/X11Libre/xserver): XLibre is a display server implementation of the X Window System Protocol Version 11 (X11)
- [i3-wm](https://i3wm.org/): Improved tiling window manager

## Step 1: Creating a bootable Artix media device

1. Acquire an artix-base-dinit ISO [here](https://artixlinux.org/download.php)
2. Write your ISO to a USB (check out [this](https://www.scaler.com/topics/burn-linux-iso-to-usb/) guide)

```bash
sudo dd bs=4M if=./artix-base-dinit-20250728-x86_64.iso of=/dev/sdb status=progress oflag=sync
```

3. Boot into the USB

## Step 2: Setting Up Our System with the Artix ISO

1. Set the console keyboard layout (US by default):

```bash
# list available keymaps
ls -R /usr/share/kbd/keymaps | less
# load the keymap
loadkeys <your keymap here>
```

2. Verify the UEFI boot mode **cat /sys/firmware/efi/fw_platform_size**. This installation is written for a system with a 64-bit x64 UEFI.

3. Connect to wifi

```bash
# Many laptops have a hardware button (or switch) to turn off the wireless card; however, the card can also be blocked by the kernel
rfkill unblock wlan
# bring interface up
ip link set wlan0 up

# Connection using guided mode
wpa_cli
scan
scan_results
add_network
set_network 0 ssid "MYSSID"
set_network 0 psk "passphrase"
enable_network 0
save_config
quit

# Connection using manual mode
# Discover access points
iw dev wlan0 scan | less
# Connect to an access point (manually)
iw dev wlan0 connect MYSSID
wpa_passphrase MYSSID passphrase > /etc/wpa_supplicant/example.conf
wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/example.conf
dhcpcd wlan0

# Using NetworkManager
# List nearby Wi-Fi networks
nmcli device wifi list
# Connect to a Wi-Fi network:
nmcli device wifi connect SSID_or_BSSID password your_password_here

# Using nmtui
nmtui
```

- Confirm that your connection is active with **ping -c 2 artixlinux.org**

**[optional]** To ssh into the target machine you will need to:

4. Configure ssh

```bash
vi /etc/ssh/sshd_config
# add these lines
Port 22
PermitRootLogin yes
PasswordAuthentication yes
# start service
dinitctl start sshd
```

- Ensure that **ssh** is running with **dinitctl status sshd**

5. **[optional]** Obtain your IP Address with **ip addr show**, now you're ready to ssh into your target machine
6. Update the system clock

```bash
dinitctl start ntpd
```

7. Partition your disk:

- list your partitions with **lsblk**
- delete the existing partitions on the target disk **[WARNING: your data will be lost]**

```bash
cfisk /dev/nvme0n1
```

- create two partitions:

  > **NOTE:** The official Arch Linux installation guide suggests implementing a swap partition and you are welcome to take this route. You could also create a swap subvolume within BTRFS, however, snapshots will be disabled where a volume has an active swapfile. In my case, I have opted instead of `zram` which works by compressing data in RAM, thereby stretching your RAM further. zram is only active when your RAM is at capacity.

- **Partition 1**
  - **Type** = EFI System
  - **size** = 500M
- **Partition 2**
  - Allocate all remaining space (or as otherwise fit for your specific case) noting that BTRFS doesn't require pre-defined partition sizes, but rather allocates dynamically through subvolumes which act in a similar fashion to partitions but don't require the physical division of the target disk.

8. Format your main partition:

```bash
# setup encryption
cryptsetup luksFormat /dev/nvme0n1p2
# open your encrypted partition
cryptsetup luksOpen /dev/nvme0n1p2 main
# format your partition
mkfs.btrfs /dev/mapper/main
# mount your main partition for installation
mount /dev/mapper/main /mnt
# cd into the /mnt directory
cd /mnt
```

- create our subvolumes:

```bash
# root
btrfs subvolume create @
# home
btrfs subvolume create @home
```

> **NOTE:** you can create your own subvolume names, but make sure you know what you are doing, because these subvolumes will also be referenced later when taking snapshots.

```bash
# go back to the original (root) directory
cd
# unmount our mnt partition
umount /mnt
# mount root subvolume
mount -o noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@ /dev/mapper/main /mnt
# create home mounting point and mount home subvolume
mkdir /mnt/home
mount -o noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@home /dev/mapper/main /mnt/home
```

9. format your efi partition:

```bash
mkfs.fat -F32 /dev/nvme0n1p1
```

- create boot mounting point and mount efi partition:

```bash
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

10. Install base packages:

```bash
basestrap /mnt base base-devel dinit elogind-dinit linux linux-headers linux-firmware
```

11. generate the file system table:

```bash
fstabgen -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab
```

12. change root into the new system:

```bash
artix-chroot /mnt
```

## Step 3: Working Within Our New System

We are now working within our Artix system, but it's important to note that we can't yet reboot our machine. Let's continue with a few steps that we need to repeat again (such as setting our root password, timezones, keymaps and language) given the previous settings were in the context of our ISO.

1. set your local time and locale on your system:

```bash
ln -sf /usr/share/zoneinfo/America/Toronto /etc/localtime
hwclock --systohc
pacman -S vim
```

- Execute **vim /etc/locale.gen** and uncomment your locale, example: **en_CA.UTF-8 UTF-8** write and exit and then run:

```bash
locale-gen
# for locale
echo "LANG=en_CA.UTF-8" >> /etc/locale.conf
# for keyboard
echo "KEYMAP=en..." >> /etc/vconsole.conf
```

2. change the hostname

```bash
echo "Artix" >> /etc/hostname
```

3. set your root password: **passwd**
4. set up a new user (replace **ice** with your preferred username):

```bash
useradd -m -g users -G wheel ice
# give your user a password with
passwd ice
# add your user to the sudoers group
pacman -S sudo
echo "ice ALL=(ALL) ALL" >> /etc/sudoers.d/ice
```

Next, we will install all of the packages we need for our system. Refer to the bottom of this guide for a short summary on each package being installed. It's imperative to always know what you are doing, and what you are installing!

6. install the main packages that our system will use:

```bash
pacman -Syu acpid acpid-dinit alsa-utils bluez bluez-dinit bluez-utils btrfs-progs \
cryptsetup dhcpcd efibootmgr git grub grub-btrfs ipset iptables-nft iw iwd \
mtools networkmanager networkmanager-dinit openntpd-dinit openssh openssh-dinit os-prober \
pacman-contrib pipewire pipewire-dinit pipewire-jack pipewire-pulse \
sof-firmware ufw wireplumber wireplumber-dinit wpa_supplicant
```

7. install the following based on the manufacturer of your CPU:

- **Intel:**
  ```bash
  pacman -S intel-ucode
  ```
- **AMD:**
  ```bash
  pacman -S amd-ucode
  ```

8. install **XLibre** + **i3-wm**:

```bash
# If using Nvidia
pacman -S nvidia
```

```bash
pacman -S xlibre-xserver xlibre-xserver-{common,devel,xvfb} \
xlibre-xf86-input-{libinput,evdev,vmmouse} \
xorg-{xdpyinfo,xinit,xmodmap,xprop,xrandr,xset,xsetroot} \
# AMD
xlibre-xf86-video-{vesa,amdgpu,fbdev,ati,dummy}
# Intel
xlibre-xf86-video-{vesa,intel,fbdev,dummy}
```

> **NOTE:** To enable loading of the proprietary Nvidia driver, please add the following to your X configuration, e.g., **/etc/X11/xorg.conf**

```
Section "ServerFlags"
    Option "IgnoreABI" "1"
EndSection
```

> **NOTE:** Any tiling window manager or graphical user environment can be installed at this stage.

9. install i3-wm and other useful packages:

```bash
pacman -S alacritty dmenu i3-wm i3status i3lock man-db man-pages \
7zip btop eza feh fzf newsboat starship stow unzip zed zoxide \
noto-fonts-cjk noto-fonts-emoji noto-fonts otf-commit-mono-nerd texinfo ttf-firacode-nerd
```

10. edit the mkinitcpio file for encrypt:

- **vim /etc/mkinitcpio.conf**
  - add `btrfs` to the MODULES: **MODULES=(btrfs)**
  - add `encrypt` to HOOKS (before filesystems): **encrypt filesystems fsck)**
- recreate the mkinitcpio.conf with **mkinitcpio -p linux**

11. setup grub for the bootloader so that the system can boot linux:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

- run **blkid** and obtain the UUID for the main partition
- edit the grub config **vim /etc/default/grub**
- **GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=c35cf57d-bfe0-4587-b89f-2594dc95b120:main root=/dev/mapper/main**
- make the grub config with **grub-mkconfig -o /boot/grub/grub.cfg**

12. enable services:

```bash
ln -s /etc/dinit.d/NetworkManager /etc/dinit.d/boot.d/
ln -s /etc/dinit.d/bluetoothd /etc/dinit.d/boot.d/
ln -s /etc/dinit.d/sshd /etc/dinit.d/boot.d/
ln -s /etc/dinit.d/acpid /etc/dinit.d/boot.d/
ln -s /etc/dinit.d/ntpd /etc/dinit.d/boot.d/
exit
```

Now for the moment of truth. Make sure you have followed these steps above carefully, then reboot your system with the **reboot** command.

## Step 4: Tweaking our new Artix system

When you boot up you will be presented with the grub bootloader menu, and then, once you have selected to boot into arch linux (or the timer has timed out and selected your default option) you will be prompted to enter your encryption password. Upon successful decryption, you will be presented with the login screen. Enter the password for the user you created earlier.

1. Install [yay](https://github.com/Jguer/yay) to get access to AUR packages

```bash
sudo pacman -S --needed base-devel
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
yay -Y --gendb
yay -Syu --devel
yay -Y --devel --save
```

2. Pacman hooks

Everytime a package is installed or removed it updates a file with the list of installed packages in the system

```bash
sudo mkdir /etc/pacman.d/hooks
sudo vim /etc/pacman.d/hooks/pkglist.hook

# add content below, replace 'username'
[Trigger]
Operation = Install
Operation = Remove
Type = Package
Target = *
[Action]
When = PostTransaction
Exec = /bin/sh -c "/usr/bin/pacman -Qqet > /home/username/pkglist.txt"
```

3. Sensors for hardware monitoring, temperatures and fan speed

Install [lm_sensors](https://wiki.archlinux.org/title/Lm_sensors)

```bash
sudo pacman -S lm_sensors
###
# Asrock B650M Pro RS / B850M Pro RS / X870 Pro RS
sudo echo "options nct6775 force_id=0xd801" > /etc/modprobe.d/nct6775.conf
sudo modprobe nct6775
sudo sensors-detect --auto
###
# Check output of sensors
sensors
# Load nct6775.ko at boot
sudo echo "nct6775" > /etc/modules-load.d/nct6775.conf
```

4. Update repository mirrorlist

**rankmirrors** command outputs a list of the fastest mirrors for your location

You need to copy the output and replace the default mirrors in /etc/pacman.d/mirrorlist

```bash
rankmirrors -v -n 5 -m 2 -w /etc/pacman.d/mirrorlist
```
