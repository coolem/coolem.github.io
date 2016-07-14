---
layout: page
title:  "Oscilloscope USB sur Launchpad Tiva - DMA"
date:   2016-07-07 18:00:00 +0200
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


### Timer

```c		
SysCtlPeripheralEnable(SYSCTL_PERIPH_GPIOF);
GPIOPinTypeGPIOOutput(GPIO_PORTF_BASE, GPIO_PIN_3 | GPIO_PIN_2 | GPIO_PIN_1);
		
		
		
SysCtlPeripheralEnable(SYSCTL_PERIPH_WTIMER0);
    //
    // Configure Timer0B as a 16-bit periodic timer.
    //
    TimerConfigure(WTIMER0_BASE, TIMER_CFG_SPLIT_PAIR | TIMER_CFG_B_PERIODIC);

    //
    // Set the Timer0B load value to 1ms.
    //
    TimerLoadSet(WTIMER0_BASE, TIMER_B, 0x4C4B400/100);

    //
    // Enable processor interrupts.
    //
    IntMasterEnable();

    //
    // Configure the Timer0B interrupt for timer timeout.
    //
    TimerIntEnable(WTIMER0_BASE, TIMER_TIMB_TIMEOUT);
		
		TimerControlTrigger(WTIMER0_BASE, TIMER_B, true);

```


### Contrôleur DMA


```c
SysCtlPeripheralEnable(SYSCTL_PERIPH_UDMA);
SysCtlPeripheralSleepEnable(SYSCTL_PERIPH_UDMA);

IntEnable(INT_UDMAERR);

uDMAEnable();

uDMAControlBaseSet(ui8ControlTable);
```




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
	//HWREG(ADC0_BASE+ADC_O_SSCTL3)|= (1<<6);
}
```




## Gestion des interruptions


```c
void ADC0SS3IntHandler()
{
	TimerDisable(WTIMER0_BASE, TIMER_B);
    TimerIntClear(WTIMER0_BASE, TIMER_TIMB_TIMEOUT);
	g_ui32Counter = 0;
	a=0;
	GPIOPinWrite( GPIO_PORTF_BASE, GPIO_PIN_3 | GPIO_PIN_2 | GPIO_PIN_1, 0 );	
}
```



```c
void UDMAIntHandler()
{
	UARTprintf("Done\n");
	for(int i = 0; i<10; i++)
	{
		//UARTprintf("%d %d\n", (const char) g_ui32SrcBuf[i], (const char) g_ui32DstBuf[i]);
	}
	UARTprintf("Printed\n");
	
	void UDMA_ERRIntHandler()
	{
		UARTprintf("Fail\n");
	}
}
```