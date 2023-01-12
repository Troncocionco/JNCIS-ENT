## High Availability Features

Redundant HW che possiamo trovare nei device che montano Junos OS (specifico per ogni famiglia e modello):
- Routing Engine
- Power Supplies
- Cooling Fans
- Control Boards (CBs)

In particolare gli switch della famiglia EX supportano le seguenti funzionalità in termini di HA:
- **LAGs** Link Aggregation Groups
- **RTGs** Redundant Trunk Groups
- **GRES** Graful Routing Engine Switchover
- **NSR** Non-stop (active) Routing
- **NSB** Non-stop Bridging

#### LAG
Combinare multiple interfacce Ethernet in una sola interfaccia L2 (standard _802.3ad_). Spesso utilizzate per aggregare link che interconnettono _access-switch_ e _aggregation-switch_.

- Incrementa la banda complessiva, l'efficienza del link (traffico bilanciato su ciascuno dei partecipanti del bundle), crea ridondanza fisica

- Requirements e condizioni per le singole interfacce partecipanti: duplex e speed configuration cdeve essere coerente su ambo i lati, fino a 16 links partecipanti, le porte partecipanti possono anche non essere contigue

- Il traffico generato dal RE (info relative ai protocolli di routing) viene instradato sul link con ID più basso (eg. ge-0/0/1)

- Il restante traffico IP è instradato in load balancing automatico scegliendo il link in base alla funzione di hashing su input dei dettagli L2, L3 e L4

- Traffico L2 pure effettua l'hashing su MAC address destination e source

Crea una interfaccia Ethernet Bundle (1 LAG con nome "ae0").

```
{master: 0} [edit chassis]
user@Switch-1# set aggregated-devices ethernet device-count 1

{master: 0} [edit chassis]
user@Switch-1# commit
configuration check succeeds
commit complete

{master: 0} [edit chassis]
user@Switch-1# run show interfaces terse | match ae0
ae0                     up      down
```

_Nota_: l'interfaccia rimane down fino a che non vengono inseriti link membri nel bundle.

Rispettivamente imposta l'interfaccia "ae0" per traffico L2 come parte della Vlan default, attiva LACP sul nodo locale, inserisce le interfacce 12 e 13 nella LAG ae0.
```
{master: 0} [edit interfaces]
user@Switch-1# set ae0 unit 0 family ethernet-switching vlan members default

{master: 0} [edit interfaces]
user@Switch-1# set ae0 aggregated-ether-options lacp active

{master: 0} [edit interfaces]
user@Switch-1# set ge-0/0/12 ether-options 802.3ad ae0

{master: 0} [edit interfaces]
user@Switch-1# set ge-0/0/13 ether-options 802.3ad ae0

{master: 0} [edit interfaces]
user@Switch-1# commit
configuration check succeeds
commit complete

{master: 0} [edit interfaces]
user@Switch-1# run show interfaces terse | match ae0
ge-0/0/12           up  up  aenet   --> ae0.0
ge-0/0/13           up  up  aenet   --> ae0.0
ae0                 up  up
ae0.0               up  up  eth-switch
```

#### RTG
Fornisce un meccanismo semplice e veloce per il **failover** di link L2 ridondati. Spesso viene utilizzato su access-layer switches che sono connessi a due diversi aggregation-layer switch, in sostituzione di protocolli com STP (dati i tempi convergenza rapidissimi > 1s dopo la detection del fault).

_Nota_: non è possibile configurare contemporaneamente RTG e STP sulle stesse porte!
Il BPDU STP ricevuti su interfacce RTG sono scartati.

_Nota_: generalmente **STP è configurato a livello di switch di aggregazione**.

- Un link viene nominato _Active_ ed il rimanente _Backup_. Su quest'ultimo il traffico viene scartato (è tuttavia possibile farci passare su il traffico L2 di controllo).
- Tutti i link che fanno parte della RTG devono essere contemporaneamente Trunk mode (e supportare le medesime VLAN) o Access mode.

Esempio di configurazione:
```
{master:0} [edit switch-options]
user@Switch-3# set redundant-trunk-group group rtg1 interface ae0.0 primary

{master:0} [edit switch-options]
user@Switch-3#set redundant-trunk-group group rtg1 interface ge-0/0/10.0

{master:0} [edit switch-options]
user@Switch-3# show
redundant-trunk-group {
	group rtg1 {
		interface ge-0/0/10.0;
		interface ae0.0 {
			primary;
		}
	}
}
```
Per effettuare il monitoraggio della RTG:
```
{master:0}
user@Switch-3> show redundant-trunk-group
```

#### GRES
 Fornisce al system control un meccanismo per switchare tra master e backup RE con impatto minimo sul traffico tramite la sincronizzazione delle _kernel table_ e delle tabelle PFE (Packet Forwarding Engine).

- RE si scambiano _keep-alive_ per stabilire se sono ancora active o se deve esserci un cambio di master.PFE si disconnette a questo punto dal vecchio RE, rimanendo operativo. Il nuovo master e PFE si sincronizzano.
```
{master: 0} [edit chassis]
user@Switch-1# set redundancy graceful-switchover

{master: 0} [edit chassis]
user@Switch-1# show
redundancy {
	graceful-switchover;
}
```
Per monitorare lo stato del GRES dal backup RE:
```
{backup:1}
user@Switch-2> show system switchover
```
#### NSR
Fornisce HA allo switch/virtual chassis switch che lo implementa per mezzo dello switch di RE ridondati (o in Virtual Chassis).  _HA perché non richiede restart dei protocolli di routing_ (sincronizza il processo relativo rpd- routing protocol process e le routing infos). Una volta abilitato e sincronizzato il commit il "_rpd_" di backup rimane allineato con tutti i messaggi inviati da e verso l'rpd master.

_Note_: va abilitato anche GRES necessariamente.
```
{master:0} [edit routing-options]
user@Switch-1# set nonstop-routing
```
#### NSB
Stessa cosa di NSR ma per il livello L2 (sincronizza RE process e le switching infos)

_Note_: va abilitato anche GRES necessariamente.
```
{master:0} [edit protocols layer2-control]
user@Switch-1# set nonstop-bridging
```
### LACP - Link Aggregation Control Protocol

Protocollo definito all'interno della specifica 802.3ad (LAG) e per mezzo del quale viene effettuato il monitoring ed il controllo dei link membri di una LAG (singolo canale logico).
1. Cancellazione ed inserimento automatico di singoli link fisici alla LAG senza intervento dell'operatore
2. Monitoraggio dei link per garantire che entrambe le estremità del "tubo" sono allineate(configurate correttamente da ambo i lati)

- Un device che opera LACP può essere configurato in modalità _active_ o _passive_:

    - **Active mode** da il via alla trasmissione di pacchetti LACP. 
    - **Passive mode** risponde ai pacchetti LACP

_Note_: Per scambiare pacchetti LACP almeno una delle due estremità deve essere configurata in active mode. By default se LACP è abilitato una interfaccia questa entra nello stato "_passive_"

- L'exchange di pacchetti LACP avviene tra **_Actors_** e **_Partners_**.

    - Actors: è l'interfaccia **locale** in un LACP exchange
    - Partner: è l'interfaccia **remota** (l'altro capo del canale logico)


Per monitorare l'utilizzo di LACP tramite i seguenti comandi:
```
{master: 0}
user@Switch-1> show interfaces extensive ae0.0 | find "LACP Statistics:"
```











