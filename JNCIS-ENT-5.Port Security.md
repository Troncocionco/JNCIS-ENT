#juniper #security #macsec
# Port Security

## MAC Limiting
## Persistent MAC
MAC address appresi dal device e che permangono (persistenti) in memoria anche se lo switch viene rebootato o se l'interfaccia da cui è stato appreso va giù.
**Condizioni:**
- Interfaccia deve essere configurata in _access mode_
- Interfaccia non deve nemmeno essere parte di un trunk redundant group
- Sull'interfaccia non devono essere abilitati meccanismi di autenticazioni 802.1x
- Sull'interfaccia deve essere abilitato l'apprendimento di un nuovi MAC address

**Nota**: la funzione è disabilitata by default

    {master: 0}[edit switch-options]
    user@Switch-1# show
    interface g-0/0/6.0 {
        persistent-learning;
    }

Per mostrare quali MAC sono "persistenti" è possiibile utilizzare questo comando. Tali indirizzi saranno indicati con il **flag "P"**.

    user@Switch-1> show ethernet-switching table

### DHCP Vulnerabilities
Quando un client invia la richiesta per una configurazione ad un server DHCP che non conosce, invia un pacchetto broadcast, che può quindi essere ricevuto da chiunque nella subnet e _a cui chiunque può rispondere_.

Due meccanismi possiamo mettere in piedi per limitare i danni (almeno sullo switch):

**DHCP Snooping**
- DHCP binding database: una volta ispezionati i pacchetti DHCP memorizza in un db l'associazione MAC address, IP configurato tramite DHCP, VLAN etc.

- DHCP packet inspection su porte non trusted
    - By default untrusted sono tutte le porte access
    - Questa operazione ovviamente non accade su interfacce con IP Static configurato  

**Nota**: il file DHCP Snooping può essere memorizzato in modo persistente su file

Per verificare lo stato delle associazioni attualmente registrate tramite tale funzionalità.

    user@Switch-1> show dhcp-security binding

### ARP Vulnerabilities 
Anti ARP Spoofing - un attaccante forgia il proprio MAC address in modo tale che questo corrisponda con il MAC address del target. ARP msg sono inoltrati in modalità broadcast su tutte le porte che partecipano alla stessa VLAN.

**Dynamic ARP Inspection** - Affidandosi al contenuto del DHCP Snooping Database (per cui tale funzionalità deve essere abilitata) per ispezionare tutti i pacchetti ARP ricevuti su porte untrusted (by default quelle in ACCESS mode).
- Se non esiste una associazione IP-MAC nelle tabella DAI droppa il pacchetto
- DAI droppa anche i pacchetti con IP invalido

By default è disabilitata come funzionalità. Viene abilitata su singola VLAN e non su ciascuna porta

Esempio di basic configuration di DAI:

    [edit vlans]
    user@Switch-1# show
    default {
        vlan-id 1;
        forwarding-options {
            dhcp-security {
                arp-inspection;
            }
        }
    }

### IP Source Guard
In sostanza per proteggersi sempre da IP Spoofing o ARP/TCP flooding. E' un meccanismo per permette l'ispezione dei pacchetti provenienti da porte non trusted (cioè in ACCESS mode) qui viene comparato l'IP sorgente del pacchetto e si cerca un riscontro sempre nel db formato dal DHCP Snooping. Se non esiste traccia di tale sorgente il pacchetto viene scartato direttamente.

**MACsec** - Il trasporto tra due switch è in molti casi in clear text esponendo tale traffico ad una moltitudine di attacchi. MACsec (**802.1AE**) fornisce comunicazione AUTENTICATA e CIFRATA fra due nodi direttamente connessi (P2P point-to-point).

**Nota**: sia sulla serie EX che su QFX è necessaria una licenza aggiuntiva per abilitare tale funzionalità.

- Su base pre-shared key
- Uno dei due nodi assume il ruolo di Key-server e aggiorna la chiave periodicamente fintanto che MACsec è abilitato 

Esempio di configuazione CAK Static:
- Create connectivity association

        [edit security macsec]
        user@Switch-1# set security-association ca-name

- Configure the MACsec mode as Static CAK

        [edit security macsec]
        user@Switch-1# set connectivity-association ca-name security-mode static-cak

- Configure the pre-shared key with CKN e CAK

        [edit security macsec]
        user@Switch-1# set connectivity-association ca-name pre-shared-key ckn hexdecimanl-number-ckn

        [edit security macsec]
        user@Switch-1# set connectivity-association ca-name cak hexdecimanl-number-cak

- Associate interfaces with the connectivity association

        [edit security macsec]
        user@Switch-1# set interface interface_name connectivity-association ca-name