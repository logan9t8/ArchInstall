# Installing Arch Linux (With KDE)

## Download ISO and signature 
https://archlinux.org/download

## Verify ISO
```powershell
gpg --keyserver-options auto-key-retrieve --verify archlinux-<version>-x86_64.iso.sig
```

## Pre-Install
```text
Disable FastBoot & Hibernation in Windows (If dual booting)
Disable SecureBoot in UEFI
```

## Burn & boot the ISO
https://rufus.ie/en/

## Verify the boot mode
```shell
ls /sys/firmware/efi/efivars
```

## WiFi
`iwctl`

```text
device list
station <device> scan
station <device> get-networks
station <device> connect <SSID>
exit
```

`ping archlinux.org`


## System clock
```shell
timedatectl list-timezones | less
timedatectl set-timezone <zone>
timedatectl set-ntp true
timedatectl status
```

## Format boot partition
```shell
fdisk -l
mkfs.ext4 <root_partition>
mount /dev/<root_partition> /mnt
```

## Update keyring
```shell
pacman -Sy
pacman -S archlinux-keyring
```

## Bootstrap packages
```shell
pacstrap /mnt base diffutils efibootmgr exfatprogs intel-ucode linux linux-firmware \
man-db man-pages memtest86+ ntfs-3g openssh reflector sudo tcpdump tmux usbutils vim zsh
```

## Mount additional data partitions (if required)
```shell
mkdir /mnt/mnt/<dir>
mount /mnt/mnt/<dir> /dev/<data_partition>
```

## Generate and verify fstab
```shell
genfstab -L /mnt >> /mnt/etc/fstab
vim /mnt/etc/fstab #Check config and add below content
    /dev/<root_partition> / ext4 rw,relatime 0 1
    /dev/<data_partition> /mnt/<dir> ntfs-3g defaults,uid=1000,gid=1000,fmask=027,dmask=027,windows_names 0 0
```

## Unmount additional data partitions (if mounted)
```shell
umount /mnt/<dir>
```

## Chroot
```shell
arch-chroot /mnt
```

## Set / permissions
```shell
chmod 755 /
```

## Set root 
```shell
passwd
```

## System config
```shell
ln -sf /usr/share/zoneinfo/<region/city> /etc/localtime
hwclock --systohc
sed -i '/#en_US.UTF-8 UTF-8/s/^#//g' /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
echo "<hostname>" > /etc/hostname

cat >> /etc/hosts << EOF
127.0.0.1    localhost
::1          localhost
127.0.1.1    <hostname>.localdomain  <hostname>
EOF
```

## Install make utils
```shell
pacman -S base-devel
```

## Install LSB packages explicitly, if required
```shell
pacman -S bc nss perl time
pacman -S gtk3 gtk4 libxslt qt5-base qt6-base xorg # Desktop installation
```

## Network management
```shell
pacman -S networkmanager
systemctl enable NetworkManager
```

## DNS
```shell
cat > /etc/NetworkManager/conf.d/dns-servers.conf << EOF
[global-dns-domain-*]
servers=1.1.1.1,1.0.0.1
EOF
```

## DNSSEC
```shell
pacman -S dnsmasq
# Do NOT enable dnsmasq service.
# NetworkManager will handle dnssec in the below method

cat > /etc/NetworkManager/dnsmasq.d/dnssec.conf << EOF
conf-file=/usr/share/dnsmasq/trust-anchors.conf
dnssec
domain-needed
bogus-priv
dnssec-check-unsigned
EOF

cat > /etc/NetworkManager/conf.d/dns.conf << EOF
[main]
dns=dnsmasq
EOF
```

## Grub
```shell
mkdir /efi
mount /dev/<efi_partition> /efi
# mount windows partition, if any to create windows entry
pacman -S grub efibootmgr os-prober
vim /etc/default/grub # Add below line
    GRUB_DISABLE_OS_PROBER=false
grub-install --efi-directory=/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

## Add user
```shell
visudo /etc/sudoers  # Uncomment the below line in the file
  %wheel    ALL=(ALL:ALL) ALL
