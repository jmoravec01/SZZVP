**Nastavení**\
Instalace OPNsense 25.7 ve VM300 (Moravec-OPNsense).
- dávat pozor na login "installer" jako první
- prvotní IP nastavena na 192.168.99.1 (staticky, později až budou VLANY přenastavíme)

VM Ubuntu VM301 (Moravec-Ubuntu).
- slouží pro GUI ovládání OPNsense

302VM (Moravec-Wazuh)\
303VM (Moravec-DMZ)

OPNsense nastavení:\
em0 = LAN\
em1 = WAN

***Chyby:***\
VLAN
- vytvořeny 20 a 30, DHCP připraveny
- nefungují, zkoušel jsem VLANY obejít skrze vlastní bridge = VM se nespouští s VLAN Tagem
```
no physical interface on bridge 'vmbr2'
kvm: -netdev type=tap,id=net0,ifname=tap303i0,script=/usr/libexec/qemu-server/pve-bridge,downscript=/usr/libexec/qemu-server/pve-bridgedown,vhost=on: network script /usr/libexec/qemu-server/pve-bridge failed with status 6400
TASK ERROR: start failed: QEMU exited with code 1
```
Kompletní reinstall OPNsense. Přešel jsem na vmbr4 -> 2 mi nefungovala (důvod neznámý).

***DŮLEŽITÉ***
Instituce často dávají na “WAN” RFC1918 adresu (10.x / 172.16–31 / 192.168.x). OPNsense má na WAN defaultně volby:
Block private networks
Block bogon networks
Unbound DNS → Query Forwarding (nutné!)
Opravilo internet.

***TESTING SURICATY***
Promiscuous mode zapnut - v LABU.
aktivovány tyto rules:
- Feodo Tracker, ThreatFox, 3coresec


***1. Síťová architektura a bezpečnostní perimetr***\
**Topologie a zóny**\
Perimetr je realizován pomocí OPNsense na Proxmoxu. Síť je rozdělena do zón:
WAN (externí síť / internet) – připojení přes Proxmox vmbr0 (uplink je ve VLAN 30 na straně WAN).
LAN (interní síť) – 192.168.99.0/24, gateway 192.168.99.1 (OPNsense), Proxmox vmbr4.
DMZ (veřejná zóna) – 192.168.20.0/24, gateway 192.168.20.1 (OPNsense).
SIEM (SOC zóna) – 192.168.30.0/24, gateway 192.168.30.1 (OPNsense).
Cíl rozdělení je oddělit veřejně dostupnou službu (DMZ) od interní sítě (LAN) a od logovací/detekční infrastruktury (SIEM).


**Síťová segmentace (VLAN / oddělené subnety)**\
Segmentace je řešena oddělenými subnety (LAN, DMZ, SIEM) a řízením provozu přes firewall pravidla v OPNsense.
Původně bylo plánováno použití VLAN, ale při implementaci se VLAN segmentace nepodařila stabilně zprovoznit (v některých konfiguracích po přidání VLAN přestával fungovat internet z interních VM). Z tohoto důvodu je segmentace v projektu doložena jako oddělené subnety s kontrolou komunikace na firewallu.

**L7 firewall a perimetr**\
OPNsense zajišťuje:
směrování mezi LAN/DMZ/SIEM,
NAT směrem do WAN,
filtraci provozu mezi zónami a na perimetru,
publikaci DMZ služby (port forward),
VPN přístup administrátora.
Pro vzdálený administrátorský přístup je připraven WireGuard:
- klíče vygenerovány, konfigurace připravena v OPNsense
- Ověření z externí sítě zatím nebylo možné provést (není k dispozici vhodný externí testovací přístup; zároveň záleží na omezeních upstream sítě na odchozí UDP a případném port forwardingu).

**Bezpečnostní pravidla (princip)**\
Použitý princip: least privilege.
WAN → DMZ: povolit pouze publikovaný port služby.
WAN → LAN/SIEM: zakázat.
DMZ → LAN: zakázat (default deny).
DMZ → SIEM: povolit jen nutné porty pro logování/agent komunikaci (později Wazuh).
LAN/VPN → (DMZ, SIEM): povolit pouze administraci.

***2. Detekce a monitoring bezpečnostních událostí***\
Plán
Pro detekci a monitoring bylo plánováno použít kombinaci:

Suricata (IDS/IPS) na firewallu OPNsense pro síťovou detekci anomálií,
Wazuh (SIEM/log server) jako centrální systém pro sběr, normalizaci a korelaci logů ze zdrojů (firewall, server, stanice).

Cílem bylo dosáhnout centrálního přehledu událostí a detekce typických anomálií (např. brute-force, scan/zakázané porty).

