<!---
 vim: set nu ai et ts=4 sw=4 ft=markdown syn=markdown :
-->

# Install Notes

### Erase Disk
---
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Erase the target disk **/dev/sda** and wipe out the previous partition table.
```bash
 sgdisk --zap-all /dev/sda
```

### Partition Disk
---
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Partition the disk so as to use encryption and lvm. Using a 1GiB **/boot** partition and **/** taking up the rest of the disk space. <br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Note:** Leaving **2MiB** of disk space is required if using the **grub** bootloader on a **gpt** partition scheme.
```bash
 parted -a optimal /dev/sda mklabel gpt mkpart primary 2MiB 1026MiB name 1 Root \
                            mkpart primary 1026MiB 100% name 2 Root set 2 lvm on
```

### Encrypt Disk
---
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Encrypt the larger partition using luks, leave the smaller as unencrypted as linux <br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;cannot load from an encrypted partition (at least not yet).
```bash
 cryptsetup --verbose --cipher aes-xts-plain64 --hash sha512 --key-size 512 --use-urandom \
            --iter-time 5000 --verify-passphrase luksFormat /dev/sda
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Now open the encrypted container and choose a mount name for it **lvm** will be the mount point.
```bash
 cryptsetup luksOpen /dev/sda lvm
```

### Formatting Disk
---
#### Use LVM to make 2 volumes on the encrypted container a swapvolume and a rootvolume.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Create a physical volume for the container **/dev/mapper/lvm**.
```bash
 pvcreate /dev/mapper/lvm 
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Create a volume group, name it, and add the physical volume to it. <br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Charon** will be used as volume group name.
```bash
 vgcreate Charon /dev/mapper/lvm
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Create swap volume
```bash
 lvcreate -L 16GB Charon -n swapVol
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Create root volume
```bash
 lvcreate -l +100%FREE Charon -n rootVol
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Activate the volumes
```bash
 vgchange -ay
```

#### Format the boot partition **/dev/sda1**, **btrfs** will be used here.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Format /dev/sda1 as **btrfs**
```bash
 mkfs.btrfs -L "BOOTFS" /dev/sda1
```

#### Format swap volume as a swap partition

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Make swap
```bash
 mkswap -L "SWAPFS" /dev/mapper/Charon-swapVol
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Activate swapping, so **genfstab** will add it to **/etc/fstab**.
```bash
 swapon /dev/mapper/Charon-swapVol
```

#### Format swap volume as a swap file
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Make swapVol as ext4
```bash
 mkfs.ext4 -L "SWAPFS" /dev/mapper/Charon-swapVol
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Mount swapVol
```bash
 mount /dev/mapper/Charon-swapVol /mnt
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Make the swapfile
```bash
 fallocate -l 16G /mnt/swapfile
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Give swapfile appropriate permissions
```bash
 chmod 600 /mnt/swapfile
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Make swap
```bash
 mkswap /mnt/swapfile
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;unmount swapVol
```bash
 umount -R /mnt
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;wait untill mounting to activate swap.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Format root volume as **btrfs** 
```bash
 mkfs.btrfs -L "ROOTFS" /dev/mapper/Charon-rootVol
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Mount the filesystems to appropriate mount points.
```bash
 mount /dev/mapper/Charon-rootVol /mnt
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Create the **ROOT** subvolume for btrfs
```bash
 cd /mnt
```
```bash
 btrfs subvolume create ROOT
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;unmount root volume and remount **ROOT** subvolume.
```bash
 umount -R /mnt
```
```bash
 mount -o ssd,noatime,compress=lzo,subvol=ROOT /dev/mapper/Charon-rootVol /mnt
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Mount the boot partition
```bash
 mkdir -p /mnt/boot
```
```bash
 mount /dev/sda1 /mnt/boot
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Create btrfs subvolumes for convienent snapshotting
```bash
 cd /mnt
```
```bash
 btrfs subvolume create etc
```
```bash
 btrfs subvolume create opt
```
```bash
 btrfs subvolume create root
```
```bash
 btrfs subvolume create home/ygoreus
```
```bash
 btrfs subvolume create var
```
```bash
 btrfs subvolume create usr
```

### Installation
---
#### Update package mirrors
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Download current US mirrorlist
```bash
 wget -q 'https://archlinux.org/mirrorlist/?country=US' -O /etc/pacman.d/mirrorlist.us
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Uncomment the servers in the mirrorlist
```bash
 sed -i 's|^#S|S|g' /etc/pacman.d/mirrorlist.us
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Rank the top 15 mirrors
```bash
 rankmirrors -n 15 /etc/pacman.d/mirrorlist.us > /etc/pacman.d/mirrorlist
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Refresh the repositories
```bash
 pacman -Syy
```

#### Install base group and other necessities
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Auto confirmation
```bash
 pacstrap /mnt base base-devel vim tmux lvm2 cryptsetup bash-completion
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Ask for confirmation
```bash
 pacstrap -i /mnt base base-devel vim tmux lvm2 cryptsetup bash-completion
```

