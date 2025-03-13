Tested on my system76 Oryx Pro (oryp8)

https://tech-docs.system76.com/models/oryp8/README.html

The aim of this installation is to support hibernation with encrypted swap and btrfs snapshots

# Set up laptop
In BIOS, under security, set "Enforce secure boot" to "disabled".

# Get install environment working

Keyboard layout for boot env:

    localectl list-keymaps | grep us
    loadkeys us

See IP setup:

    ip -c a

Connect to wifi:

    iwctl
    station wlan0 scan
    station wlan0 get-networks
    station wlan0 connect myssid

Test the internet connection:

    ping www.google.com

IMPORTANT, set up system time:

    timedatectl list-timezones | grep New_York
    ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
    timedatectl set-ntp true
    date
    hwclock --systohc

Check mirror list; hopefully this has been updated automatically by Reflector:

    vim /etc/pacman.d/mirrorlist

Sync package database:

    pacman -Sy

# Partition disk

See disks:

    lsblk

Partition disk:

    gdisk /dev/vda

Make boot EFI partition:

    n
    1
    ENTER for default first sector
    +1GB
    ef00

Make main linux partition:

    n
    2
    ENTER for default first sector
    ENTER for default type linux

Print parition table:

    p

Write the partiion table and exit:

    w
    Y

# Format paritions

Format the boot parition:

    mkfs.vfat /dev/vda1

Set up BTRFS on and encrypted LUKS partition:

     cryptsetup luksFormat /dev/vda2
     YES
     passphrase
     cryptsetup luksOpen /dev/vda2 vg0
     vgcreate vg0 /dev/mapper/vg0
     lvcreate -L 40G -n swap vg0 # 32GB ram, with buffer
     lvcreate -l 100%FREE -n root vg0
     mkswap /dev/mapper/vg0-swap
     swapon /dev/mapper/vg0-swap
     mkfs.btrfs /dev/mapper/vg0-root
 
 Set up BTRFS sub-volumes:
 
     mount /dev/mapper/vg0-root /mnt
     btrfs subvolume create /mnt/@
     btrfs subvolume create /mnt/@home
     btrfs subvolume create /mnt/@var
     btrfs subvolume create /mnt/@snapshots
     umount /mnt
 
 Mount the filesystems:

    mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@ /dev/mapper/vg0-root /mnt
    mkdir -p /mnt/{boot,home,var,swap,.snapshots}
    mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@home /dev/mapper/vg0-root /mnt/home
    mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@var /dev/mapper/vg0-root /mnt/var
    mount -o noatime,compress=zstd,ssd,discard=async,space_cache=v2,subvol=@snapshots /dev/mapper/vg0-root /mnt/.snapshots
    mount /dev/vda1 /mnt/boot
    lsblk

# Install Arch

Pacstrap:

    pacstrap /mnt base linux-zen linux-zen-headers linux-firmware git vim intel-ucode btrfs-progs

More setup:

    genfstab -U /mnt >> /mnt/etc/fstab
    vim /mnt/etc/fstab
    :q

Chroot time:

    arch-chroot /mnt

Localisation:

    ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
    vim /etc/locale.gen

Un-comment desired locales, e.g. en_US.UTF-8 and en_US

    :wq
    locale-gen
    echo "LOCALE=en_US.UTF-8" > /etc/locale.conf
    echo "KEYMAP=us" > /etc/vconsole.conf

Other setup:

    echo "myhost" > /etc/hostname
    vim /etc/hosts

Make /etc/hosts look like:

    127.0.0.1   localhost
    ::1   localhost
    127.0.1.1   myhost.localdomain  myhost

Set root password:

    passwd

# Set up boot

    pacman -Syu grub efibootmgr networkmanager wpa_supplicant mtools dosfstools base-devel linux-headers lvm2

Choose defaults when prompted:

    grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

Get UUID of boot device and append it to end of file for cutting and pasting:

    blkid | grep vda2 | cut -d\" -f 2 >> /etc/default/grub # we need to edit this next
    vim /etc/default/grub

Edit GRUB_CMDLINE_LINUX_DEFAULT to look like this; you can cut the UUID from the end of the file by pressing v to go
into visual mode, moving cursor to the end, and then pressing d to cut. Later you can paste it with the p command.

    "loglevel=3 quiet root=/dev/mapper/vg0-root rw rd.lvm.vg=vg0 rd.luks.uuid=/dev/vda2UUID resume=/dev/mapper/vg0-swap"

