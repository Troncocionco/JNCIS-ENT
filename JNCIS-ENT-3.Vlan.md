#juniper #vlan
# VLAN

VLAN gruppo di apparati che fanno parte della stessa LAN virtuale e quindi dello stesso, logico, "broadcast-domain".

L2 ports possono essere configurate per funzionare in 2 modalità:
- access mode (switch-to-end-user-device) --> tipicamente trasportano **traffico non taggato**
- trunk mode (switch-to-switch/router) --> tipicamente trasportano **traffico taggato**, ma possono tuttavia trasportare anche traffico non taggato se configurate come  **native-vlna-id**

_By default_: tutte le porte di un L2 switch sono _enabled_ e pre-configurate per lavorare in access-mode e sono assegnate alla VLAN default (**VLAN-ID = 1**)
```
{master:0}
root> show vlans
```
**ZTP - Zero Touch Provisioning** o in alternativa specificare una differente configurazione.
```
{master:0} [edit]
root# set vlans default vlan-id 100

{master:0} [edit]
root# commit and-quit

{master:0} [edit]
root> show vlans
```
Un esempio di configurazione di switch con interfaccia assegnata alla vlan 10 (access mode):
```
{master:0} [edit]
user@Switch-1# show interfaces ge-0/0/8

unit 0 {
	family ethernet-switching {
		interface-mode access;
		vlan {
			members v10;
		}
	}
}
```
Un esempio di configurazione di switch con interfaccia assegnata alla vlan 10 e 20 (trunk-mode esplicitate tutte le vlan che sono ammesse al transito) :
```
{master:0} [edit]
user@Switch-1# show interfaces ge-0/0/12

unit 0 {
	family ethernet-switching {
		interface-mode trunk;
		vlan {
			members [ v10 v20 ];
		}
	}
}
```
## Switch Interface

- Protocol Family: ogni interfaccia può essere configurata come "**Ethernet Switching**" o come "**IPv4/IPv6 routing**"

- Port Mode: **Access Mode** o **Trunk Mode** (by default abilitato su tutte le porte come Access Mode)

- Power Over Ethernet PoE: by default è abilitato su tutte le porte

- Physical Link Settings: by default **auto-negotiation** per **speed** e **duplex mode** (altri parametri disponibili _flow control_ e _MTU_)

## Voice VLAN

Feature che permette ad una porta di uno switch di accettare sia traffico non taggato che traffico taggato. 

Questi due flussi saranno marchiati dallo switch con due differenti VLAN permettendo una differente CoS tra i due flussi.

_Genericamente viene implementato per distinguere tra traffico dati e traffico voce_. Di seguito un esempio di configurazione:
```
[edit switch-options]
user@Switch-1# show
voip {
	interface (access-ports | interface-name) {
		vlan (vlan-name | vid);
		forwarding-class class;
	}
}
```
-  "access-port": associa i parametri VoIP a tutte le access ports
- "interface-name": associa i parametri VoIP ad una specifica access port
- vid | class: referenced VLAN and forwarding class mut be defined locally in switch

### Native Vlan ID for Trunking Mode ports
```
{master:0} [edit interfaces]
user@Switch-1# show ge-0/0/12

native-vlan-id 1;
unit 0 {
	family ethernet-switching {
		interface-mode trunking;
		vlan {
			members [ 1 v14 v15 ];
		}
	}
}
```

### IRB - Integrated Routing and Bridging

E' un tipo di interfaccia  **logica** di livello 3 che viene definita sugli switch come quelli della serie EX e facilita il processo di **inter-VLAN routing**

In questo modo gli end-user device connessi ad uno stesso switch ma in VLAN differenti possono utilizzare tale IRB interface come **default gateway**.

Tipicamente questa funzionalità è implementata a livello di Access / Aggregation switches.

Esempio di configurazione interfaccia IRB di uno switch con 3 VLAN che insistono su 3 subnet:
```
{master: 0} [edit]
user@Switch-1# show interfaces irb

unit 14 {
	family inet {
		address 172.23.14.1/24
	}
}
unit 15 {
	family inet {
		address 172.23.15.1/24
	}
}
unit 16 {
	family inet {
		address 172.23.16.1/24
	}
}
```
_Nota_: "family inet" corrisponde alla family protocol IPv4

