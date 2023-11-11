# Intro

I'm gonna be honest, you probably probably won't find anything groundbreaking here and I'm not gonna act like I've discovered a mother-lode,
but with this you have the neccessary basics to consider your laptop "secure".

I hope you've managed to end up with reasonably secure laptop that is still convenient to use.

**THINGS NOT COVERED HERE:**

* Sandboxing
* VPN's
* Secure backup with public cloud

## YouTube version of this tutorial

I recorded a YT series where I follow all of these instructions. The installation there is performed on a real-life hardware. Check it out if you get stuck.

[Secure ArchLinux Installation part 3 - Basic Hardening](https://www.youtube.com/watch?v=ivXTv5ate-M)

## Firewall

I'm goning to use nftables. Most distros started switching to it and it streamlines persistence compared to iptables.

Install nftables:

```
pacman -S nftables
```

Edit the `/etc/nftables.conf`. Proposed firewall rules:

* drop all forwarding traffic (we're not a router),
* allow loopback (127.0.0.0)
* allow ICMP for v4 and v6 (you can turn it off, but for v6 it will dissable [SLAAC](https://wiki.archlinux.org/title/IPv6#Stateless_autoconfiguration_(SLAAC))),
* allow returning packets for established connections,
* block all else.

```
#!/usr/bin/nft -f
 
 table inet filter
 delete table inet filter
 table inet filter {
   chain input {
     type filter hook input priority filter
     policy drop
 
     ct state invalid drop comment "early drop of invalid connections"
     ct state {established, related} accept comment "allow tracked connections"
     iifname lo accept comment "allow from loopback"
     ip protocol icmp accept comment "allow icmp"
     meta l4proto ipv6-icmp accept comment "allow icmp v6"
     pkttype host limit rate 5/second counter reject with icmpx type admin-prohibited
     counter
   }

   chain forward {
     type filter hook forward priority filter
     policy drop
   }
 }
```

Enable the nftables service and list loaded rules for confirmation:

```
systemctl enable --now nftables

nft list ruleset
```

## Kernel parameters

Since firewall allows ICMP traffic, it may be a good idea to disable some network options. Edit your `/etc/sysctl.d/90-network.conf`:

```
# Do not act as a router
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# SYN flood protection
net.ipv4.tcp_syncookies = 1

# Disable ICMP redirect
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# Do not send ICMP redirects
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
```

Load your new rules with:

```
sysctl --system
```

## Password manager

My preffered solution is KeePass with their .kdbx format, that can be opened by multitude of programs and solutions.

For arch you can install local client, KeePassXC:

```
pacman -S keepassxc
```

## Closing remarks

My number one security advice is:

**Always lock your screen when leaving your laptop or PC.**

Make it your new habit. Getting up from the chair == locking screen.