useradd -m -G wheel -s /usr/bin/zsh <username>
passwd <username>
```

## Add user dirs
```shell
pacman -S xdg-user-dirs
xdg-user-dirs-update
```

## Display
```shell
lspci -v | grep -A1 -e VGA -e 3D
pacman -S xf86-video-intel
pacman -S intel-media-driver # For hardware acceleration
pacman -S xorg
pacman -S plasma-meta
pacman -S xdg-desktop-portal # Desktop integration for sandboxed apps
pacman -S colord-kde # Color correction
pacman -S wayland plasma-wayland-session # Wayland support
systemctl enable sddm
```
NOTE: Install a terminal emulator before rebooting\
Reboot

## Disable annoyance
Turn off 'restore plasma session on logon' in kde settings
```shell
touch ~/.hushlogin
sudo ln -s /usr/bin/vim /usr/bin/vi
```

## Enable user sudo for gui applications - kdesu
```shell
mkdir ~/.config
cat > ~/.config/kdesurc << EOF
[super-user-command]
super-user-command=sudo
EOF
```

##  Enable required pacman options
```shell
sudo sed -ri '/HookDir|Color|ParallelDownloads/s/^#//g' /etc/pacman.conf
```

## Pacman related packages
```shell
sudo pacman -S pacman-contrib pacutils
```

## Zsh rehash hook
```shell
sudo mkdir /etc/pacman.d/hooks
sudo tee /etc/pacman.d/hooks/zsh-rehash.hook 1> /dev/null << EOF
[Trigger]
Type = Package
Operation = Install
Target = *

[Action]
When = PostTransaction
Exec = /usr/bin/zsh -c "zstyle ':completion:*' rehash true"
EOF
```

## Refresh mirror list
```shell
sudo reflector --save /etc/pacman.d/mirrorlist --protocol https --latest 50 --sort rate
sudo pacman -Syyu
```

## Command not found
```shell
sudo pacman -S pkgfile
sudo systemctl enable --now pkgfile-update.timer
sudo pkgfile -u
```

## Firewall
```shell
sudo pacman -S ufw gufw
sudo systemctl enable --now ufw
sudo ufw enable
```

## Install extra fonts
```shell
sudo pacman -S noto-fonts-emoji
sudo pacman -S noto-fonts-cjk # Chinese, Japanese, Korean logograms ~300MB
```

## Bluetooth
```shell
sudo pacman -S bluez bluez-utils
sudo systemctl enable bluetooth
```

## Discover backend
```shell
sudo pacman -S packagekit-qt5
sudo pacman -S appstream # Reinstall when already installed
echo "Hidden=true" | sudo tee -a /etc/xdg/autostart/org.kde.discover.notifier.desktop 1> /dev/null
```

## Enable print (to pdf)
```shell
sudo pacman -S cups cups-pdf system-config-printer
sudo systemctl enable cups.socket
echo "Out /home/${USER}/Documents" | sudo tee -a /etc/cups/cups-pdf.conf 1> /dev/null
```

## Wallpapers
```shell
pacman -S archlinux-wallpaper
sudo ln -s /usr/share/backgrounds/archlinux /usr/share/wallpapers/archlinux
```

## Firefox
```shell
sudo pacman -S firefox plasma-browser-integration
xdg-settings set default-web-browser firefox.desktop # Set firefox as default to open links
```
```text
Set flags in about:config
    
    #Hardware acceleration
    
    media.ffmpeg.vaapi.enabled to true
    media.ffvpx.enabled to false
    media.rdd-vpx.enabled to false
    media.navigator.mediadatadecoder_vpx_enabled to true
    gfx.x11-egl.force-enabled to true
    gfx.x11-egl.force-disabled to false

    #XDG portal (Breaks with default firefox profile in firejail)
    
    widget.use-xdg-desktop-portal to true
```
```shell
sudo intel_gpu_top # Verify harware acceleration
```

## Media keys setup if KDE's 'Media Player' is disabled
https://wiki.archlinux.org/title/MPRIS
```shell
sudo pacman -S playerctl
```
```text
KDE Settings -> Custom Shortcuts -> Edit -> New Group -> 'playerctl'
Edit -> New -> Global Shortcut -> Command/URL -> 'Play/Pause' ; Action='playerctl play-pause' ; Trigger='Media Play'
Edit -> New -> Global Shortcut -> Command/URL -> 'Previous'   ; Action='playerctl previous'   ; Trigger='Media Previous'
Edit -> New -> Global Shortcut -> Command/URL -> 'Next'       ; Action='playerctl next'       ; Trigger='Media Next'
Override default keybindings when setting trigger
```

## Install yay
```shell
sudo pacman -S git
git clone https://aur.archlinux.org/yay-bin.git
cd yay-bin
makepkg -si
```

## Silent Boot - https://wiki.archlinux.org/index.php/Silent_boot
```shell
sudoedit /etc/default/grub
    GRUB_TIMEOUT=0 # -1 for dual boot system
    GRUB_CMDLINE_LINUX_DEFAULT="quiet splash loglevel=3 i915.fastboot=1"
