# Intro
Here I don't want to focus on the security aspect of every single thing introduced. You yourself decide if you wanna go with it or not.

## SystemD Resolved
I've switched to systemd-resolved due to... VPN's. With traditional `/etc/resolv.conf` approach you can pretty much define one DNS server, others are only fallbacks in case the first one is unreachable. With systemd-resolved you can define DNS server per domain or per network interface! When I connect to work VPN, corporate.org queries are resolved with DNS supplied on the VPN network, while the rest goes through my usually defined server.

First, if you're using DHCPd, disable it overwriting `/etc/resolv.conf`

```
vim /etc/dhcpcd.conf

  # Add this line at the end of the file
  nohook resolv.conf
```

Now enable the systemd-resolved service:

```
systemctl enable --now systemd-resolved
```
