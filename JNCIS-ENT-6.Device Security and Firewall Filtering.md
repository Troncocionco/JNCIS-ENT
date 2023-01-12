#juniper #firewall #acl
# Traffic Storm 
Sulla serie EX switch di Juniper esiste una modalità *anti-storm* chiamata "**Storm Control**".

_Nota_: per traffic storm si intende una situazione in cui il flusso di traffico broadcast, multicast o anche unicast da sorgenti ignote, esaurisce le risorse di rete comportando rallentamenti o addirittura interruzioni del servizio per gli utenti finali.

Tra gli indici generali di una traffic-storm abbiamo il timeout dei timer relativi ai protocolli utilizzati o quando i tempi di risposta della rete sono anomali(rallentati)

La Storm Controlo è una funzionalità che agisce in base al superamento delle soglie(**Storm Control Level**), andando a droppare il flusso di traffico che ha triggerato questo superamento.

By default l'abilitazione di questa funzione setta lo Storm Control Level all'80% del traffico (quale che sia il tipo di traffico Broadcast, Multicast,..) su ogni singola interfaccia.

- La configurazione può essere cambiata al livello _edit forwarding-options_.

Esempio della configurazione base:

    [edit]
    user@Switch-1# show interfaces
    ge-0/0/0 {
        family ethernet-switching {
            vlan {
                member default;
            }
            storm-control default;
        }
    }

    [edit]
    user@Switch-1# show forwarding-options
    storm-control-profiles default {
        all;
    }

La configurazione del limiti di banda e del tipo di traffico può essere fatta per ciascuna delle interfaccie dell'apparato.

Quando il livello di Storm Control Level viene superato su una interfaccia questa scarta tutto il traffico targettizzato in eccesso.
Tale azione può essere anche configurata in modo tale da portare in shutdown l'interfaccia direttamente. --> L'interfaccia deve essere riabilitata manualmente o va configurato un timer di recovery (tra 10 e 3600 secondi).

    [edit forwarding-options]
    user@Switch-1# show
    storm-control-profiles default {
        all;
        action-shutdown;
    }

Per rimuovere il trigger dello Storm Control:

    user@Switch-1> clear ethernet-switching recovery-timeout


## Firewall Filters (ACL - Access Control List)

- Stateless
- In HW technology
- Support a 3 tipi di Firewall Filters (ingress/egress):
    - Port-based L2 interfaces
    - VLAN-based 
    - Router-based L3 interfaces
- Una sola ACL per interfaccia per direzione


Le regole di FW Filtering sono applicate nel seguente ordine per il traffico Ingress (per quello Egress esattamente al contrario):
1. Port -based
2. Vlan-based
3. Routing-based

Una regola di FW filtering è composta da **termini**: ciascun termine valutato sequenzialmente uno dopo l'altro ed è costituito da una _matching condition_ **from** e da una azione conseguente **then**.

Nota: tutto il traffico che non viene **esplicitamente** permesso (allow) e non matcha nessuna delle regole di FW viene silenziosament scartato a causa del termine di default di ogni regola.

Common Matching criteria:
- Numeric range
- Address
- Bit field

FW Filtering **Actions**:
- **Terminating Action**
    - accept
    - discard (silently, no ICMP response back)
    - reject
- **Action Modifiers** (implicitamente se ne specifichiamo una ma non una Terminating Action --> Il device impone una Terminating Action con cui accetta il pacchetto)
    - analyzer, count, log and syslog
    - forwarding-class and loss-priority
    - policer
    