#### Setup local environment variables
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Generate the filesystem table
```bash
 genfstab -U -p /mnt >> /mnt/etc/fstab
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Set language and some locale settings
```bash
 locale | head -n -1 > /mnt/etc/locale.conf
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Change into new root to configure further
```bash
 arch-chroot /mnt /bin/bash
```

### Configure system
---
#### Enable Pacman's extra features
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Enable color output
```bash
 sed -i 's|^#Color|Color|' /etc/pacman.conf
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Enable Verbose package listing
```bash
 sed -i 's|^#VerbosePkgLists|VerbosePkgLists|' /etc/pacman.conf
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Enable ILoveCandy feature
```bash
 sed -i -e '/# Misc options/a ILoveCandy' /etc/pacman.conf
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Enable system loging
```bash
 sed -i 's|^#UseSyslog|UseSyslog|' /etc/pacman.conf
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Enable Total Download
```bash
 sed -i 's|^#TotalDownload|TotalDownload|' /etc/pacman.conf
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Enable multilib repository
```bash
 sed -i 's|^#\[multilib\]|\[multilib\]|' /etc/pacman.conf
```
```bash
 sed -i -e '/\[multilib\]/{n;d}' /etc/pacman.conf
```
```bash
 sed -i -e '/\[multilib\]/a Include = /etc/pacman.d/mirrorlist' /etc/pacman.conf
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Refresh repositories
```bash
 pacman -Syy
```

#### Environment Variables

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Configure default character set
```bash
 sed -i 's|^#en_US|en_US|g' /etc/locale.gen
```
```bash
 locale-gen
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Set default language
```bash
 export LANG=en_US.UTF-8
```

#### Set time and date settings
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Set timezone to America/New\_York
```bash
 ln -s /usr/share/zoneinfo/America/New_York /etc/localtime
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Set hardware clock to UTC
```bash
 hwclock --systohc --utc
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install and activate network time protocal
```bash
 pacman -S --asexplicit ntp
```
```bash
 systemctl enable ntp.service
```

#### Set hostname and console settings
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Set the hostname
```bash
 echo Charon > /etc/hostname
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Add hostname to **/etc/hosts**
```bash
 sed -i 's|localhost|localhost Charon|2' /etc/hosts
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Set console keymap
```bash
 echo "KEYMAP=us" >> /etc/vconsole.conf
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Set console font
```bash
 echo "" >> /etc/vconsole.conf
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Set console font map
```bash
 echo "" >> /etc/vconsole.conf
```

#### Install and Configure bootloader
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install bootloader package and selected tools
```bash
 pacman -S --asexplicit syslinux gptfdisk mtools dosfstools 
```
```bash
 pacman -S --asexplicit perl-digest-sha1 perl-crypt-passwdmd5 
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install bootloader to disk **/dev/sda**
```bash
 syslinux-install_update -iam
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Backup bootloader config file
```bash
 cp /boot/syslinux/syslinux.cfg /boot/syslinux/syslinux.cfg.default
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Change append line to reflect encryption
```bash
 sed -i "s|APPEND.*$|APPEND cryptdevice=/dev/disk/by-uuid/${uuid}:lvm \
         root=/dev/mapper/Charon-rootVol rw elevator=deadline|" /boot/syslinux/syslinux.cfg
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Change initrd line to reflect microcode image
```bash
 sed -i "s|INITRD.*$|INITRD ../intel-ucode.img,../initramfs-linux.img|g" /boot/syslinux/syslinux.cfg
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Make changes to **/etc/mkinitcpio.conf** to reflect lvm and encryption
```bash
 sed -i 's|block filesystems|block encrypt lvm2 resume filesystems|' /etc/mkinitcpio.conf
```

### Install system components
---
#### Install and configure kernels
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install arch kernel with headers
```bash
 pacman -S --asexplicit linux linux-headers
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install LTS kernel and headers
```bash
 pacman -S --asexplicit linux-lts linux-lts-headers
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install zen kernel and headers
```bash
 pacman -S --asexplicit linux-zen linux-zen-headers
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install firmware, microcode image, and other utils
```bash
 pacman -S --asexplicit intel-ucode linux-firmware linux-api-headers util-linux
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Remake the initramfs for installed kernels
```bash
 mkinitcpio -p linux
```
```bash
 mkinitcpio -p linux-lts
```
```bash
 mkinitcpio -p linux-zen
```

