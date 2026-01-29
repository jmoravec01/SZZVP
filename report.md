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

***DŮLEŽITÉ***
Instituce často dávají na “WAN” RFC1918 adresu (10.x / 172.16–31 / 192.168.x). OPNsense má na WAN defaultně volby:
Block private networks
Block bogon networks
Unbound DNS → Query Forwarding (nutné!)

***TESTING SURICATY***
Promiscuous mode zapnut - v LABU.
aktivovány tyto rules:
- Feodo Tracker, ThreatFox, 3coresec

VLAN
- vytvořeny 20 a 30, DHCP připraveny
- nefungují, zkoušel jsem VLANY obejít skrze vlastní bridge = VM se nespouští s VLAN Tagem

**Síťová architektura a bezpečnostní perimetr**\
***Topologie a zóny***\
Perimetr je realizován pomocí OPNsense na Proxmoxu. Síť je rozdělena do zón:
WAN (externí síť / internet) – připojení přes Proxmox vmbr0 (uplink je ve VLAN 30 na straně WAN).
LAN (interní síť) – 192.168.99.0/24, gateway 192.168.99.1 (OPNsense), Proxmox vmbr4.
DMZ (veřejná zóna) – 192.168.20.0/24, gateway 192.168.20.1 (OPNsense).
SIEM (SOC zóna) – 192.168.30.0/24, gateway 192.168.30.1 (OPNsense).
Cíl rozdělení je oddělit veřejně dostupnou službu (DMZ) od interní sítě (LAN) a od logovací/detekční infrastruktury (SIEM).


***Síťová segmentace (VLAN / oddělené subnety)***\
Segmentace je řešena oddělenými subnety (LAN, DMZ, SIEM) a řízením provozu přes firewall pravidla v OPNsense.
Původně bylo plánováno použití VLAN, ale při implementaci se VLAN segmentace nepodařila stabilně zprovoznit (v některých konfiguracích po přidání VLAN přestával fungovat internet z interních VM). Z tohoto důvodu je segmentace v projektu doložena jako oddělené subnety s kontrolou komunikace na firewallu.

***L7 firewall a perimetr***\
OPNsense zajišťuje:
směrování mezi LAN/DMZ/SIEM,
NAT směrem do WAN,
filtraci provozu mezi zónami a na perimetru,
publikaci DMZ služby (port forward),
VPN přístup administrátora.
Pro vzdálený administrátorský přístup je připraven WireGuard:
- klíče vygenerovány, konfigurace připravena v OPNsense
- Ověření z externí sítě zatím nebylo možné provést (není k dispozici vhodný externí testovací přístup; zároveň záleží na omezeních upstream sítě na odchozí UDP a případném port forwardingu).

***Bezpečnostní pravidla (princip)***\
Použitý princip: least privilege.
WAN → DMZ: povolit pouze publikovaný port služby.
WAN → LAN/SIEM: zakázat.
DMZ → LAN: zakázat (default deny).
DMZ → SIEM: povolit jen nutné porty pro logování/agent komunikaci (později Wazuh).
LAN/VPN → (DMZ, SIEM): povolit pouze administraci.
