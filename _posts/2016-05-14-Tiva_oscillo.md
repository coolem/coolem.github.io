---
layout: page
title:  "Oscilloscope USB sur Launchpad Tiva - ADC"
date:   2016-05-14 18:00:00 +0200
categories: tiva oscilloscope adc arm
lang : fr
---

### Pourquoi???
Pourquoi passer du temps à coder son oscilloscope USB alors qu'on peut en trouver pour 30$ sur des sites chinois? Pour le plaisir bien sûr! Apprendre aussi, et également l'espoir d'obtenir un produit plus performant.

### Parce que...
...contrairement à bien des oscilloscopes pas chers, le TM4C équipant le Launchpad Tiva possède 2 ADC, chacun :

* capable d'acquérir 500ksps.
* travaillant sur 12 bits.
* avec une entrée simple ou différentielle.

Il nous sera donc possible de lancer une acquisition sur 2 canaux, ou un seul canal en fréquence double (décalage de phase de l'acquisition).
Qui plus est, le TM4C est basé sur un Cortex M4F cadencé jusqu'à 80 MHz, ce qui laisse une certaine puissance de traitement derrière.

### Mise en oeuvre de l'ADC

