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


### Environnements de programmation
Le TM4C est supporté par [Energia](http://energia.nu), alternative à Arduino. Il s'agit là d'une possibilité intéressante pour débuter l'apprentissage de la plateforme... ou vérifier rapidement que tout fonctionne.
L'idée ici est d'aller plus loin en utilisant "sérieusement" le SoC. Ceci est possible via Energia, mais on préfèrera utiliser un IDE digne de ce nom (IAR, Keil, DS-5, etc), ou suivant les goûts un bon éditeur de texte et des Makefile.

Ici nous utiliserons Keil couplé au compilateur ARM (par défaut). Celui-ci est généralement payant, cependant son utilisation avec Keil est gratuite dans la limite d'un binaire de 32kB. Il est possible d'utiliser GCC, qui a l'immense avantage d'être gratuit, produit du code de bonne qualité, mais est moins bien supporté que le compilateur maison d'ARM. Nous verrons l'utilisation de GCC dans un prochain post.

Côté développement, 3 étages sont disponibles :

* Utiliser directement les registres.
* Utiliser les intrinsics.
* Utiliser les pilotes fournis par Texas Instruments (Tivaware).

Utiliser le pilote rend le code plus lisible et facile à développer, mais augmente l'overhead. Personnellement je mixe l'utilisation des pilotes et des registres suivant le cas.



### Un programme Keil avec le compilateur ARM
Ce programme sera très simple : Allumer une couleur de la LED RGB.
La première étape est de fixer la fréquence du CPU.
![TM4C clock system]({{ site.url }}/img/tiva_clock.png)
Le schéma précédent présente la gestion des horloges des périphériques composant le TM4C. Dans le cas présent, le quartz de 16 MHz est attaché aux deux bornes du **Main OSC**. Nous sélectionnerons ce dernier pour alimenter le **PLL (400 MHz)**. **DIV400**  fournit en entrée du **SYSDIV** 200 MHz ou 400 MHz, suivant la configuration des registres associés. Pour plus de simplicité, la combinaison de ces deux éléments est gérée par les pilotes via la fonction **SysCtlClockSet()**. La fréquence résultante sera la fréquence système (CPU, timers). Au niveau du programme, nous fixeront donc la fréquence système à 80 MHz via :

```c
SysCtlClockSet(SYSCTL_SYSDIV_2_5 | SYSCTL_USE_PLL | SYSCTL_OSC_MAIN | SYSCTL_XTAL_16MHZ);
```
L'étape suivante consiste à allumer le Port F du GPIO.
```c
SysCtlPeripheralEnable(SYSCTL_PERIPH_GPIOF);
```
Puis à configurer les pins du Port F câblés à la LED en sortie (registre GPIODIR).
```c
GPIOPinTypeGPIOOutput(GPIO_PORTF_BASE, GPIO_PIN_3 | GPIO_PIN_2 | GPIO_PIN_1);
```
Enfin, on allume la LED rouge (registre GPIODATA).
```c
GPIOPinWrite( GPIO_PORTF_BASE, GPIO_PIN_3 | GPIO_PIN_2 | GPIO_PIN_1, GPIO_PIN_1 );
```
