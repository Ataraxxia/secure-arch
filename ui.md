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
pacman -S sway dmenu polkit swaybg waybar xorg-xwayland
```

This basically gives you a starting base. You can start the UI manually with:

```
sway
```

and personally I like to keep it that way. This means you will log via terminal and decide whether you want to jump into the UI or not.

### Keys cheatsheet

https://depau.github.io/sway-cheatsheet/

### Terminal

### Popup command window

https://aur.archlinux.org/packages/sway-launcher-desktop

### Status bar

Modifying the swaybar

### Lockscreen

### Background

