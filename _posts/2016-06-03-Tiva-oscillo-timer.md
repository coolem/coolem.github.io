---
layout: page
title:  "Oscilloscope USB sur Launchpad Tiva - Timer"
date:   2016-06-03 18:00:00 +0200
categories: tiva oscilloscope adc timer arm
lang : fr
---

### L'idée

Pour fabriquer un oscilloscope, il nous faut acquérir plusieurs valeurs à l'aide de l'ADC, chaque valeur étant de préférence séparée par un intervalle de temps précis. Rien de mieux donc que de fixer cet intervalle avec un timer.

Le timer étant relié à la fréquence du CPU, nous allons fixer la fréquence du Cortex M4 à 80 MHz, afin de disposer d'une fréquence d'acquisition élevée si besoin.
L'acquisition sera lancée lors de la génération d'une interruption par le timer lorsque celui-ci passe en timeout.


<br/>

### Initialisation

```c
SysCtlPeripheralEnable(SYSCTL_PERIPH_WTIMER0);
TimerConfigure(WTIMER0_BASE, TIMER_CFG_SPLIT_PAIR | TIMER_CFG_B_PERIODIC);
TimerLoadSet(WTIMER0_BASE, TIMER_B, 0x4C4B400/100);
IntMasterEnable();
TimerIntEnable(WTIMER0_BASE, TIMER_TIMB_TIMEOUT);
TimerControlTrigger(WTIMER0_BASE, TIMER_B, true);
```

```c
;******************************************************************************
;
; External declarations for the interrupt handlers used by the application.
;
;******************************************************************************
    EXTERN  SysTickIntHandler
		EXTERN  WTimer0BIntHandler
		EXTERN	UDMAIntHandler
		EXTERN	UDMA_ERRIntHandler
		
;******************************************************************************
;
; The vector table.
;
;******************************************************************************
    ;DCD     IntDefaultHandler           ; Wide Timer 0 subtimer A
		DCD		WTimer0BIntHandler
```


### Interruption

```c
void WTimer0BIntHandler(void)
{
  TimerIntClear(WTIMER0_BASE, TIMER_TIMB_TIMEOUT);
  ADCSequenceEnable(ADC0_BASE, 3);
  ADCProcessorTrigger(ADC0_BASE, 3);
  
  g_ui32Counter++;

  if(g_ui32Counter >= (SSIZE/2) )
  {
  }
}
```