Realizace (co bylo nasazeno)
OPNsense byl nasazen jako perimetr a byl nakonfigurován IDS modul Suricata (Intrusion Detection).

Byl připraven server pro Wazuh jako centrální logovací systém (SIEM), avšak během realizace nebylo možné Wazuh plně propojit s OPNsense.

**Sběr logů z více zdrojů**\
Záměr byl sbírat logy minimálně ze dvou zdrojů:
firewall (OPNsense),
server (Ubuntu – SSH / web),
případně koncová stanice (VM).

V praxi se nepodařilo spolehlivě zprovoznit napojení firewall logů do Wazuh z důvodu problémů s kompatibilitou pluginu (viz níže). Sběr logů tak zůstal omezený a nebylo možné kompletně doložit centralizovaný přehled ze všech plánovaných zdrojů.

**Normalizace a přehled logů**\
Wazuh byl zvolen právě pro schopnost normalizace a dashboardového přehledu událostí. Z důvodu neúspěšného propojení s OPNsense však nebylo možné naplno předvést jednotnou normalizaci firewall událostí v centrálním systému.
Detekce anomálií (stav)

Suricata byla v OPNsense aktivní, ale při testování na provozu generovaném z Ubuntu VM nedocházelo k očekávané detekci (např. scan/brute-force). Suricata vykazovala pouze obecný síťový provoz (přenosy paketů), nikoli alerty odpovídající generovaným testům.
Z důvodu výše uvedených problémů nebylo možné spolehlivě prokázat detekci alespoň dvou typických bezpečnostních anomálií v požadované podobě.

***3. Automatizovaná reakce a alerting***\
*Zvolený koncept*\
Automatizovaná reakce měla být realizována přes Wazuh:
alert při splnění podmínky (např. brute-force / scan),
následná reakce (např. dočasné blokování IP na OPNsense, změna pravidla, zvýraznění incidentu).
*Realizace*\
Wazuh byl nasazen, nicméně nebyl dostatek času na kompletní integraci s OPNsense a zprovoznění plné automatizované reakce.
Hlavní technická překážka (Wazuh plugin v OPNsense)
Při pokusu o instalaci/provoz Wazuh pluginu na OPNsense docházelo k opakovaným problémům s verzemi:
plugin požadoval novější verzi balíčku (např. 25.7.11_9), i když byla nainstalovaná aktuální (např. 27.5.11 - bližší info verze v OPNsense jsem nenašel),
při pokusu o aktualizaci firmware se objevovalo („přání nového roku“) - pravděpodobně aktualizováno, což znemožnilo standardní aktualizační postup a zkomplikovalo troubleshooting.
Z těchto důvodů nebylo možné plně propojit OPNsense (firewall události) s Wazuh a navázat na to alerting/active response.

------------------------------------------------------------------------------------------------------------------

Architektura perimetru je realizována virtuálním firewallem OPNsense na Proxmoxu, který směruje provoz mezi zónami LAN/DMZ/SIEM a vynucuje bezpečnostní politiku na hranicích segmentů. Segmentace je provedena oddělenými L3 subnety a komunikace mezi nimi je řízena firewall pravidly v režimu default deny. Do internetu je provoz překládán pomocí NAT, veřejná služba v DMZ je publikována kontrolovaně přes port forward. Administrátorský přístup je plánován přes WireGuard VPN, aby bylo management rozhraní dostupné pouze z důvěryhodného kanálu.

Bezpečnostní opatření vychází z principu least privilege: z WAN je povolen pouze publikovaný port do DMZ, přístupy do LAN a SIEM jsou blokovány; DMZ nemá přístup do LAN, směrem do SIEM jsou povoleny pouze nutné porty pro logování/agent komunikaci. Na WAN rozhraní je zohledněno prostředí instituce (WAN může být RFC1918), proto je upravena konfigurace blokování privátních/bogon sítí a DNS forwardování tak, aby byla zachována konektivita.

Detekce je realizována pomocí IDS modulu Suricata na OPNsense a plánovaného SIEM řešení Wazuh. Suricata byla aktivní včetně vybraných threat-intel rulesetů, nicméně v rámci laboratorních testů nebyla spolehlivě prokázána detekce typických anomálií (scan/brute-force), což souvisí s volbou signatur a umístěním senzoru vůči testovanému provozu. Automatizovaná reakce (alerting/active response) byla navržena přes Wazuh, ale nebyla dokončena kvůli problémům s kompatibilitou pluginu a verzováním balíčků; jako další krok je vhodné uvažovat standardní integraci přes syslog a následnou korelaci a reakce na úrovni SIEM.