Chiaramente ogni VLAN deve avere almeno una interfaccia fisica attiva per poter funzionale e nella configurazione delle singole VLAN va **esplicitata l'associazione tra la VLAN e l'interfaccia IRB**.

```
{master:0} [edit]
user@Switch-1# show vlans

default {
	vlan-id 1;
}
v14 {
	vlan-id 14;
	l3-interface irb.14;
}
v15 {
	vlan-id 15;
	l3-interface irb.15;
}
v16 {
	vlan-id 16;
	l3-interface irb.16;
}

{master:0} [edit]
user@Switch-1> show vlans
Routing instance        VLAN name       Tag     Interfaces
default-switch          default          1      ge-0/0/0.0*
												...
default-switch            v14            14     
												ge-0/0/6.0*
												ge-0/0/7.0*
default-switch            v15            15      
												ge-0/0/8.0*
												ge-0/0/9.0*
default-switch            v16            16      
												ge-0/0/10.0*
												ge-0/0/11.0*
```

## Lab - Section
```
{master:0}
lab@ex2> configure
Entering configuration mode

{master:0} [edit]
lab@ex2# load override jex/lab2-start.config 
load complete

{master:0} [edit]
lab@ex2# commit
configuration check succeds
commit complete

{master:0} [edit]
lab@ex2# run show vlans
Routing instance        VLAN name       Tag     Interfaces
default-switch          default          1      ge-0/0/1.0*
												ge-0/0/2.0*
												ge-0/0/3.0*

{master:0} [edit]
lab@ex2# copy interfaces ge-0/0/1 to ge-0/0/0

{master:0} [edit]
lab@ex2# show interfaces
ge-0/0/0 {
	unit 0 {
		family ethernet-switching {
			interface-mode access;
			vlan {
				members default;
			}
		}
	}
}
ge-0/0/1 {
	unit 0 {
		family ethernet-switching {
			interface-mode access;
			vlan {
				members default;
			}
		}
	}
}
ge-0/0/2 {
	unit 0 {
		family ethernet-switching {
			interface-mode access;
			vlan {
				members default;
			}
		}
	}
}
ge-0/0/3 {
	unit 0 {
		family ethernet-switching {
			interface-mode access;
			vlan {
				members default;
			}
		}
	}
}
me0 {
	unit 0 {
		family inet {
			address 10.210.34.132/26
		}
	}
}

{master:0} [edit]
lab@ex2# commit
```
Crea le VLAN 11 e 12
```
{master:0} [edit]
lab@ex2# edit vlans

{master:0} [edit vlans]
lab@ex2# set v11 vlan-id 11

{master:0} [edit vlans]
lab@ex2# set v12 vlan-id 12

{master:0} [edit vlans]
lab@ex2# show
default {
	vlan-id 1;
}
v11 {
	vlan-id 11;
}
v12 {
	vlan-id 12;
}
```
Assegnamo le interfacce ge-0/0/2 e 3 rispettivamente alle vlan 11 e 12
```
{master:0} [edit vlans]
lab@ex2# top edit interfaces

{master:0} [edit interfaces]
lab@ex2# set ge-0/0/2 unit 0 family ethernet-switching vlan members v11

{master:0} [edit interfaces]
lab@ex2# set ge-0/0/3 unit 0 family ethernet-switching vlan members v12
```
Rimuoviamo le interfacce qui sopra dalla default vlan
```
{master:0} [edit interfaces]
lab@ex2# delete ge-0/0/2 unit 0 family ethernet-switching vlan members default

{master:0} [edit interfaces]
lab@ex2# set ge-0/0/3 unit 0 family ethernet-switching vlan members default
 ```   
Rimuoviamo la vlan di default dalla ge-0/0/1 e attiviamola in modalità trunking con le vlan assegnate 11 e 12
```
{master:0} [edit interfaces]
lab@ex2# delete ge-0/0/1 unit 0 family ethernet-switching vlan members default

{master:0} [edit interfaces]
lab@ex2# set ge-0/0/1 unit 0 family ethernet-switching interface-mode trunk

{master:0} [edit interfaces]
lab@ex2# set ge-0/0/2 unit 0 family ethernet-switching vlan members [ v11 v12 ]
```
