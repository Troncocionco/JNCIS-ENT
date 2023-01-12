## Ethernet Ring Protection Switching

Principalmente per topologie ad **anello** dove cioè ogni nodo è connesso ad almeno altri 2 nodi (3 nodi totali minimi).

- fornisce protezione su reti ethernet con tempi sotto i 50 ms ed assenza di loop
- può essere usato in sostituzione di STP

Il conceto base è quello di identificare un link dell'anello detto RPL "**Ring Protection Link**" su cui interrompere l'anello (loop free, porta in blocking state) in condizioni operative normali. Quando non si è sotto normali condizioni di funzionamento la porta viene messa in forwarding state dallo switch (**RPL Owner**) per ricollegare i nodi dell'anello.
Il processo viene invertito una volta rientrato il guasto.

- RPL Owner in condizioni normali invia periodicamente un R-APS "**Ring Automatic Protection Switching**" message per notificare agli altri nodi lo stato del RPL;
- Normal node, ricevono ed inoltrano R-APS. Se rilevano un fault su un local link sono loro stessi a generare R-APS per notificarlo agli altri.
#

### APS Protocol
Ogni nodo dell'anello ne deve far parte, configurando le proprie due interfacce con la specifica VLAN di servizio per il protocollo APS.
- Uso di una singola VLAN per scambiare R-APS tra nodi dell'anello


_Nota_: tutte le VLAN anche quelle di una trunk port sono affette dall'algoritmo APS (non è limitata al traffico su una singola VLAN _"protetta"_).

Struttura del Frame definita secondo specifiche ITU:
- **CFM Connectivity Fault Management**
    - **Header CFM** contiene: _"Domain Level", "Version", "Opcode", "Flags", "TLV Info"_
    - **Data** (Payload APS)

| Request/State | Reserved 1 | RPL Blocked | Do Not Flush | Status Reserved | Node ID | Reserved 2 |
| --- | --- | --- | --- | --- | --- | --- |
| 0000 / 1011 | 

**Request / State** (4 bit): 2 stati finora, nel primo (0000) il nodo vuole segnalare che non ha riscontrato nessun fault nelle sue adiacenze. 

**Reserved 1** (4 bits): sempre 0000, per futuri usi.

**RPL Blocked** (1 bit): utilizzabile sono dal RPL Owner, 0 il RPL è in stato _blocked_, 1 è in stato _unblocked_

**Do Not Flush** (1 bit): 0 indica che il device non deve azzerare la sua MAC table, 1 indicare che deve svuotarla

**status Reserved & Reserved 2**: tutti zero, per futuri sviluppi

**Node ID**: MAC address univoco rispetto all'anello

| Dest. MAC | Sou. MAC | VLAN tag | Type/Lenght | CFM Header | Data FCS |
| --- | --- | --- | --- | --- | --- |
| 01-19-A7-00-00-01 |

_Nota_: Spefico MAC address per la destinazione quando sono scambiati messaggi R-APS.

#
APS: Idle State (Normal Condition)

1. RPL Owner blocca RPL
2. RPL Owner invia ogni 5 secondi R-APS con seguenti parametri:
    - Request/State = 0000 (request) no failure occurred
    - Do not flush = 0 (flush MAC table)
    - RPL Blocked = 1 (blocked)
3. tutti gli altri switch hanno entrambe le porte in stato unblocked

APS: Link Faiulure

Alla rilevazione di un fault, gli switch coinvolti (i due adiacenti sul link incrimanato) passano nello _stato protection_:
1. Block della porta coinvolta
2. Flush della loro MAC table
3. Invio segnalazioni tempestive tramite R-APS a tutti gli altri nodi (3 msg nei primi 10 ms successivi, no hold-time), poi 1 nuovo msg ogni 5 secondi fino a che  il guasto non rientra:
    - Request / State = link failure (1011)
    - Do not flush = 0 (flush)    
4. Lo stesso fanno gli switch che ricevono tali R-APS:
    - Transizione al protection state
    - Flush della MAC Table
5. RPL Owner sblocca l'RPL e smette di inviare periodicamente R-APS fino a nuovo ordine

APS: Restoration

Una volta ristabilito il collegamento i due nodi adiacenti cominciano ad avvisare gli altri nodi del cambio di topologia:
1. Invio R-APS (**_No Request_**, **_No Flush_**)
2. Il link ristabilito viene tenuto ancora in blocking state fintanto che non viene ricevuto nuovamente un R-APS dal RPL Owner
3. Alla ricezione del R-APS di link ristabilito l'RPL Owner avvia un timer (5 min) allo scadere del quale blocca l'RPL e ricomincia ad inviare R-APS msg (No Request, RPL Blocked, Flush MAC Table)
4. Alla ricezione del R-APS dal RPL Owner tutte le porte dei restanti nodi tornano in stato forwarding idle_state

Di seguito un esempio di configurazione per il nodo RPL Owner:

        protection-group {
            ethernet-ring my-erps {
                ring-protection-link-owner;
                restore-interval XX;
                guard-interval XX;
                east-interface {
                    control-channel {
                        ge-0/0/12.0;
                    }
                    ring-protection-link-end;
                }
                west-interface {
                    control-channel {
                        ge-0/0/4.0;
                    }
                }
                control-vlan control;
                data-channel {
                    vlan data;
                }
            }
        }

        {master:0} [edit vlan]
        user@Switch-1#  show
        control {
            vlan-id 100;
        }
        data {
            vlan-id 101;
        }

        {master:0} [edit interfaces]
        user@Switch-1# show
        ge-0/0/4 {
            unit 0 {
                family ethernet-switching {
                    port-mode trunking;
                    vlan {
                        members all;
                    }
                }
            }
        }
        ge-0/0/12 {
            unit 0 {
                family ethernet-switching {
                    port-mode trunking;
                    vlan {
                        members all;
                    }
                }
            }
        }

_Note_: almeno una delle due interfacce deve essere nominata "ring-protection-link-end". "ring-protection-link-owner" indica chiaramente che lo switch è RPL owner in questo anello.

La configurazione per un nodo normale differisce solo per poche linee tra cui quelle citate poco prima.

### Monitoring APS

    user@Switch-1> show protection-group ethernet-ring aps [detail]

    user@Switch-1> show protection-group ethernet-ring interface [detail]

    user@Switch-1> show protection-group ethernet-ring node-state [detail]

    user@Switch-1> show protection-group ethernet-ring statistics [detail]