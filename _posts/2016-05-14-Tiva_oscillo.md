---
layout: post
title:  "Oscilloscope USB sur Launchpad Tiva - 1ère partie"
date:   2016-05-14 18:00:00 +0200
categories: tiva oscilloscope adc timer arm
lang : fr
---

### Mais pourquoi? Pourquoi???
Pourquoi passer du temps à coder son oscilloscope USB alors qu'on peut en trouver pour 30€ sur des sites chinois? Pour le plaisir bien sûr!
Mais également l'espoir d'obtenir un produit plus performant.

### Parce-que...
...le TM4C équipant le Launchpad Tiva possède 2 ADC, chacun :
- capable d'acquérir 500ksps.
- travaillant sur 12 bits.
- avec une entrée simple ou différentielle.

Puis le TM4C est basé sur un Cortex M4F cadencé jusqu'à 80 MHz, ce qui laisse pas mal de marge de traitement derrière.

### Mise en oeuvre de l'ADC

