Instalace OPNsense 25.7 ve VM300 (Moravec-OPNsense).
- dávat pozor na login "installer" jako první
- prvotní IP nastavena na 192.168.99.1 (staticky, později až budou VLANY přenastavíme)

K tomu Ubuntu ve VM301 (Moravec-Ubuntu).
- slouží pro GUI ovládání OPNsense

302VM Wazuh
303VM DMZ

OPNsense nastavení:
em0 = LAN
em1 = WAN

Nefunguje mi internet
- Query Forwarding (use system nameservers) - nejde
- vmbr0 (vlan30) dostávám IP (192.168.30.108)
- vmbr0 (bez vlan) dostávám IP (10.200.201.142/23)
- 

Chyby:
VM nestartuje se síťovkou s VLAN tagem.
```
no physical interface on bridge 'vmbr2'
kvm: -netdev type=tap,id=net0,ifname=tap303i0,script=/usr/libexec/qemu-server/pve-bridge,downscript=/usr/libexec/qemu-server/pve-bridgedown,vhost=on: network script /usr/libexec/qemu-server/pve-bridge failed with status 6400
TASK ERROR: start failed: QEMU exited with code 1
```

Kompletní reinstall OPNsense. Přešel jsem na vmbr4 - 2 mi nefungovala (?).
Internet stále nefunguje.
vmbr4 s VLAN30 funguje na klasickém VM, ale v OPNsense NE.

Instituce často dávají na “WAN” RFC1918 adresu (10.x / 172.16–31 / 192.168.x). OPNsense má na WAN defaultně volby:
Block private networks
Block bogon networks
- nepomohlo
Unbound DNS → Query Forwarding (nutné!)