sudo grub-mkconfig -o /boot/grub/grub.cfg
sudo sed -i 's/echo/#echo/g' /boot/grub/grub.cfg
```

## Plymouth - https://wiki.archlinux.org/index.php/plymouth
```shell
yay -S plymouth
sudoedit /etc/mkinitcpio.conf
# Add module i915
# Replace hooks 'base udev' with 'systemd'
# Add hook sd-plymouth
sudo mkinitcpio -P

systemctl disable sddm
systemctl enable sddm-plymouth

sudo cp /usr/share/plymouth/arch-logo.png /usr/share/plymouth/themes/spinner/watermark.png
sudo plymouth-set-default-theme -R bgrt
```

## AppArmor
```shell
sudo pacman -S apparmor
sudo systemctl enable apparmor
sudoedit /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="lsm=lockdown,yama,apparmor,bpf"
sudo grub-mkconfig -o /boot/grub/grub.cfg
sudo sed -i 's/echo/#echo/g' /boot/grub/grub.cfg
# Reboot
aa-enabled
```

## FireJail
```shell
sudo pacman -S firejail firetools
sudo firecfg # Enable firejail for all apps (that have profiles) by default
sudo firecfg --clean # Reverses the above
sudo aa-enforce firejail-default
sudoedit /etc/firejail/firejail.config # Set below options in the file
    browser-allow-drm yes
    quiet-by-default yes
