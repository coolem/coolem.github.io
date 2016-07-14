---
layout: page
title:  "Oscilloscope USB sur Launchpad Tiva - DMA"
date:   2016-07-14 18:00:00 +0200
categories: tiva oscilloscope adc timer arm
lang : fr
---

## ADC et Timer

Précédemment, chaque interruption en timeout du timer générait les événements suivants:

- Activation de l'ADC par le CPU
- Attente de la fin de l'acquisition par polling.
- Transfert de la valeur du registre de l'ADC en mémoire.
- Incrémentation du compteur.
- Test puis désactivation du timer si but atteint.

Tout ceci demande trop d'intervention du processeur. Une meilleure voie serait donc d'automatiser tout le processus, de façon à ce que les périphériques communiquent directement entre eux.
Quelle chance nous avons : le TivaC permet tout ça! La marche à suivre serait donc la suivante.

- Un timeout du timer déclenche directement l'acquisition de l'ADC.
- A la fin de la conversion, le registre de l'ADC est transféré directement dans la mémoire via DMA.
- Le processus continue jusqu'à ce que le contrôleur DMA aie reçu la quantité de données requises.



## Initialisation des périphériques

### ADC

```c
SysCtlPeripheralEnable(SYSCTL_PERIPH_ADC0);
HWREG(SYSCTL_RCGCADC) = 1;
HWREG(SYSCTL_RCGCGPIO) = 0x10;
		
SysCtlPeripheralEnable(SYSCTL_PERIPH_GPIOE);
GPIOPinTypeADC(GPIO_PORTE_BASE, GPIO_PIN_2);

ADCSequenceConfigure(ADC0_BASE, 3, ADC_TRIGGER_TIMER, 0);
ADCSequenceStepConfigure(ADC0_BASE, 3, 0, ADC_CTL_IE | ADC_CTL_END | ADC_CTL_CH1); 
```

L'initialisation du périphérique ADC est la même qu'au [premier chapitre](http://localhost:4000/tiva/oscilloscope/adc/arm/2016/05/14/Tiva_oscillo_adc.html).
A l'exception de la fonction **ADCSequenceConfigure()** où l'argument ADC_TRIGGER_PROCESSOR est remplacé par ADC_TRIGGER_TIMER.
Ainsi l'acquisition de l'ADC est lancée directement par un évènement provenant du timer.



### Timer

```c		
SysCtlPeripheralEnable(SYSCTL_PERIPH_WTIMER0);
TimerConfigure(WTIMER0_BASE, TIMER_CFG_SPLIT_PAIR | TIMER_CFG_B_PERIODIC);
TimerLoadSet(WTIMER0_BASE, TIMER_B, 0x4C4B400/interval_time);
IntMasterEnable();
TimerIntEnable(WTIMER0_BASE, TIMER_TIMB_TIMEOUT);
TimerControlTrigger(WTIMER0_BASE, TIMER_B, true);
```

L'initialisation du timer est également la même qu'au [précédent chapitre](http://localhost:4000/tiva/oscilloscope/adc/timer/arm/2016/06/06/Tiva_scillo_timer.html).



### Contrôleur DMA

```c
SysCtlPeripheralEnable(SYSCTL_PERIPH_UDMA);
IntEnable(INT_UDMAERR);
uDMAEnable();
uDMAControlBaseSet(ui8ControlTable);
```

Comme pour les autres périphériques, le périphérique uDMA doit être activé, ici via la fonction **SysCtlPeripheralEnable()**.
On active ensuite l'interruption d'erreur du contrôleur DMA, INT_UDMAERR, juste au cas où...
Puis on active le contrôleur uDMA lui-même avec **uDMAEnable()**.
Enfin, la fonction **uDMAControlBaseSet()** configure la table de contrôle du contrôleur.
La table de contrôle est en quelque sorte un registre géant qui contient 

- Les pointeurs de départ et de destination.
- La taille des transferts.
- Le mode de transfert pour chaque canal.

La table de contrôle, alignée sur 1kB, est déclarée comme tel :

```c
uint8_t ui8ControlTable[1024] __attribute__ ((aligned(1024)));
```

<br><br>






## Initialisation du canal du DMA


```c
void AdcDmaInit()
{
	uDMAChannelAttributeDisable(UDMA_CHANNEL_ADC3, UDMA_ATTR_ALL);
	
	uDMAChannelAssign(UDMA_CH17_ADC0_3);
	
	uDMAChannelControlSet(UDMA_CHANNEL_ADC3 | UDMA_PRI_SELECT,
                           UDMA_SIZE_16 | UDMA_SRC_INC_NONE | UDMA_DST_INC_16 |
                           UDMA_ARB_1);
	
	uDMAChannelTransferSet(UDMA_CHANNEL_ADC3 | UDMA_PRI_SELECT,
                            UDMA_MODE_BASIC,  (void*)(ADC0_BASE+ADC_O_SSFIFO3), g_ui16DstBuf,
                            127);

	uDMAChannelAttributeEnable(UDMA_CHANNEL_ADC3,UDMA_ATTR_USEBURST);
															 
	uDMAChannelEnable(UDMA_CHANNEL_ADC3);
															 
	IntEnable(INT_ADC0SS3);
}
```


<br><br>







## Gestion des interruptions

La gestion des interruptions est différente des précédents chapitres. Le timer envoie directement un signal à l'ADC.
L'interruption de l'ADC est toujours présente mais modifiée : La fonction suivante n'est exécutée que lorsque le contrôleur uDMA a terminé son travail.

```c
void ADC0SS3IntHandler()
{
TimerDisable(WTIMER0_BASE, TIMER_B);
TimerIntClear(WTIMER0_BASE, TIMER_TIMB_TIMEOUT);
}
```

Ainsi à la fin de la série de mesures le timer est désactivé, et l'interruption du timer remise à zéro.


```c
void UDMAIntHandler()
{
	UARTprintf("Done\n");
}	

void UDMA_ERRIntHandler()
{
	UARTprintf("Fail\n");
}

```

On peut également récupérer les signaux d'interruption du contrôleur uDMA via **UDMAIntHandler()**, ainsi que les interruptions sur erreur via **UDMA_ERRIntHandler()**.