Set up Grub:

    grub-mkconfig -o /boot/grub/grub.cfg

Set up boot image:

    vim /etc/mkinitcpio.conf

Edit the MODULES line to look like this:

    MODULES=(btrfs)

Edit the HOOKS line to look like this:

    HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems resume fsck)

Then run:

    mkinitcpio -P

Enable services:

    systemctl enable NetworkManager
    systemctl enable fstrim.timer # ssd cleanup

Set up user:

    useradd -m myuser
    passwd myuser
    password
    usermod -aG wheel myuser
    id
    EDITOR=vim visudo

Un-comment the line underneath "uncomment to allow members of group wheel to execute any command".

# Boot for the first time

    exit
    umount -R /mnt
    reboot

Remove the USB stick. Type the LUKS password when prompted. Log in with your user created above.

# Configure network after first boot

Use this tool to configure the network:

    sudo nmtui

# Fix watchdog error message on reboot:

    sudo vim /etc/systemd/system.conf

Un-comment and replace RebootWatchdogSec line with:

    RebootWatchdogSec=0

# Set up the AUR

    sudo pacman -S --needed git base-devel
    git clone https://aur.archlinux.org/yay.git
    cd yay
    makepkg -si

# Install system packages:

    sudo pacman -S man-db man-pages nvidia-open-dkms nvidia-utils nvidia-prime openssh sshfs foot snapper duf lf fzf cups rsync plocate
    yay -S system76-dkms-git system76-acpi-dkms system76-io-dkms system76-driver system76-power system76-firmware

# Start system76 services

    sudo systemctl enable system76
    sudo systemctl enable system76-firmware-daemon
    sudo systemctl enable com.system76.PowerDaemon
    reboot

# Start printer service

    sudo systemctl enable cups.socket

# Install KDE desktop environment and necessary packages:

    sudo pacman -S plasma-meta plasma-browser-integration kde-gtk-config xdg-desktop-portal xdg-desktop-portal-kde sddm sddm-kcm wl-clipboard foot spectacle okular ark unrar dolphin unzip mpv kdegraphics-thumbnailers libappimage ffmpegthumbs noto-fonts-cjk
    sudo yay -S zen-browser-bin phonon-qt6-mpv

Choose pipewire-jack, wireplumber, noto-fonts, vlc

    sudo systemctl enable sddm
    sudo systemctl start sddm
    sudo systemctl --user enable foot-server

Fingerprint reader:

    sudo pacman -Sy fprintd
    fprintd-enroll

# Disable indexing
    balooctl6 disable
    systemctl --user mask plasma-baloorunner.service

# Setup automatic login

    vim /etc/sddm.conf.d/kde_settings.conf
    add
    [Autologin]
    User=john
    Session=plasma

# install snapper packages

    sudo pacman -S grub-btrfs snap-pac

# Set up snapper

    sudo -s
    umount /.snapshots
    rm -r /.snapshots
    snapper -c root create-config /
    btrfs subvolume list /
    btrfs subvolume delete /.snapshots
    mkdir /.snapshots
    more /etc/fstab
    mount -a
    btrfs subvol get-default /
    btrfs subvol list/
    btrfs subvolume set-default ${@TOPLEVEL_ID} /
    btrfs subvol get-default /
    vim /etc/snapper/configs/root
    change ALLOW_GROUPS="wheel"
    change TIMELINE_LIMITs according to arch wiki
    :wq
    chown -R :wheel /.snapshots/
    systemctl enable --now grub-btrfs.path
    grub-mkconfig -o /boot/grub/grub.cfg
    systemctl enable --now snapper-timeline.timer
    systemctl enable --now snapper-cleanup.timer
    snapper -c root create -d "***Base System Configuration***"
    grub-mkconfig -o /boot/grub/grub.cfg

# set up virtual machine


# additional packages

    sudo pacman -S freecad paraview inkscape miniconda3 transmission-qt code electrum wireshark-qt jupyter-notebook pass pass-otp nerd-fonts zsh zsh-completions zsh-doc python-scipy python-seaborn neovim-qt obsidian tmux pigz dictd  element-desktop virt-manager qemu-base qemu-desktop qview texlive
    sudo pacman -S imagemagick
    yay -S zoom zotero-bin onlyoffice-bin pybind11
    yay -S klatexformula

# (troubleshooting) add kernel parameter if you see a ghost monitor
    "initcall_blacklist=simpledrm_platform_driver_init"
