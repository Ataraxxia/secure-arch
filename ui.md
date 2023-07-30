# Introduction

In this part I will present you two options for user interface.

## GNOME

Just install gnome. This will get you the usuall, complete desktop expirience.

```
pacman -S gnome
```

Then go with all the default options, and enable systemD unit so it autostarts:

```
systemctl enable gdm.service
```

Reboot.


## Sway

Sway is a tiling window manager. It's like i3 but instead of Xorg it uses Wayland for display server protocol. From my expirience at this moment everything I need works, 
like screensharing on various platforms ( web google meet, teams, zoom ), and even Steam and GoG games.

For sway we needt to install few extra packets:

```
pacman -S sway dmenu polkit swaybg waybar xorg-xwayland xdg-utils fakeroot
```

This basically gives you a starting base. You can start the UI manually with:

```
sway
```

and personally I like to keep it that way. This means you will log via terminal and decide whether you want to jump into the UI or not.

Copy `config` file to your user's directory so you can enable your own changes:

```
mkdir .config/sway
cp /etc/sway/config .config/sway/
```

### Keys cheatsheet

https://depau.github.io/sway-cheatsheet/

### Terminal

Install your preffered terminal emulator (foot, alacritty, whatever you want), I'm going to use alacritty for this example:

```
pacman -S alacritty
```

Then edit your config file and change terminal executable

```
vim ~/.config/sway/config

# Your preferred terminal emulator
set $term alacritty
```

Reload your sway config `Shift + mod + c`, now you can open terminal by pressing `mod + Enter`

Personally I like my terminal windows a bit transparent, so let's do that:

```
mkdir ~/.config/alacritty/
vim ~/.config/alacritty/alacritty.yml

window:
  opacity: 0.75

```

Changes should be visible immediately.

### Popup command window

Create direcotory for your AUR repos and clone sway launcher there:

```
mkdir -p ~/repos/AUR/
cd ~/repos/AUR
git clone https://aur.archlinux.org/sway-launcher-desktop.git
```

Install `fzf` dependency and font-awesome for icons, then build and install sway launcher:

```
cd sway-launcher-desktop
pacman -S fzf ttf-font-awesome
makepkg --install
```

Configure sway to start using that as our launcher:

```
vim ~/.config/sway/config

set $menu dmenu_path | dmenu | xargs swaymsg exec -- # Comment out this line

# Put those lines below the commented out one
for_window [app_id="^launcher$"] floating enable, sticky enable, resize set 30 ppt 60 ppt, border pixel 10
set $menu exec $term --class=launcher -e /usr/bin/sway-launcher-desktop

```


### Status bar

Modifying the swaybar

### Lockscreen

### Background

