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

Côté Linux, le choix est large, ma préférence va à Arch, que je trouve trouve élégante, stable, efficace, et dont la documentation est d'excellente qualité. Cet article est un condensé de plusieurs pages de la documentation d'Arch ainsi que de pages trouvées sur le web. Nous aurons besoin des paquets ou services suivants : netctl, dnsmasq, usb_modeswitch, iptables.

### Installation
L'installation d'Arch sur un Raspberry Pi 2 est très bien expliqué sur le site {https://archlinuxarm.org/platforms/armv7/broadcom/raspberry-pi-2}[d'ArchLinux ARM], rien de plus à ajouter.
Une fois installé, la première chose à faire lors du premier boot est d'installer le package **usb_modeswitch**, qui permettra de passer l'état du modem 4G vers le mode Hilink, et ainsi de créer le périphérique **eth1** (usbnet0 sur Odroid C2 en kernel 3.18 à l'heure de rédaction de cet article).
