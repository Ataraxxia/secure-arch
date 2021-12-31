# Introduction

This section of guide will tell you how to install fully encrypted base system with SecureBoot enabled. This specific guide uses Unified Boot Image for booting and therefore there is no need for software like GRUB.

** Check this list before starting! **

- Your computer supports SecureBoot/UEFI
- Your computer allows for enrollment of your own secureboot keys
- Your computer does not have manufacturer's backdoors
- You DO NOT need dualboot with windows
- You do not requrie dedicated SWAP partition

If you are not interested in SecureBoot, you can just skip last section of this document.

## Preparing USB and booting the installer

Download the latest Archlinux ISO and copy it to your USB:

	sudo dd if=/path/to/file.iso of=/dev/sdX status=progress
	sync

Reboot your machine and if enabled, disable secureboot in BIOS. After that, boot ArchLinux USB.

When your installer has booted, especially on laptop, you may want to enable WiFi connection:

	iwctl
	station wlan0 connect SSID
	<password prompt>
	exit

## Disk partitioning

Following example assumes you have a nvme drive. Your drive may as well report as /dev/sdX.

You can use your favoruite tool, like fstab or have minimal GUI with cgdisk. Propsed partioning schema presents as follows:

	+----------------------+----------------------+----------------------+----------------------+
	| EFI system partition |         Logical volume 1                                           |
	|                      |                                                                    |
	| /efi                 |         /                                                          |
	|                      |                                                                    |
	|                      |         /dev/vg/root                                               |
	| /dev/nvme0n1p1       |----------------------+----------------------+----------------------+
	| unencrypted          | /dev/nvme0n1p2 encrypted using LVM on LUKS2                        |
	+----------------------+--------------------------------------------------------------------+

My partition sizes and used partition codes look like this:

	/dev/nvme0n1p1 - EFI - 512MB;				partition code EF00
	/dev/nvme0n1p2 - encrypted LUKS - remaining space;	partition code 8309

The lack of SWAP partition is intentional; if you need it, you can configure SWAP as file in your filesystem later.

Now we can create encrypted volume and open it (--perf options are optional and recommended for SSD):
	cryptsetup luksFormat --type luks2 /dev/nvme0n1p2
	cryptsetup open --perf-no_read_workqueue --perf-no_write_workqueue --persistent /dev/nvme0n1p2 cryptlvm


We also need to format EFI partition:
	mkfs.fat -F32 /dev/nvme0n1p1

Configuring LVM and formatting root partition:
	pvcreate /dev/mapper/cryptlvm
	vgcreate vg /dev/mapper/cryptlvm
	lvcreate -l 100%FREE vg -n root

	mkfs.ext4 /dev/vg/root

After all is done we need to mount our drives:
	mount /dev/vg/root /mnt
	mkdir /mnt/efi
	mount /dev/nvme0n1p1 /mnt/efi

## System bootstraping

__In the next step it is recommended to install CPU microcode package. Depending on whether you have intel of amd you should apend intel-ucode or amd-ucode to your pacstrap__

My pacstrap presents as follows:
	pacstrap /mnt base linux linux-firmware sudo vim lvm2 dracut sbsigntools iwd git efibootmgr binutils YOUR_UCODE_PACKAGE

Generate fstab:
	genfstab -U /mnt >> /mnt/etc/fstab

Now you can chroot to your system and perform some basic configuration:
	arch-chroot /mnt

Set the root password:
	passwd

My suggestion is to also install man for additional help you may require:
	pacman -Syu man-db

Set timezone and generate /etc/adjtime:
	ln -sf /usr/share/zoneinfo/<Region>/<city> /etc/localtime
	hwclock --systohc

Set your desired locale:
	vim /etc/locale.gen # uncomment locales you want
	locale-gen

	vim /etc/locale.conf
		LANG=en_GB.UTF-8

Konfigure your keyboard layout (mine is adapted to polish-programmer keyboard):
	vim /etc/vconsole.conf
		KEYMAP=pl
		FONT=Lat2-Terminus16
		FONT_MAP=8859-2