```

## Create firejail pacman hook 
https://wiki.archlinux.org/index.php/Firejail#Using_Firejail_by_default
```shell
sudo tee /etc/pacman.d/hooks/firejail.hook 1> /dev/null << EOF
[Trigger]
Type = Path
Operation = Install
Operation = Upgrade
Operation = Remove
Target = usr/bin/*
Target = usr/local/bin/*
Target = usr/share/applications/*.desktop

[Action]
Description = Configure symlinks in /usr/local/bin based on firecfg.config...
When = PostTransaction
Depends = firejail
Exec = /bin/sh -c 'firecfg >/dev/null 2>&1'
EOF
```

## EFI Boot Manager
```shell
sudo pacman -Syu --needed efibootmgr
sudo mount /dev/<efi_partition> /efi
```

## Enable Secureboot - Preloader (Option #1 MAY not work anymore)
The below MAY not work anymore\
https://git.savannah.gnu.org/cgit/grub.git/commit/?id=6fe755c5c07bb386fda58306bfd19e4a1c974c53
```shell
yay -S preloader-signed
sudo cp /usr/share/preloader-signed/{PreLoader,HashTool}.efi /efi/EFI/GRUB/
sudo cp /efi/EFI/GRUB/grubx64.efi /efi/EFI/GRUB/loader.efi
sudo efibootmgr --verbose --disk /dev/sd<disk letter> --part <ESP number> --create --label "PreLoader" --loader /EFI/GRUB/PreLoader.efi
cp /efi/EFI/Microsoft/Boot/bootmgfw.efi /mnt/<somewhere> # As backup
```
```text
Reboot (to UEFI settings)
Enable secure boot
Reboot
Message "Failed to start loader" - Press OK
Select "Enroll Hash"
Pick loader.efi and confirm
Reboot
Verify GRUB is chainloaded
```

## Enable Secureboot - Shim (Option #2)
```shell
yay -S shim-signed
sudo cp /usr/share/shim-signed/{shimx64,mmx64}.efi esp/EFI/GRUB/
sudo cp /efi/EFI/GRUB/grubx64.efi /efi/EFI/GRUB/loader.efi
sudo efibootmgr --verbose --disk /dev/sd<disk letter> --part <ESP number> --create --label "Shim" --loader /EFI/GRUB/shimx64.efi
cp /efi/EFI/Microsoft/Boot/bootmgfw.efi /mnt/<somewhere> # As backup
```

## Audio

```shell
sudo pacman -S pipewire easyeffects # pulseaudio & pulseffects replacements
```
Reboot (Very important)\
Open easyeeffects and close it (to create dirs and configs)

```shell
curl -fsSLO https://raw.githubusercontent.com/JackHack96/PulseEffects-Presets/master/install.sh
sed -i '/^check_installation$/c\PRESETS_DIRECTORY=$HOME/.config/easyeffects' ./install.sh
chmod +x ./install.sh && bash -c ./install.sh
```
Launch easyeffects -> Click Plugins -> Select Convolver plugin -> Click Impulses -> Enable preferred preset.\
Remove all other unneeded plugins # Devs removed checkboxes in favor of add/remove, WTF?\
Enable 'Start service at login' in hamburger button in title bar

```easyeffects --gapplication-service & disown```

## Swap
```shell
cat /sys/power/image_size #Hibernation size, by default 2/5 of RAM
sudo dd if=/dev/zero of=/swapfile bs=1M count=6554 status=progress # 2/5 of 16GB
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudoedit /etc/fstab # Add below line
	/swapfile none swap defaults 0 0
resume_offset=$(sudo filefrag -v /swapfile | awk '{ if($1=="0:"){print substr($4, 1, length($4)-2)} }')
sudoedit /etc/default/grub # Add below line
	GRUB_CMDLINE_LINUX="resume=<root_partition> resume_offset=<value>"
```

## swappiness
```shell
sysctl vm.swappiness # Current swappiness
echo "vm.swappiness=1" | sudo tee /etc/sysctl.d/99-swappiness.conf 1> /dev/null
```

## Always Hybrid Sleep
```shell
sudoedit /etc/systemd/sleep.conf # Add below lines
	suspend=hybrid-sleep
	SuspendMode=suspend
	SuspendState=disk
	hibernate=hybrid-sleep
	HibernateMode=suspend
	HibernateState=disk
```

## VScode
```shell
yay -S visual-studio-code-bin
pacman -S python-pip gnome-keyring
```

## Remove packages & cache
```shell
sudo paccache -rk1
sudo paccache -ruk0
yay -Qtdq | yay -Rns -
sudo pacman -Scc
```

## Clipboard
Set clipboard shortcut in KDE settings

## JournalSize
```shell
sudoedit /etc/systemd/journald.conf # Add below line
    SystemMaxUse=100M
sudo systemctl restart systemd-journald
journalctl --vacuum-size=100M
```

## Swap vim with gvim
```shell
sudo pacman -S gvim
```

## NVIDIA (with prime render offloading)
```shell
sudo pacman -S nvidia nvidia-prime nvidia-settings
sudo pacman -S opencl-nvidia #If required
sudoedit /etc/mkinitcpio.conf # Add below line
    MODULES = (nvidia nvidia_modeset nvidia_uvm nvidia_drm) #Append to i915
sudo mkinitcpio -P
sudo tee /etc/pacman.d/hooks/nvidia.hook 1> /dev/null << EOF
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia
Target=linux
# Change the linux part above and in the Exec line if a different kernel is used

[Action]
Description=Update Nvidia module in initcpio
Depends=mkinitcpio
When=PostTransaction
NeedsTargets
Exec=/bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esac; done; /usr/bin/mkinitcpio -P'
EOF
```

## NVIDIA NOTES
* Do not run nvidia-xconfig (which modifies /etc/x11/xorg.conf)
* Do not modify /etc/x11/xorg.conf /etc/x11/xorg.conf.d/ /usr/share/X11/xorg.conf.d
* The above setup is already configured for prime render offloading
* The 1 xorg process that nvidia-smi shows is more like a hook for running xorg apps in nvidia gpu, it is NOT xorg itself (Citation needed)
* PROOF: Check GPU utilization in 'NVIDIA X Server settings' in the below scenarios
* Nothing running - 0% Usage
* Running glxrears - 0% Usage
* Running prime-run glxrears - >0% Usage

## CUDA (if required)
```shell
sudo pacman -S cuda
sudo systemctl enable nvidia-persistenced.service
/opt/cuda/extras/demo_suite/deviceQuery #Verify result
```

## git
```shell
yay -S git-credential-manager-core-bin
git config --global user.name <Github_username>
git config --global user.email <Github_email>
git config --global credential.credentialStore secretservice
git clone https://github.com/<Github_username>/<Github_repo>
    Username: <Github_username>
    Password: <Github_Personal_Access_token>
git-credential-manager-core configure
```
