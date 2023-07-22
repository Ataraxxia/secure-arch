 # Introduction

This section of guide will tell you how to install fully encrypted base system with SecureBoot enabled. This specific guide uses Unified Boot Image for booting and therefore there is no need for software like GRUB.

__Check this list before starting!__

- Your computer supports SecureBoot/UEFI
- Your computer allows for enrollment of your own secureboot keys
- Your computer does not have manufacturer's backdoors
- ~~You DO NOT need dualboot with windows~~ You can choose whether you want Microsoft CA when enrolling keys

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

You can use your favoruite tool, that supports creating the GPT partiton, for example `gdisk`:

	+----------------------+----------------------+----------------------+----------------------+
	| EFI system partition |         LVM                                                        |
	|                      |                                                                    |
	| /efi                 |         /                                                          |
	|                      |                                                                    |
	| /dev/nvme0n1p1       |         /dev/vg/root                                               |
	|                      |----------------------+----------------------+----------------------+
	| unencrypted          | /dev/nvme0n1p2 encrypted using LUKS2                               |
	+----------------------+--------------------------------------------------------------------+

My partition sizes and used partition codes look like this:

	/dev/nvme0n1p1 - EFI - 512MB;				partition code EF00
	/dev/nvme0n1p2 - encrypted LUKS - remaining space;	partition code 8309

The lack of SWAP partition is intentional; if you need it, you can configure SWAP as file in your filesystem later.

We also need to format EFI partition:

	mkfs.fat -F32 /dev/nvme0n1p1

Now we can create encrypted volume and open it (--perf options are optional and recommended for SSD):

	cryptsetup luksFormat --type luks2 /dev/nvme0n1p2
	cryptsetup open --perf-no_read_workqueue --perf-no_write_workqueue --persistent /dev/nvme0n1p2 cryptlvm


Configuring LVM and formatting root partition:

	pvcreate /dev/mapper/cryptlvm
	vgcreate vg /dev/mapper/cryptlvm
	lvcreate -l 100%FREE vg -n root

	mkfs.ext4 /dev/vg/root

After all is done we need to mount our drives:

	mount /dev/vg/root /mnt
	mkdir -p /mnt/boot/efi
	mount /dev/nvme0n1p1 /mnt/boot/efi

## System bootstraping

_In the next step it is recommended to install CPU microcode package. Depending on whether you have intel of amd you should apend intel-ucode or amd-ucode to your pacstrap_

My pacstrap presents as follows:

	pacstrap /mnt base linux linux-firmware YOUR_UCODE_PACKAGE sudo vim lvm2 dracut sbsigntools iwd git efibootmgr binutils dhcpcd

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

Configure your keyboard layout (mine is adapted to polish-programmer keyboard):

	vim /etc/vconsole.conf
		KEYMAP=pl
		FONT=Lat2-Terminus16
		FONT_MAP=8859-2

Set your hostname:

	vim /etc/hostname

Create your user:

	useradd -m YOUR_NAME
	passwd YOUR_NAME

Add your user to sudo:

	visudo
		%wheel	ALL=(ALL) ALL # Uncomment this line

	usermod -aG wheel YOUR_NAME

 Enable some basic systemd units:

 	systemctl enable dhcpcd
  	systemctl enable iwd

## Creating Unified Kernel Image and configuring boot entry

Create dracut scripts that will hook into pacman:

	vim /usr/local/bin/dracut-install.sh

		#!/usr/bin/env bash

		mkdir -p /boot/efi/EFI/Linux
  
		while read -r line; do
			if [[ "$line" == 'usr/lib/modules/'+([^/])'/pkgbase' ]]; then
				kver="${line#'usr/lib/modules/'}"
				kver="${kver%'/pkgbase'}"
		
				dracut --force --uefi --kver "$kver" /boot/efi/EFI/Linux/arch-linux.efi
			fi
		done

And the removal script:

	vim /usr/local/bin/dracut-remove.sh

		#!/usr/bin/env bash
	 	rm -f /boot/efi/EFI/Linux/arch-linux.efi

