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

<br/>

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

  ADCSequenceConfigure(ADC0_BASE, 3, ADC_TRIGGER_PROCESSOR, 0);	// Sélection du séquenceur 3, trigger par processeur 
  ADCSequenceStepConfigure(ADC0_BASE, 3, 0, ADC_CTL_IE | ADC_CTL_END | ADC_CTL_CH1);	// Config du séquenceur 3
  											// Interruption active à la fin de la séquence
  											// End : Dernière étape de la séquence 
  											// Entrée canal 1 (PE2)
}
```

Les 5 premières lignes du code sont relativement simples. Les fonctions**HWREG()** sont utilisées pour écrire directement la valeur souhaitée dans le registre désiré (on donne l'adresse mémoire du registre en argument de HWREG).
Les deux dernières lignes concernant la configuration de la séquence ADC sont plus compliquées. Pour comprendre cette partie il est nécessaire de visualiser le fonctionnement d'un module ADC :

![Launchpad Tiva ADC blocks]({{ site.url }}/img/tiva_adc_details.png)

Un module ADC est composé de 4 séquenceurs, chacun déclenché via des événements provenant du timer, d'un pin, du PWM ou d'un comparateur de tension (via ADC). Les séquenceurs n'ont pas les mêmes propriétés.

|Séquenceur | Echantillons | FIFO |
|-----------|--------------|------|
|SQ3        | 1            |  1   |
|SQ2 | 4 | 4 |
|SQ1 | 4 | 4 |
|SQ0 | 8 | 8 |

Les différences portent sur l'échantillonage de chaque séquenceur, et la taille du buffer FIFO associé. Ainsi le séquenceur 0 échantillonera 8 valeurs par acquisition, contre une seule pour le séquenceur 3. Pour nos besoins, nous utiliserons ici le séquenceur 3.

Ainsi la fonction **ADCSequenceConfigure()** configure le séquenceur, ici le 3 (adresse mémoire de l'ADC0 en premier argument, puis troisième séquenceur).
Le troisième argument de la fonction concerne le trigger, qui peut être un évènement provenant en autres d'un timer, d'un comparateur, ou ici d'une commande générée par le CPU.
Le dernier argument prend en charge la priorité de l'acquisition, de 0 à 3.

Enfin, la fonction **ADCSequenceStepConfigure()** configure une étape du séquenceur, ici une seule étape (étape 0 en troisième argument) du séquenceur 3. On active la génération d'une interruption à la fin de la séquence (ADC_CTL_IE), la fin de la séquence avec cette étape (ADC_CTL_END), et enfin l'acquisition sur le canal 1 (ADC_CTL_CH1) sur le pin PE2.



### Lancement de l'acquisition
On lance l'acquisition par la fonction **ADCProcessorTrigger()**, dont les paramètres sont l'addresse de l'ADC0 puis le troisième séquenceur.

```
ADCProcessorTrigger(ADC0_BASE, 3);			// CPU trigger

while(!ADCIntStatus(ADC0_BASE, 3, false)) { }		// Wait until the sample sequence has completed.
tmp = HWREG(ADC0_BASE + ADC_O_SSFIFO3);			// Read result from register
```

Puis on attend via polling que l'ADC complète son acquisition via le statut de l'interruption.
Enfin, on lit le résultat directement du registre via la fonction HWREG();

Cette technique est assez basique et peu efficace. Nous verrons lors s'un prochain article une manière plus élégante de travailler avec l'ADC en utilisant les interruptions.
