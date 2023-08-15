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

Edit `/etc/systemd/resolved.conf` and uncomment following options. You can set whatever DNS servers you want. I'm using Quad9 for primary resolvers and Cloudflare for fallback.

```
vim /etc/systemd/resolved.conf

  DNS=9.9.9.9 2620:fe::9
  FallbackDNS=1.1.1.1 2606:4700:4700::64
  #Domains=
  DNSSEC=yes
  DNSOverTLS=yes
  #MulticastDNS=yes
  #LLMNR=yes
  #Cache=yes
  #CacheFromLocalhost=no
  #DNSStubListener=yes
  #DNSStubListenerExtra=
  #ReadEtcHosts=yes
  #ResolveUnicastSingleLabel=no
  #StaleRetentionSec=0
```


Now enable the systemd-resolved service:
```
systemctl enable --now systemd-resolved
```

There's a chance that systemd-resolved took care of `/etc/resolv.conf` but verify that it points to `127.0.0.53`:

```
vim /etc/resolv.conf
  nameserver 127.0.0.53
```

Finally, verify with `resolvectl`:

```
$ resolvectl status

Global
           Protocols: +LLMNR +mDNS +DNSOverTLS DNSSEC=yes/supported
    resolv.conf mode: stub
  Current DNS Server: 2620:fe::9
         DNS Servers: 9.9.9.9 2620:fe::9
Fallback DNS Servers: 1.1.1.1 2606:4700:4700::64

Link 2 (wlan0)
    Current Scopes: LLMNR/IPv4 LLMNR/IPv6 mDNS/IPv4 mDNS/IPv6
         Protocols: -DefaultRoute +LLMNR +mDNS +DNSOverTLS DNSSEC=yes/supported
```

Additional verification can be done with `dig` and `tcpdump`.