Make those scripts executable and create pacman's hook directory:

	chmod +x /usr/local/bin/dracut-*
 	mkdir /etc/pacman.d/hooks

 Now the actual hooks, first for the install and upgrade:

	 vim /etc/pacman.d/hooks/90-dracut-install.hook
  
		[Trigger]
		Type = Path
		Operation = Install
		Operation = Upgrade
		Target = usr/lib/modules/*/pkgbase
		
		[Action]
		Description = Updating linux EFI image
		When = PostTransaction
		Exec = /usr/local/bin/dracut-install.sh
		Depends = dracut
		NeedsTargets

And for removal:

	vim /etc/pacman.d/hooks/60-dracut-remove.hook
 
		[Trigger]
		Type = Path
		Operation = Remove
		Target = usr/lib/modules/*/pkgbase
		
		[Action]
		Description = Removing linux EFI image
		When = PreTransaction
		Exec = /usr/local/bin/dracut-remove.sh
		NeedsTargets

Check UUID of your encrypted volume and write it to file you will edit next:

	blkid -s UUID -o value /dev/nvme0n1p2 >> /etc/dracut.conf.d/cmdline.conf

Edit the file and fill with with kernel arguments:

	vim /etc/dracut.conf.d/cmdline.conf
		kernel_cmdline="rd.luks.uuid=luks-YOUR_UUID rd.lvm.lv=vg/root root=/dev/mapper/vg-root rootfstype=ext4 rootflags=rw,relatime"

Create file with flags:

	vim /etc/dracut.conf.d/flags.conf
		compress="zstd"
		hostonly="no"

Generate your image by re-installing `linux` package and making sure the hooks work properly:

	pacman -S linux

 You should have `arch-linux.efi` within your `/efi/EFI/Linux/`

 Now you only have to add UEFI boot entry and create an order of booting:

	efibootmgr --create --disk /dev/nvme0n1 --part 1 --label "Arch Linux" --loader 'EFI\Linux\arch-linux.efi' --unicode
	efibootmgr 		# Check if you have left over UEFI entries, remove them with efibootmgr -b INDEX -B and note down Arch index
	efibootmgr -o ARCH_INDEX_FROM_PREVIOUS_COMMAND # 0 or whatever number your Arch entry shows as

Now you can reboot and log into your system.

:exclamation: :exclamation: :exclamation: **Compatilibity thing I noticed** :exclamation: :exclamation: :exclamation:

Some (older?) platforms can ignore entries by efibootmgr all together and just look for `EFI\BOOT\bootx64.efi`, in that case you may generate your UKI directly to that directory and under that name. It's very important that the name is also `bootx64.efi`.

## SecureBoot

At this point you should enable Setup Mode for SecureBoot in your BIOS, and erase your existing keys (it may spare you setting attributes for efi vars in OS). If your system does not offer reverting to default keys (useful if you want to install windows later), you should backup them, though this will not be described here.

Configuring SecureBoot is easy with sbctl:

	pacman -S sbctl

Check your status, setup mode should be enabled (You can do that in BIOS):

	sbctl status
	  Installed:      ✘ Sbctl is not installed
	  Setup Mode:     ✘ Enabled
	  Secure Boot:    ✘ Disabled

Create keys and sign binaries:

	sbctl create-keys
	sbctl sign -s /boot/efi/EFI/Linux/arch-linux.efi #it should be single file with name verying from kernel version

Configure dracut to know where are signing keys:

	vim /etc/dracut.conf.d/secureboot.conf
		uefi_secureboot_cert="/usr/share/secureboot/keys/db/db.pem"
		uefi_secureboot_key="/usr/share/secureboot/keys/db/db.key"

We also need to fix sbctl's pacman hook. Creating the following file will overshadow the real one:

	vim /etc/pacman.d/hooks/zz-sbctl.hook
		[Trigger]
		Type = Path
		Operation = Install
		Operation = Upgrade
		Operation = Remove
		Target = boot/*
		Target = efi/*
		Target = usr/lib/modules/*/vmlinuz
		Target = usr/lib/initcpio/*
		Target = usr/lib/**/efi/*.efi*

		[Action]
		Description = Signing EFI binaries...
		When = PostTransaction
		Exec = /usr/bin/sbctl sign /boot/efi/EFI/Linux/arch-linux.efi

Enroll previously generated keys (drop microsoft option if you don't want their keys):

	sbctl enroll-keys --microsoft

Reboot the system. Enable only UEFI boot in BIOS and set BIOS password so evil maid won't simply turn off the setting. If everything went fine you should first of all, boot into your system, and then verify with sbctl or bootctl:

	sbctl status
	  Installed:	✓ sbctl is installed
	  Owner GUID:	YOUR_GUID
	  Setup Mode:	✓ Disabled
	  Secure Boot:	✓ Enabled
