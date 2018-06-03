![cyberpunk](https://image.ibb.co/d72MDJ/Screenshot_2018_06_03_20_45_18.png)


Particionamento EFI:
```
# parted -a optimal /dev/sda
 (parted) print
 (parted) mklabel gpt

 (parted) unit mib

 (parted) mkpart primary 3MiB 515MiB
 (parted) name 1 "EFI System Partition"
 (parted) set 1 boot on

 (parted) mkpart primary 515MiB 100%
 (parted) name 2 luks_partition
 (parted) set 2 lvm on
```

```
 # cryptsetup luksFormat -c aes-xts-plain64 -s 512 /dev/sda2 --debug
 # cryptsetup luksOpen /dev/sda2 crypt_disc --debug
```

Configuração LVM:

```
 # pvcreate /dev/mapper/crypt_disc
 # vgcreate vg_01 /dev/mapper/crypt_disc
 # lvcreate -L 10G -n rootfs vg_01
 # lvcreate -L 20G -n var vg_01
 # lvcreate -l 100%VG -n home vg_01
```

[OPCIONAL] caso o vg não seja reconhecido, terá que ativá-lo:

```
# vgscan --mknodes -v
# vgchange -a y vg_01
```

Formatação:

```
 # mkfs.vfat -F 32 /dev/sda1
 # mkfs.ext4 /dev/vg_01/rootfs
 # mkfs.ext4 /dev/vg_01/var
 # mkfs.btrfs /dev/vg_01/home
```

Montando dispositivos:

```
 # mount /dev/vg_01/rootfs /mnt/gentoo
 # mkdir /mnt/gentoo/{var,home,boot}
 # mount /dev/vg_01/var /mnt/gentoo/var
 # mount /dev/vg_01/home /mnt/gentoo/home
 # mount /dev/sda1 /mnt/gentoo/boot
```

Entrando no diretório e baixando o Stage3:

```
 # cd /mnt/gentoo/
 # wget http://gentoo.c3sl.ufpr.br/releases/amd64/autobuilds/current-stage3-amd64/stage3-amd64-20180322T214502Z.tar.xz
 # tar xvpf stage3-* --xattrs-include='*.*' --numeric-owner
```

Ajustes iniciais nas configurações do Portage:

```
 # nano -w /mnt/gentoo/etc/portage/make.conf

----------- insira isto ----------------
CHOST="x86_64-pc-linux-gnu"
CFLAGS="-march=sandybridge -O2 -pipe"
CXXFLAGS="${CFLAGS}"
MAKEOPTS="-j4"

USE="cryptsetup"
----------- insira isto ----------------
```

Informações sobre flags: https://wiki.gentoo.org/wiki/Safe_CFLAGS

```
 # mirrorselect -i -o >> /mnt/gentoo/etc/portage/make.conf
 # mkdir --parents /mnt/gentoo/etc/portage/repos.conf
 # cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
 # cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
```

Montando pseudo-filesystems:
```
 # mount --types proc /proc /mnt/gentoo/proc; mount --rbind /sys /mnt/gentoo/sys; mount --make-rslave /mnt/gentoo/sys; mount --rbind /dev /mnt/gentoo/dev; mount --make-rslave /mnt/gentoo/dev
```

Fazendo chroot e alterando o ambiente:

```
 # chroot /mnt/gentoo /bin/bash
 # source /etc/profile; export PS1="(chroot) ${PS1}"
```

Sincronizando repositório, setando profile, definindo timezone e idioma:

```
 # emerge-webrsync
 # eselect profile set default/linux/amd64/17.0/desktop/gnome/systemd
 # echo "Brazil/East" > /etc/timezone
 # emerge --config sys-libs/timezone-data
 # eselect locale list # opcional
 # eselect locale set 593
 # env-update && source /etc/profile && export PS1="(chroot) $PS1"
```

Baixando source do kernel e compilando:

```
 # emerge --ask sys-kernel/gentoo-sources
 # cd /usr/src/linux
 # make mrproper
 # make menuconfig
 # make
 # make modules_install
 # make install
```

Gerando initramfs:

```
 # emerge --ask sys-kernel/genkernel-next
 # genkernel --luks --udev --lvm initramfs
```

/etc/fstab:
```
----------- insira isto ----------------
UUID=6C0D-4530          /boot	        vfat            noauto,noatime	0 2
UUID=14aa64ea-a2fb-4b77-be9d-1d61faeeac1a	/	ext4	noatime	0 1
UUID=36a8cd7c-3fb7-4766-b3d8-1664031d2195	/var	ext4	noatime 0 2
UUID=b6584762-0904-45d1-92ab-b2848b538cea	/home	btrfs	noatime 0 2
----------- insira isto ----------------
```

```
 # systemctl enable lvm2-monitor
```

Aplicações:
```
 # emerge --ask net-wireless/iw net-wireless/wpa_supplicant sys-fs/e2fsprogs sys-fs/btrfs-progs sys-process/cronie app-admin/syslog-ng net-misc/dhcpcd vim sys-kernel/linux-firmware
```

Instalando e configurando GRUB2 EFI:

```
 # echo 'sys-boot/grub:2 device-mapper' >> /etc/portage/package.use/package.use
 # echo 'GRUB_PLATFORMS="efi-64 efi-32"' >> /etc/portage/make.conf
 # emerge --ask sys-boot/grub:2
 # blkid
 # echo "GRUB_CMDLINE_LINUX=\"dolvm crypt_root=UUID=09db4417-7b56-4081-9327-d71503bc4a8d root=UUID=8853e707-1f55-48af-92e9-877d0872f74d\"" >> /etc/default/grub

 # grub-install --target=x86_64-efi --efi-directory=/boot --removable
 # grub-mkconfig -o /boot/grub/grub.cfg
```

Definir senha de root:

```
 # passwd
```

Saindo do chroot e desmontando

```
 # exit
 # umount -l /mnt/gentoo/dev{/shm,/pts,}
 # cd; umount -R /mnt/gentoo
 # reboot
```

```
nano -w /mnt/gentoo/etc/portage/make.conf

---------------------------------------------------------
## (For mouse, keyboard, and Synaptics touchpad support)
INPUT_DEVICES="libinput synaptics"
VIDEO_CARDS="intel"
---------------------------------------------------------
```

```
 # emerge --ask --verbose x11-base/xorg-drivers
 # portageq envvar INPUT_DEVICES
 # emerge --ask x11-base/xorg-server
 # env-update
 # source /etc/profile
 # emerge --ask x11-apps/xclock
 # emerge --ask xterm
```

Acrescentar as seguintes flags:
```
---------------------------------------------------------
USE="X gtk dbus policykit alsa acl apm acpi hardened -kde -qt4 -qt5"
---------------------------------------------------------
```

Instalando o D-Bus:

```
 # emerge --ask --changed-use --deep @world
```

Definir no make.conf

```
---------------------------------------------------------
XFCE_PLUGINS="brightness clock trash"
---------------------------------------------------------
```
```
 # eselect profile set default/linux/amd64/17.0/desktop
 # emerge --ask xfce-base/xfce4-meta
 # for x in cdrom cdrw usb ; do gpasswd -a kingm0b_ $x ; done
 # env-update && source /etc/profile
 # emerge --ask x11-terms/xfce4-terminal
 # su seu_usuario -c "echo \"exec startxfce4\" > ~/.xinitrc"
 # emerge --ask x11-themes/xfwm4-themes app-editors/mousepad xfce-extra/xfce4-power-manager xfce-extra/tumbler xfce-extra/thunar-volman www-client/firefox
 # rc-update add dbus default
```

Instalar Display Manager:

```
 # emerge --ask x11-misc/lightdm
```

https://wiki.gentoo.org/wiki/Udisks
https://wiki.gentoo.org/wiki/LightDM
