> Study case: Linux version 5.15.0 on OpenBMC

## Index

- [Introduction](#introduction)

- [Reference](#reference)

## <a name="introduction"></a> Introduction

Hardware components send out interrupts to inform all kinds of events, e.g., timer tick, data arrival on a network card, users input a character. 
Unlike instruction SWI, these device events are asynchronous, and current tasks don't trigger them directly. 
Once the CPU receives an interrupt, it sequentially switches to IRQ and SVC mode and then calls the interrupt service routine (ISR) to handle the event properly. 
This chapter will introduce this mechanism with an example and talk about software interruptions.

## <a name="reference"></a> Reference




