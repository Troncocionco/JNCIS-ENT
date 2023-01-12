#juniper #virtualchassis 
# Virtual Chassis

Interconnessione tra uno o più switch (tipicamente dello stesso modello) al fine di creare e gestire un unico device virtual, detto appunto "**virtual chassis**".
Es. EX4300 possono essere collegati in cascata utilizzato le porte di uplink fino ad un massimo stack di 10 unit.

Due sono le principali ragioni che ci spingono ad adottare questa tecnologia:

- HA - nel caso di due switch, viene nominato uno switch **master** ed uno di **backup**. Questo permette le funzionalità di NSR (non stop routing) e NSB (non stop bridging).
Ossia i controllo lato Routing Engine può passare, secondo necessità, da una macchina all'altra senza interruzione dei servizi e delle funzionalità/protocolli L2 e L3.

- Simplified Design - concettualmente un virtual Chassis è un device singolo e lo è anche sotto altri aspetti come quello del software. Nel fare l'upgrade di queste macchine, va considerato l'upgrade solo ed esclusivamente del nodo master e non di tutti. Inoltre non è necessario implementare STP in quanto si tratta di una singola macchina.

Serve:
- Switch famiglia EX o QFX
- **VCP** -  Virtual Chassis Ports (sono generalmente porte dedicate o **porte di uplink convertite** come _extended VCP_)

_Nota_: chiaramente nel secondo caso la configurazione è richiesta per convertire le porte di uplink in VCP. Di seguito un esempio della configurazione necessaria.

    {master: 0}
    user@Switch-1# request virtual-chassis vc-port set pic-slot 1 port 0

Comando per controllare lo status delle porte VCP:

    {master: 0}
    user@Switch-1> show virtual-chassis vc-port

Gli switch partecipanti un virtual chassis possono svolgere il ruolo di **Routing Engine** o quello di **Line Cards**.

Esistono diversi standard di interconnessione:

1.
2.
3.

Ogni Switch all'interno del virtual chassis posseggono una numerazione da 0 a 9. Lo 0 è riservato per il master RE. Questi **Master ID** possono essere assegnati dinamicamente dal master node o possono essere assegnati manualmente dall'amministratore di rete.

Esempio per modificare la numerazione una volta assegnata per via manuale:

    {master: 0}
    user@Switch-1> request virtual-chassis renumber member-id 0 new-member 5

    {master:5}
    user@Switch-1>

Per richiedere lo shutdown di uno specifico membro del virtual chassis:

    {master: 5}
    user@Switch-1> request system hault member ?

Per richiedere l'accesso alla sessione specifica di un membro del virtual chassis:

    {master: 5}
    user@Switch-1> request session member 1

La porta di management (_me - management ethernet_) di ogni singolo switch partecipante il virtual chassis è assegnata alla VLAN speciale di management del virtual chassis. A questa VLAN corrisponde una L3 interface detta VME - **Virtual Management Ethernet** a cui chiaramente viene assegnato un **IP unico**.
Questo permette di fatto la gestione del Virtual Chassis anche quando la ME del Master Node non è disponibile.

Anche le porte Console funzionano a tutti gli effetti come un'unica virtual console port, nel senso che reindirizzano tutte alla console session del master node. Ogni info sui nodi partecipanti è ottenibile infatti dal master. Tuttavia è possibile stabilire sessioni singole verso i singoli participanti come illustrato nel comando di esempio più su.

Al fine di partecipare nel Virtual Chassis ogni partecipante deve eseguire la **medesima versione di Junos OS**. In questo senso quando un nuovo device viene aggiunto allo stack, il master node controlla la sua versione e se diversa (e se la funzione di **SW Auto Upgrade** è disabilitata) lo setta in stato _Inactive_ fino a che la  situazione non viene manualmente sanata dall'amministratore di rete.

A partire dal masterr node usare _request system softwar add_ per upgradare tutti gli switch membri del virtual chassis.

Alternativamente se si vuole aggiornare la versione di un solo dei partecipanti usare l'opzione _member_ dello stesso comando.

    {master: 0}
    user@Switch-1> request system software add member ?

Esempio della configurazione che ci serve per avere la funzionalità di autoupgrade:

    [edit virtual-chassis]
    user@Switch-1# show
    auto-sw-update {
    package-name /path/to/junos-img.tgz;
    }

Avendo un Virtual Chassis è possibile sostenere l'upgrade SW di tutti i partecipanti con una minima interruzione del servizio, grazie alla funzionalità di NSSU - **Non-Stop Software Upgrade**.
Dopo aver scaricato la nuova versione ed averla caricata sul Master node, è possibile iniziare la procedura loggandosi appunto sul master node e lanciando il seguente comando:

    {master: 0}
    user@Switch-1# request system software nonstop-upgrade /var/temp/packageXXX.tgz


1. I partecipanti devono essere connessi in una topologia logica ad anello
2. Master e Backup node devono essere adiacenti
3. Line Card devono essere preprovisioned con il relativo _Line Card Role_ in modo da rimanere tali anche dopo il reboot
4. Tutti i membri devono avere in esecuzione la stessa versione sw
5. NSR - nonstop Routing e GRES - graceful Routing Engine Switchover devono essere abilitati per eliminare gli impatti sul traffico dovuti allo switchover tra master e backup
6. NSB per minimizzare gli impatti sui protocolli L2
7. Richiedere un backup verso una fonte esterna del System Configuration e della Startup Configurazione mediante il comando:
_**request system snapshot**_

Esempio di preprovisioned configuration role:

    {master: 0}
    user@Switch-1# show
    preprovisioned;
    member 0 {
        role routing-engine;
        ...
    }
    member 1 {
        role line-card;
        ...
    }
    member 2 {
        role routing-engine;
        ...
    }
    ...


Virtual Chassic Control Protocol VCCP al fine di creare una topologia priva di loop. I membri si scambiano LSA - Link State Advertisement tra i vari PFE al fine di costruire la topology map dei PFE.

Trami SPF ogni switch determina il miglior path per creare una mappa tra tutti i PFE (ossia quale sia il miglior path da ciascun PFE per raggiungere ogni altro PFE).

Lo **shortest** path per ogni pacchetto con flow intra-chassis è determinato su base **#hop** e **bandwith**.

E' importante settare la più alta priorità al parametro di Mastership (per il nodo master e backup), in quanto l'elezione di un nuovo nodo master passa per i seguenti step:
1. Più alta priority (1-255)
2. Se in uno stato precedente era un nodo master prima del reboot
3. Membro che è in funzione da più tempo
4. MAC più basso

Il secondo nodo che giunge in questa selezione è assurto a nuovo nodo di backup.

    {master: 0}
    user@Switch-1> show configuration virtual-chassis

    {master: 0}
    user@Switch-1> show virtual-chassis status