Set your hostname:
  vim /etc/hostname

Create your user (in my case the home dir has not created automatically):
	useradd YOUR_NAME
	mkdir /home/YOUR_NAME
	chown YOUR_NAME:YOUR_NAME /home/YOUR_NAME
	passwd YOUR_NAME

Add your user to sudo:
	visudo
		%wheel	ALL=(ALL) ALL # Uncomment this line

	usermod -aG wheel YOUR_NAME

## Configuring bootloader

Install systemd-boot to EFI partition
	bootctl install # it will find /efi on its own

Install dracut-uefi-hook (dracut-hook-uefi is also a viable option), currently available only from AUR:
	su - YOUR_NAME 
	sudo pacman -S fakeroot
	mkdir repos && cd repos
	git clone https://aur.archlinux.org/dracut-uefi-hook.git
	cd dracut*

Build and install AUR package:
	makepkg -si

After it is done, return to root:
	exit

Add pacman hook that will update bootctl on systemd-boot package upgrade:
	mkdir /etc/pacman.d/hooks
	vim /etc/pacman.d/hooks/998-systemd-boot.hook
		[Trigger]
		Type = Package
		Operation = Install
		Operation = Upgrade
		Target = systemd
	
		[Action]
		Description = Updating systemd-boot
		When = PostTransaction
		Exec = /usr/bin/bootctl update;

Check UUID of your encrypted volume and write it to file you will edit next:
	blkid -s UUID -o value /dev/nvme0n1p2 >> /etc/dracut.conf.d/cmdline.conf

Edit the file and fill with with kernel arguments:
	vim /etc/dracut.conf.d/cmdline.conf
		kernel_cmdline="rd.luks.uuid=luks-YOUR_UUID rd.lvm.lv=vg/root root=/dev/mapper/vg-root rootfstype=ext4 rootflags=rw,relatime"

Create file with flags:
	vim /etc/dracut.conf.d/flags.conf
		compress="zstd"
		hostonly="no"

Generate your image, first check your kernel version and use that as an argment in dracut --kver. The reason for that is often times kernel installed from pacstrap is a bit newer from what you have on your USB in chrooted enviroment:
	ls /lib/modules # You should see single directory with your kernel version
	dracut --uefi --kver FULL_NAME_OF_DIRECTORY_FROM_PREVIOUS_COMMAND

Depending on your setup, before reboot you may want to install at least dhclient to your system:
	pacman -S dhclient

Now you can reboot and log into your system.

## SecureBoot

At this point you should enable Setup Mode for SecureBoot in your BIOS, and erase your existing keys (it may spare you setting attributes for efi vars in OS). If your system does not offer reverting to default keys (useful if you want to install windows later), you should backup them, though this will not be described here.

Configuring SecureBoot is easy with sbctl:
	pacman -S sbctl

Check your status, setup mode should be enabled:
	sbctl status
	  Installed:      ✘ Sbctl is not installed
	  Setup Mode:     ✘ Enabled
	  Secure Boot:    ✘ Disabled

Create keys and sign binaries:
	sbctl create-keys
	sbctl sign -s /efi/EFI/BOOT/BOOTx64.EFI
	sbctl sign -s /efi/EFI/Linux/*.efi #it should be single file with name verying from kernel version
	sbctl sign -s /efi/EFI/systemd/systemd-bootx64.efi

Configure dracut to know where are signing keys:
	vim /etc/dracut.conf.d/secureboot.conf
		uefi_secureboot_cert="/usr/share/secureboot/keys/db/db.pem"
		uefi_secureboot_cert="/usr/share/secureboot/keys/db/db.key"

Enroll previously generated keys:
	sbctl enroll-keys

Reboot the system. Enable only UEFI boot in BIOS and set BIOS password so evil maid won't simply turn off the setting. If everything went fine you should first of all, boot into your system, and then verify with sbctl or bootctl:
	sbctl status
	  Installed:	✓ sbctl is installed
	  Owner GUID:	MY_GUID
	  Setup Mode:	✓ Disabled
	  Secure Boot:	✓ Enabled
