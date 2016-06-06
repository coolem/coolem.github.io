---
layout: page
title:  "Oscilloscope USB sur Launchpad Tiva - Timer"
date:   2016-06-06 18:00:00 +0200
categories: tiva oscilloscope adc timer arm
lang : fr
---

### L'idée

Pour fabriquer un oscilloscope, il nous faut acquérir plusieurs valeurs à l'aide de l'ADC, chaque valeur étant de préférence séparée par un intervalle de temps précis. Rien de mieux donc que de fixer cet intervalle avec un timer.

Le timer étant relié à la fréquence du CPU, nous allons fixer la fréquence du Cortex M4 à 80 MHz, afin de disposer d'une fréquence d'acquisition élevée si besoin.
L'acquisition sera lancée lors de la génération d'une interruption par le timer lorsque celui-ci passe en timeout.


<br/>




### Initialisation

Utiliser le TM4C à 80 MHz, c'est sympa, mais avec un 1 tick par clock on se rend compte qu'un timer 16 bit ne tiendra que 0.8 ms. Dit autrement, la fréquence d'acquisition minimale sera de 1220 Hz. Il nous faut donc utiliser un timer 32 bit, soit un timer 32 bit complet, ou par exemple le **WTIMER0, subtimer B**.

L'initialisation du timer repose sur le code suivant :

```c
SysCtlPeripheralEnable(SYSCTL_PERIPH_WTIMER0);
TimerConfigure(WTIMER0_BASE, TIMER_CFG_SPLIT_PAIR | TIMER_CFG_B_PERIODIC);
TimerLoadSet(WTIMER0_BASE, TIMER_B, 0x4C4B400/interval_time);
IntMasterEnable();
TimerIntEnable(WTIMER0_BASE, TIMER_TIMB_TIMEOUT);
TimerControlTrigger(WTIMER0_BASE, TIMER_B, true);
```

Il s'agit donc d'activer le timer via **SysCtlPeripheralEnable()**, le configurer avec **TimerCOnfigure()**.
On fixe ensuite la limite du timer avec **TimerLoadSet()**, en divisant 80000000 ticks (0x4C4B400) par la durée souhaitée d'un intervalle. Puis on active l'interruption générale, **IntMasterEnable()** ainsi que l'interruption liée au timer en timeout (**TimerIntEnable()**).
On configure enfin le trigger du timer.

L'interruption générée par le timer est récupérée par la fonction **WTimer0IntHandler()**, qu'il convient de déclarer dans la table d'interruptions du core (fichier startup en assembleur). 

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
    ;DCD     IntDefaultHandler           ; Wide Timer 0 subtimer B
    DCD	     WTimer0BIntHandler
```

<br/>

### Interruption

Après avoir déclaré le vecteur d'interruption dans la table, il s'agit maintenant d'exploiter l'interruption.
On distingue deux parties dans le code suivant :
- La partie récurrente à chaque interruption.
- Un test sur un compteur.

Lors de chaque interruption du timer, il faut effacer l'interruption du timer (**TimerIntClear()**) afin de garder le maximum de précision dans la mesure des intervalles.
On active ensuite la séquence de l'ADC via **ADCSequenceEnable()**, puis l'acquisition **ADCProcessorTrigger()**.

A la finc de l'acquisition, le résultat est enregistré dans le tableau **tmp**, à la case désignée par **g_ui32Counter**, qui aura préalablement été remis à zéro avant le lancement du timer.

```c
void WTimer0BIntHandler(void)
{
  TimerIntClear(WTIMER0_BASE, TIMER_TIMB_TIMEOUT);
  ADCSequenceEnable(ADC0_BASE, 3);
  ADCProcessorTrigger(ADC0_BASE, 3);
  
  while(!ADCIntStatus(ADC0_BASE, 3, false)) {; }
  tmp[ g_ui32Counter ] = HWREG(ADC0_BASE + ADC_O_SSFIFO3);
  
  g_ui32Counter++;

  if(g_ui32Counter >= (SSIZE) )
  {
        TimerIntDisable(WTIMER0_BASE, TIMER_TIMB_TIMEOUT);
        TimerIntClear(WTIMER0_BASE, TIMER_TIMB_TIMEOUT);
  }
}
```

Le test vérifie si le compteur est arrivé au nombre d'acquisitions souhaitées par SSIZE. Si oui :
- On désactive l'interruptions en timeout du timer.
- On efface le drapeau d'interruption en timout du timer.

Nous pouvons ainsi lancer des acquisitions séparées par des intervalles bien précis. Cependant le processeur travaille encore un peu trop à mon goût. La troisième partie sera donc dédiée à la mise en place du DMA.
