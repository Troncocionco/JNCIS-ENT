# MSTP - Multiple Spanning Tree Protocol 802.1Q-2003

Rappresenta sostanzialmente una estensione di RSTP (Rapid Spanning Tree Protocol) mappando istanze multiple/indipendenti di Spanning Tree Instances STI su di un'unica topologia fisica.

_Nota_: Ogni STI contiene al suo interno una o più VLAN

**MSTP**: Ogni **MSTI** (multiple spanning-tree instances) crea uno più "_topology-tree_", associabili ad una o più VLAN, distribuendo in modo bilanciato il traffico attraverso tutti i possibili link.

MSTP permette non solo di assegnare una porta a più VLAN ma anche di aver tale porta _blocked_ in un topology tree e in stato _forwarding_ in un secondo.

MSTP adotta anche le modifiche apportate da RSTP relative alla compressione dei tempi di riconvergenza.

## MST Region
Gruppo di switch all'interno della stessa region e che quindi condividono i seguenti parametri:
- region name
- revision level
- VLAN-to-instance mapping

Ogni Region MST può contenere fino ad un massimo di 64 MSTI.

MSTP efficienta lo scambio di msg BPDU, concentrando tutte le info per i vari MSTI in un unico BPDU.MSTP elegge un nodo root per ciascun MSTI sulla base della Bridge Priority.

**CST - Common Spanning Tree** permette l'interconnessione di multiple regioni MST ma anche di interconnettere queste ultime a switch stand-alone running STP legacy (retrocompatibile).

Ogni root node di un MSTI è responsabile del calcolo del topology tree verso il CST. In sostanza ogni MST region è trattata come se fosse un **Virtual Bridge**, indipendentemente dal numero di switch dietro di esso.

MTSP codifica le informazioni relative alla _region_ (MSTI messages) subito dopo il classico RSTP BDPU, questo mostra come in gran parte l'header MSTP sia sovrapponibile con RSTP e retrocompatibile con esso.

I primi 13 campi del BDPU MSTP contengono informazioni similari al BPDU di RSTP.

Alcuni parametri devono essere consistenti traversalmente su tutti i device facenti parti della stessa _region_ MSTP:
- Configuration Name
- Revision Level
- MSTI-to-vlan mapping (corrispondenza tra l'ID del MSTI e le VLAN che questo dovrà servire)

_Nota_:tutte le istanze MSTI che non sono mappate vengono di default assegnate alla MSTI 0
```
lab@Brandy> show spanning-tree mstp configuration
MSTP information
Context identifier : 0
Region name : book-example
Revision : 1
Configuration digest : 0x53973bbb358bdb2d6dcab806a189064f
MSTI Member VLANs
0 0-9,11-19,21-29,31-4094
10 10,20
30 30
30 30
```

