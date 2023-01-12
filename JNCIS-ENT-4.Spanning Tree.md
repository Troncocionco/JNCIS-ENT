#juniper #stp #rstp
# STP - Spanning Tree Protocol

STP mira a bloccare il forwarding incontrollato, troncando i rami che genererebbero dei loop.

Dal nodo root vengono calcolati i best path per raggiungere ogni nodo dell'albero. 
I link _non necessari_ a questi best-path vengono **esclusi** da STP finche' questa topologia non cambia stato (un link cade o similare).

Come scegliere il root_node? Ogni nodo di rete e' associato con un **BRIDGE_ID**=(**priorita'**, **MAC_add**). 

Default: Priority_value e' uguale per tutti ma e' chiaramente modificabile. Valori piu' bassi vengono prima.

**BPDU** - Bridge Protocol Data Unit e' utilizzato per la negoziazione e l'elezione del nodo_root.


La sezione serve per il calcolo dello spanning tree:
- Root ID
- RPC - Root Path Cost
- Bridge ID - Chi sta mandando il BPDU in questione
- Port ID - Quale porta mandando fuori questo BPDU --> Come il Bridge_ID e' dato dalla coppia (Priorita', Port Number)

## Procedura - STP

1) Scambio di BPDU msg tra i nodi per l'**elezione del Root Node**. I nodi si mettono in ascolto e scambiano BPDU, partendo dall'assunzione che ciascuno si considera inizialmente come root. I messaggi che vengono scambiati in questa fase sono detti 'Hello BPDU' e come detto il campo Root_ID e' riempito con il Bridge ID che lo switch mittente crede essere il root in quel momento (inizialmente se stesso). Tramite il confronto del campo Bridge ID, si arriva ad individuare il nodo root effettivo. In sostanza: quando uno switch identifica un migliore Bridge_ID smette di annunciarsi come Root e si limita a forwardare gli Hello BPDU che gli arrivano.

2) **Determinazione del Port Role e State**: Una volta stabilito il nodo root inizia il calcolo del best-path da ciascuno switch verso il nodo root, questo al fine di identificare la porta su ciascuno switch che lo connette, secondo il best-path, al nodo root. Durante questa fase vengono associati dei ruoli alle porte degli switch. Il calcolo dei path-cost avviene in modo similare all'annunciazione dei pacchetti Hello. Partendo dal nodo root, ogni switch riceve i pacchetti Hello provenienti dal Root e li inoltra verso le altre interfacce. Ogni switch lungo l'albero ricevendo questi pacchetti aggiungerà il costo associato all'interfaccia ricevente e memorizzando il costo parziale. Questa operazione viene effettuata su tutte le porte: a fine giro quella che presenta il Best Root Path, ossia che ha il Port Cost minore associato, viene eletta Root Port sullo switch.

3)  **Determinazione del Designated Switch**: tra gli switch non root viene designato quello che offre Root Path migliore di tutti (spareggio su Bridge ID, Priority). Una volta designato lo switch designato a rappresentare il segmento LAN (ossia i restanti nodi non root) viene identificata la porta che da questo switch si affaccia su tale segmento detta "designated port" (spareggio con Port ID)


| DA | SA | L | LLC | BPDU | FCS |
| --- | --- | --- | --- | --- | --- |
| Dest. MAC | Sour. MAC | Length | Logical Link Control Header | BPDU Configiration/TCN | Checksum |
| Special Multicast Reserved | Port Originated | - | 0x42 | - |- |


### Formula x Convergence Delay 

    Convergence_Delay = 2 x Forwarding_Delay + Max_Age

## Node Failure

1. TCN ACK verso il root node
2. Il root node risponde con al TCN ACK
3. Root node inviata tramite BPDU Configuration la topologia aggiornata (flag topology settato)
4. Switch non root cambiano Aging Timer della _forwarding table_ a 15s forzandoli a cancellare le loro entry correnti che non sono riconfermate entro i prossimi 15s.

_Nota_: by default gli switch Juniper, se abilitati STP, utilizzano RSTP force 0, retrocompatibile con STP.

## RSTP

Introduce importanti migliorie per ridurre considerevolmente il tempo di convergenza di STP:

