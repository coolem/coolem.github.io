---
layout: page
title:  "Un routeur sous ArchLinux et Raspberry Pi 2"
date:   2016-05-1 13:00:00 +0200
categories: raspi routeur arch linux wifi
lang : fr
---

### Contexte

Si la couverture est adéquate, on peut aujourd'hui tout à fait utiliser un réseau LTE (4G) pour surfer chez soi. Cependant, suivant la configuration du réseau domestique, le choix du modem peut s'avérer délicat :

- Un routeur WiFi c'est sympa, mais ça ne dispose pas de port ethernet.
- Un routeur classique avec modem 4G, c'est cher.
- Sur certains routeurs, d'Asus notamment, on peut brancher une clé USB 4G, mais sans vraiment savoir d'avance si le modèle sélectionné sera totalement pris en charge (cf forums).

Mon choix s'est porté vers une clé USB Huawei e3372, plus précisément e3372h-153, qu'on peut trouver à très bon prix en Chine. Le routeur sera basé sur un Raspberry Pi 2, même si un Odroid C2 conviendrait mieux (ethernet gigabit, 900mA pour chaque port USB). Le WiFi est un ajout sympathique, ici une clé Ralink fera l'affaire.

Côté Linux, le choix est large, ma préférence va à Arch, que je trouve trouve élégante, stable, efficace, et dont la documentation est d'excellente qualité. Cet article est un condensé de plusieurs pages de la documentation d'Arch [1](https://wiki.archlinux.org/index.php/Internet_sharing) [2](https://wiki.archlinux.org/index.php/software_access_point) ainsi que de pages trouvées sur le web [3](https://nims11.wordpress.com/2012/04/27/hostapd-the-linux-way-to-create-virtual-wifi-access-point/) [4](http://ubuntuforums.org/showthread.php?t=716192). Nous aurons besoin des paquets ou services suivants : netctl, dnsmasq, usb_modeswitch, iptables, hostapd.



### Installation

L'installation d'Arch sur un Raspberry Pi 2 est très bien expliqué sur le site [d'ArchLinux ARM](https://archlinuxarm.org/platforms/armv7/broadcom/raspberry-pi-2), rien de plus à ajouter.
Une fois installé, la première chose à faire lors du premier boot est d'installer le package **usb_modeswitch**, qui permettra de passer l'état du modem 4G vers le mode Hilink, et ainsi de créer le périphérique **eth1** (usbnet0 sur Odroid C2 en kernel 3.18 à l'heure de rédaction de cet article).



### Configuration des interfaces

Les interfaces seront configurées avec [netctl](https://wiki.archlinux.org/index.php/Netctl). Pour **eth0** (ethernet), le fichier, placé dans /etc/netctl, sera nommé ethernet-static et contiendra :

```
Interface=eth0
Connection=ethernet
IP=static
```

L'adresse IP pour l'ethernet sera attribuée au pont. Pour eth1, la clé 4G contient un routeur et donne fixe ainsi directement la valeur de l'interface eth1 par dhcp. Le contenu du fichier ethernet-4g sera ainsi :

```
Interface=eth1
Connection=ethernet
IP=static
```

Dans l'optique du routeur WiFi, il faudra également créer un pont **br0** reliant eth0 et wlan0 (interface WiFi), le fichier bridge. wlan0 n'est pas contenu dans ce fichier, il sera ajouté automatiquement par hostapd.

```
Description="Bridge"
Interface=br0
Connection=bridge
BindsToInterfaces=(eth0)
IP=static
Address='192.168.1.1/24'
```

Chaque interface doit être activée par netctl pour être configurée au démarrage :

```
netctl enable ethernet-static
```

ou, après un changement de configuration dans le fichier : 

```
netctl reenable ethernet-static
```



### Port forwarding
On établit ensuite la redirection des ports, dans le fichier /etc/sysctl.d/30-ipforward.conf

```
net.ipv4.ip_forward=1
net.ipv6.conf.default.forwarding=1
net.ipv6.conf.all.forwarding=1
```



### NAT
Le NAT sera géré par iptables. Dans shell, entrer ces différentes lignes :

```
# iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
# iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
# iptables -A FORWARD -i br0 -o eth1 -j ACCEPT
```

Puis sauvegarder l'état des iptables dans iptables.rules via :

```
iptables-save > /etc/iptables/iptables.rules
```

Et on active iptables : 

```
systemctl enable iptables
```





### Serveur DHCP
Le serveur DHCP sera géré par dnsmasq, configuré par /etc/dnsmasq.conf :

```
interface=br0 
expand-hosts      
domain=whatever
dhcp-range=192.168.1.10,192.168.1.40,255.255.255.0,1h 
```


### Le WiFi
A ce stade, la connexion devrait être partagée sur l'ethernet. Reste à s'occuper du WiFi. La première chose à faire est de déclarer deux interface virtuelles, comme expliqué [ici](https://wiki.archlinux.org/index.php/software_access_point).
Les interfaces sont donc ajoutées via [udev](http://superuser.com/questions/759542/how-to-permanently-add-wireless-interfaces-with-iw) :

```
ACTION=="add", SUBSYSTEM=="ieee80211", KERNEL=="phy1", \
    RUN+="/usr/bin/iw phy %k interface add vwlan_sta type station", \
    RUN+="/usr/bin/iw phy %k interface add vwlan_ap type __ap"
```

On active l'interface wlan0 via une règle dans sysctl, /etc/sysctl.d/90-wireless :

```
ip link set dev wlan_ap up
```

Vient la configuration de hostapd, via /etc/hostapd.conf : 

```
interface=wlan0
driver=nl80211
ssid=somessid
hw_mode=g
channel=6
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=3
wpa_passphrase=KeePGuessinG
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP

// Pour le mode 11n
wmm_enabled=1   
ieee80211n=1
ht_capab=[HT40-]  // 2x20MHz
```

Reste à activer hostapd : 

```
systemctl enable hostapd
```
