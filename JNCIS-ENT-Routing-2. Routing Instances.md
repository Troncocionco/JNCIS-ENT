#routing #juniper 
# Routing Instances
Una Routing in buona sostanza è una collezione di:
- Tabelle di routing
- Interfacce
- Routing Protocol Parameters

Questo fa si che un singolo device possa emulare il comportamento di più device.
By default Junos crea una Routing Instance predefinita di nome "master", all'interno del quale troviamo:
- inet.0 tabella di routing per l'indirizzamento di pacchetti IPv4
- inet6.0 tabella di routing per l'indirizzamento di pacchetti IPv6
- ge-0/0/0.0
- ge-0/0/1.0
- lo0.0
- Default Route
- OSPF

### How to configure & share routes between Routing Instances

_Nota_: la configurazione di Routing Instance aggiuntive può essere fatta dall'amministratore di rete a partire dal livello gerarchico `[edit routing-instances]`

Tipicamente la configurazione di Routing Instances permette l'implementazione di alcuni scenari di utilizzo come:
- FBF Filter-based-Forwarding (o policy-based-forwarding)
- VPN Virtual Private Network
- System Virtualization

Le Routing Instance hanno un **tipo** assegnato al momento della creazione:
- forwarding (FBF)
- l2backhaul-vpn
- l2-vpn
- layer2-control
- mpls-internet-multicast
- no-forwarding
- virtual-router
- virtual-switch
- vpls
- vrf

_Nota_: by default tutti i comandi generalmente sono associati alla master routing instance in quanto quella di default.

`show interface terse routing-instances <routing-instance-name>`

Per esempio è la versione del comando "show interface terse" per ricercare in una routing instance che non sia quella di default. Di seguito l'esempio anche per mostrare a video la tabella di routing "inet.0":

`show route table <user-instance-name-inet.0>`

Allo stesso modo anche altre operazioni, come il test di raggiungibilità tramite _ping_, dove non diversamente specificato utilizzano l'istanza di routing di default.

Per poter installare la stessa rotta simultaneamente su una o più tabelle di routing, Junos introduce il concetto di **RIB Group** (gruppi di tabelle di routing).

1. Creare le diverse tabelle di Routing a livello gerarchico `[edit routing-options]`
2. Esportare porzioni di tali tabelle tramite comandi di  "import/export" nella configurazione di altre tabelle

I comandi menzionati al punto due sono:
- instance import
- instance export
- auto-export

Nello specifico tali comandi si basano su ciò che viene configurato 
```
    [edit routing-options]
    user@R1# show
    rib-groups {
        <rib-group-name> {
            export-rib <routing-table-name>;
            import-rib <routing-table-names>;
            import-policy <policy-name>;
        }
    }
```

`import-rib` - può indicare più di una tabella di routing e sostanzialmente informa Junos su dove installare le rotte apprese tramite protocolli di routing.

**Nota**: la prima delle tabelle di routing indicate in questa lista è considerata la **primary routing table** ossia la tabella dove vengono importate le rotte apprese anche senza la dichiarazione del rib-group. Infatti la rib-group configuration è spesso omessa nel template.

`export-rib` - può indicare come parametro un'unica tabella di routing e sostanzialmente indica da dove estrarre le informazioni di routing


Una volta creato un rib-group può essere "applicato" in più parti del template di configurazione, come ad esempio:
- Interface routes
- Static routes
- OSPF
- IS-IS
- RIP
- BGP
- Protocol Independent Multicast (PIM)
- Multicast Source Discovery Protocol (MSDP)

Il comando da utilizzare ai vari livelli gerarchici è un semplice "set". Di seguito un esempio di applicazione su OSPF:
```
[edit protocols ospf]
user@R1# set rib-group <rib-group-name>

[edit routing-options]
user@R1# show
rib-groups {
	test {
		import-rib [ inet.0 test.inet.0 ];
	}
}

[edit protocols ospf]
user@R1# show
rib-group test;
	area 0.0.0.0 {
		interface ge-0/0/1.0;
		interface lo0.0;
	}
```
Per poter effettuare una connessione tra le varie _routing instances_ è possibile definire quello che si chiama un **_logica tunnel lt_**. Su ciascuna delle due instanze coinvolte nel logical tunnel va configurata una lt-interface (tipicamente del tipo **lt-fpc/pic/port**), la quale può appartenere ad un unico lt alla volta. Una volta configurate va instaurata/configurata tra queste due una **relazione di peering** (parametro "peer-unit" indica l'altra interfaccia logica facente parte del tunnel).
```
[edit interfaces lt-0/0/0]
user@R1# show
	unit 0 {
		encapsulation ethernet;
			peer-unit 1;
			family inet;
	}
	unit 1 {
		encapsulation ethernet;
			peer-unit 0;
			family inet;
	}
```

Per ogni interfaccia è possibile scegliere il tipo di **encapsulation** tra i seguenti tipi:
- Ethernet
- Ethernet circuit cross-connect
- Ethernet virtual private LAN service (VPLS)
- Frame Relay
- Frame Relay CCC
- VLAN
- VLAN CCC
- VLAN VPLS

Allo stesso modo di un interfaccia classica questa può far parte delle famiglie **IP** (inet), **IPv6** (inet6), **ISO** o **MPLS**.

Ovviamente per questi due parametri la configurazione _da ambo i lati del tunnel deve essere omogenea_.