- P2P Link Designation (ossia le Ethernet port in modalita' _full duplex_)
- Edge Port designation (ports che connettono segmenti di LAN che non hanno altri sbocchi: Switch - End Device) --> Sempre in Forwarding State
- New Root or Designated Port possono transitare nello stato forwarding senza aspettare lo scadere di timer (se sono porte Ethernet Full Duplex)
- Introduce Alternate/Backup Port Role
    - ***Alternate*** = Root port di backup (discarding traffico sino a che non riceve un Superior BPDU da un vicino)
    - ***Backup*** = backup su Designated Switch alle D port (in stato discarding finche' D port funzionano correttamente sullo switch)
- Inferiore numero di stati (Discarding, Learning, Forwarding )

Discarding: porte non amministrativamente attive nella topologia. Ricevono comunque BDPU.

Learning: porte non attive nell'inoltro del traffico ma attive dal punto di vista della forwarding table e apprendimento dei MAC address.

BDPU Configuration sono ora utilizzati anche in funzione di Keep Alive. Ogni Designated Port ne invia uno ogni 2 secondi. 

**Se uno switch non riceve un BPDU dal vicino entro 3 x Keep Alive = 6 sec deduce la presenza di un fault sulla rete e avvia il processo di aggiornamento della topologia.**

RSTP introduce inoltre diversi nuovi flags:

- TCN Acknowledgment - per ack STP TCNs
- Agreement & Proposal - per una veloce transizione di una designated port allo stato forwarding
- Forwarding & Learning - per advertise lo stato della porta sending
- Port Role - specifica il 'role' della porta sending:
    - 0: Unknown
    - 1: Alternate // Backup
    - 2: Root
    - 3: Designated
- Topology Change 

### Transizioni

RSTP permette una semplificazione del processo di transizione allo stato 'forwarding', in quanto non e' piu' necessario che la porta transiti per gli stati listening e learning (per cui e' necessario attendere fino a 30s) ma se la porta lavora in full-duplex questo puo' essere fatto tramite proposal & agreement handshake.

- Edge port, sono porte che non ricevono BPDU e assumono il ruole Edge port automaticamente
- Se una Edge port riceve un BPDU di configurazione questa transita ad uno stato di normal spanning-tree port

RSTP migliora la stabilita' della rete **generando TCN solo in corrispondenza di un cambio di stato di una porta Edge**.
1. In questo caso il device initiator flood su tutte le designated port e root port il TCN BDPU
2. I neighboors sono quindi avvisati molto prima rispetto ad STP (dove era necessario che il device fosse nel path verso il root node) e possono iniziare a cancellare entry dalla MAC table senza attendere ulteriori istruzioni
3. Non vengono cancellati MAC address appresi dalle Edge port e appresi dalla porta riceventi il TCN


## Configuring RSTP
Un esempio di configurazione base per RSTP.
**Nota**: by default RSTP è attivo in versione 0, ossia fully retro compatibile con STP (per questo sono presenti molti parametri in uso a STP):

    [edit protocols rstp]
    user@switch# show

    bridge-priority 32k;
    max-age 20;
    hello-time 2;
    forward-delay 15;
    interface ge-0/0/10.0 {
        disable;
    }
    interface ge-0/0/13.0 {
        cost 20000;
        mode point-to-point;
    }
    interface ge-0/0/14.0 {
        priority 128;
        mode shared;
    }
    interface ge-0/0/2.0 {
        edge;
    }

    user@switch> show spanning-tree interface

    Spanning tree interface parameters for instance 0

| Interface |  Port ID | Designated port ID | Designated bridge ID | Port Cost | State | Role |
| ----- | ------ | ------ | ------ | ------ | ------ | ------ |


    user@switch> show spanning-tree statistics interface

| Interface | BPDUSs sent | BPDUs received | Next BPDU transmission |
| ----- | ----- | ----- | ----- |

    user@switch> show spanning-tree bridge

    STP bridge parameters                       
    Routing instance name                       :   GLOBAL
    Context ID                                  :   0
    Enabled protocol                            :   RSTP
        Root ID                                 :   4096.C0:42:D0:09:B6:E0
        Root Cost                               :   20000
        Root port                               :   ge-0/0/1
        Hello time                              :   2 seconds
        Maximum age                             :   20 seconds
        Forward delay                           :   15 seconds
        Message age                             :   1
        Number of topology changes              :   1   
        Time since last topology change         :   26 seconds         
        Local parameters                        
            Bridge ID                           :   32768.c0:42:d0:09:aa:e0
            Extended system ID                  :   0