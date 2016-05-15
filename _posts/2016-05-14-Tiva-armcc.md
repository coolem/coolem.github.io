---
layout: page
title:  "Introduction au Launchpad Tiva (Keil & ARM compiler)"
date:   2016-05-14 13:00:00 +0200
categories: launchpad tiva keil arm
lang : fr
---

### Le Launchpad Tiva
Le Launchdad Tiva-C (TM4C123GXL) remplace le Launchpad Stellaris, en y ajoutant principalement le support de l'USB Host.
![Launchpad Tiva]({{ site.url }}/img/launchpad-tivac-01.jpg)
Je laisse les détails de la plateforme à [Wikipedia](https://en.wikipedia.org/wiki/Tiva-C_LaunchPad) pour vous citer l'essentiel:

* Un SoC TM4C123GH6PM
* Un core ARM Cortex M4F (F pour unité de calcul des nombres à virgule flottante) cadencé 80 MHz.
* 32 kB de SRAM
* 256 kB de Flash
* 2 kB d'EEPROM
* 4 ports I2C
* Un contrôleur uDMA
* 2 ADC 12 bits @ 500ksps chacun
* Un port microUSB sur la carte
* au moins 2 UART
* 40 pins sur la carte

Bref, de quoi s'amuser pendant un moment. L'image suivante résume le pinout de la carte :
![Launchpad Tiva card pinout]({{ site.url }}/img/LaunchPads-LM4F-TM4C-Pins-Maps-11-32.jpg)

### Programmation
Le TM4C est supporté par [Energia](http://energia.nu), alternative à Arduino. Il s'agit là d'une possibilité intéressante pour débuter l'apprentissage de la plateforme... ou vérifier rapidement que tout fonctionne.
L'idée ici est d'aller plus loin en utilisant "sérieusement" le SoC. Pour ce faire, 2 instruments sont indispensables :

* Un éditeur de texte
* Un compilateur

Côté éditeur de texte, le choix est infini ou presque. Côté compilateur, le choix est plus limité, les deux plus utilisés étant GCC et ARM. Les deux nous laissent le choix d'écrire en C ou C++. GCC a l'immense avantage d'être gratuit, produit du code de bonne qualité, mais est moins bien supporté que le compilateur maison d'ARM. Ce dernier est généralement payant, cependant son utilisation est gratuite dans certaines limites.

Nous verrons l'utilisation de GCC dans un prochain post, et utiliserons le compilateur ARM aujourd'hui. On trouve celui-ci dans trois IDE : IAR, Keil et ARM DS-5, les deux derniers étant appartenant à ARM.

