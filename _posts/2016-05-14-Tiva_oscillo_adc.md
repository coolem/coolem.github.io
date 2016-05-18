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

![Launchpad Tiva ADC blocks]({{ site.url }}/img/tiva_adc_blocks.png)

Il nous sera donc possible de lancer une acquisition sur 2 canaux, ou un seul canal en fréquence double (décalage de phase de l'acquisition).
Qui plus est, le TM4C est basé sur un Cortex M4F cadencé jusqu'à 80 MHz, ce qui laisse une certaine puissance de traitement derrière.

### Mise en oeuvre de l'ADC
Nous aurons besoins de la base développée lors de l'[introduction au Launchpad Tiva](http://www.coolem.tech/launchpad/tiva/keil/arm/2016/05/14/Tiva-armcc.html) pour initialiser le SoC. Nous pouvons alors créer une fonction d'initialisation de l'ADC.

```c
void InitADC()
{
  SysCtlPeripheralEnable(SYSCTL_PERIPH_ADC0);     // Activation de l'ADC
	
  HWREG(SYSCTL_RCGCADC) = 1;	// Activation de l'horloge de l'ADC
  HWREG(SYSCTL_RCGCGPIO) = 0x10;	// Activation de l'horloge du PORT E

  SysCtlPeripheralEnable(SYSCTL_PERIPH_GPIOE);    // Activation du PORT E
  GPIOPinTypeADC(GPIO_PORTE_BASE, GPIO_PIN_2);    // PORT E PIN2 assigné à l'ADC

  ADCSequenceConfigure(ADC0_BASE, 3, ADC_TRIGGER_TIMER, 0);	// Sélection du séquenceur 3, trigger par processeur 
  ADCSequenceStepConfigure(ADC0_BASE, 3, 0, ADC_CTL_IE | ADC_CTL_END | ADC_CTL_CH1);	// Config du séquenceur 3
  											// Interruption active à la fin de la séquence
  											// End : Dernière étape de la séquence 
  											// Entrée canal 1 (PE2)
}
```