#### Install filesystem support
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;For fuse support
```bash
 pacman -S --asexplicit fuse sshfs s3fs-fuse 
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;For apple device filesystems
```bash
pacman -S --asexplicit mtpfs ifuse fuseiso
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;For btrfs support
```bash
 pacman -S --asexplicit btrfs-progs
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;For f2fs support
```bash
 pacman -S --asexplicit f2fs-tools
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;For fat support
```bash
 pacman -S --asexplicit dosfstools
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;For exfat support
```bash
 pacman -S --asexplicit fuse-exfat exfat-utils
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;For ntfs support
```bash
 pacman -S --asexplicit ntfs-3g
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;For ext{2..4} support
```bash
 pacman -S --asexplicit e2fsprogs lib32-e2fsprogs
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;For xfs support
```bash
 pacman -S --asexplicit xfs-progs
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;For jfs support
```bash
 pacman -S --asexplicit jfsutils
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;For hfs support
```bash
 pacman -S --asexplicit hfsprogs
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;For nilfs support
```bash
 pacman -S --asexplicit nilfs-utils
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;For reiserfs v3 support
```bash
 pacman -S --asexplicit reiserfsprogs progsreiserfs
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;For shell filesystem support
```bash
 pacman -S --asexplicit shfs-utils
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;For squashfs support
```bash
 pacman -S --asexplicit squashfs-tools
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;For crypto file system support
```bash
 pacman -S --asexplicit encfs ecryptfs-utils
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;For ftp support
```bash
 pacman -S --asexplicit curlftpfs
```

#### Install package for a build environment
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install development 64-bit requirements
```bash
 pacman -S --asexplicit base-devel multilib-devel jshon ed
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install git vcs
```bash
 pacman -S --asexplicit git
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install bazar vcs
```bash
 pacman -S --asexplicit bzr
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install subversion vcs
```bash
 pacman -S --asexplicit subversion
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install mercurial vcs
```bash
 pacman -S --asexplicit mercurial
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install the arch bulid system (abs)
```bash
 pacman -S --asexplicit abs
```

#### Install archiving utilities
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install regular archive tools and libraries
```bash
 pacman -S --asexplicit xz cpio tar lrzip bzip2 gzip p7zip unzip zip zziplib libzip zlib libarchive
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install foreign uncompressing tools and libraries
```bash
 pacman -S --asexplicit unrar unace unarj rpmextract ecm-tools 
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install 32-bit compatability libraries
```bash
 pacman -S --asexplicit lib32-xz lib32-bzip2 lib32-zlib
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install various {un,}compression tools and libraries
```bash
 pacman -S --asexplicit lz4 lzo lzop fcrackzip haskell-zlib
```

#### Shell interpreter
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install z shell and database
```bash
 pacman -S --asexplicit zsh zshdb
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install completions and colours
```bash
 pacman -S --asexplicit zsh-completions bash-completion zsh-syntax-highlighting zsh-lovers
```

#### Editor
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install vim editor and pager
```bash
 pacman -S --asexplicit vim vim-runtime vim-spell-en vimpager
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install some amazing plugins
```bash
 pacman -S --asexplicit vim-airline vim-fugitive vim-nerdtree vim-systemd
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install spell checker
```bash
 pacman -S --asexplicit aspell aspell-en hunspell hunspell-en hyphen hyphen-en
```
```bash
 pacman -S --asexplicit libmythes mythes-en psiconv
```

#### Install cryptography tools and libraries
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install basic crypto tools
```bash
 pacman -S --asexplicit cryptsetup libgcrypt lib32-libgcrypt libgpg-error lib32-libgpg-error gpgme
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install python crypto tools
```bash
 pacman -S --asexplicit python-crypto python2-crypto python-cryptography python2-cryptography
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install other crypto tools
```bash
 pacman -S --asexplicit nettle lib32-nettle haskell-entropy libmcrypt
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install policy kit
```bash
 pacman -S --asexplicit polkit polkit-gnome haveged
```

#### Install sound subsystem
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install alsa utilities and libraries
```bash
 pacman -S --asexplicit alsa-lib lib32-alsa-lib alsa-plugins lib32-alsa-plugins alsa-utils alsa-firmware
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install pulseaudio utilities and libraries
```bash
 pacman -S --asexplicit pulseaudio pulseaudio-alsa libpulse lib32-libpulse libao 
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install canberra modules
```bash
 pacman -S --asexplicit libcanberra lib32-libcanberra libcanberra-pulse lib32-libcanberra-pulse
```

####
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
```bash
 pacman -S --asexplicit 
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
```bash
 pacman -S --asexplicit 
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
```bash
 pacman -S --asexplicit 
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
```bash
 pacman -S --asexplicit 
```


### Install userspace tools and utilities
---
#### File management and manipulation
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install ranger
```bash
 pacman -S --asexplicit ranger atool
```









### Install programming packages
---
#### Install lua programming packages
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install lua packages
```bash
 pacman -S --asexplicit lua lua51 lua52
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install lua documentation and love game engine
```bash
 pacman -S --asexplicit love love08 ldoc
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install lua filesystem libraries
```bash
 pacman -S --asexplicit lua-filesystem lua51-filesystem lua52-filesystem
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Install package manager luarocks
```
 pacman -S --asexplicit luarocks luarocks5.1 luarocks5.2
```



profont

