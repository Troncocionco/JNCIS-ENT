#routing #juniper
# Protocol Independent Routing

Sono rotte statiche configurate manualmente dagli amministratori di rete.
Nelle **routing table** (anche dette _RIB_ - Routing Information Base) sono riportate con la dicitura "_Static_" ed un costo/preference che di default è configurato a **5**.

- Tutte le configurazioni di queste rotte avvengono al livello gerarchico `[edit routing-options]`.

- La definizione di una rotta statica consta di un **_prefix_** e di un **_next-hop_**.
  - Per next-hop intendiamo l'indirizzo IP "**direttamente connesso**" del device attraverso cui è raggiungibile il prefisso indicato
  - Per link P2P è possibile specificare l'identificativo dell'interfaccia _egress_ invece che l'IP del device directly connected, **purchè tale interfaccia egress non sia di tipo _Ethernet_**.
  - Il next-hop può anche essere un'azione (**reject / discard**) nel caso di configurazioni **bit-bucket**, ossia in cui il pacchetto è droppato dalla rete.
  

_Nota_: by default Junos non ha abilitato l'auto-resolve nel caso il next-hop non sia directly connected. Quindi by default il next-hop deve essere raggiungibile tramite link diretto perché la rotta statica possa essere _active_.

-   Una rotta statica rimane attiva finché l'amministratore non la rimuove o finchè non diventa _inactive_.
-   Al livello gerarchico menzionato, sotto il campo "_default_" è possibile specificare options che verranno applicate by default su tutte le rotte statiche per cui tali parametri non sono stati esplicitamente sovrascritti.

## Static Route Options

- `no-readvertise` per proibire la redistribuzione della rotta associata attraverso i protocolli dinamici tipo OSPF
  - Fortemente consigliato per rotte statiche che coinvolgono l'instradamento del traffico di management (o proveniente dalla interfaccia Ethernet di Management) 
- `as-path`(BGP) se la rotta deve essere _redistribuita_ tramite BGP e se vogliamo modificare manualmente il valore di AS path attribute
- `community` (BGP) se la rotta è per BGP e si vogliono aggiungere valori _community_ per la rotta per il proprio AS 
- `metric` per specificare la metrica da utilizzare, utile per scegliere tra rotte che hanno magari lo stesso valore di "preference"
- `preference` 


## Floating Static Route
E' possibile specificare per una rotta statica un **_Qualified-next-hop_**, ossia un next-hop secondario che subentra al next-hop primario in funzione delle preferenze ad esse associate. Nel caso riportato, avendo una _preference_ più alta il qualified-next-hop sarà utilizzato per instradare il traffico solo qualora il next-hop principale non sia disponibile.
```
[edit routing-options]
user@R1# show
static {
	route 0.0.0.0/0 {
		next-hop 172.30.25.1;
		qualified-next-hop 172.30.25.5 {
			preference 7;
		}
	}
}
```
_Nota_: e' possibile specificare a questo livello gerarchico anche l'opzione "_next-table_" per cui il traffico che match particolari criteri specificati dall'amministratore di rete viene inoltrato per una **seconda lookup** alla tabella di routing indicata.

## Aggregated Routes
Rappresenta un vantaggio per la riduzione del traffico di controllo da parte del router. Se R1 deve esportare connessioni interne a diversi prefix del tipo (172.29.0.0/24, 172.29.1.0/24, 172.29.2.0/24) può invece esportare direttamente un prefix più "ampio" per rappresentare con una rotta singola tutte le sue rotte interne (172.29.0.0/22)

**Configurazione**
A livello gerarchico le _aggregated routes_ sono configurate sempre al livello `[edit routing-options]` e sono costituite principlamente dal _summary destination prefix_.
- Una aggregated route è attiva solo nel momento almeno una delle _contributing route_ diviene attiva
- By default la preferenza configurata per questa tipologia di rotte è pari a **130**
- Come next-hop è specificato di default "reject". Questo fa sì che ogni pacchetto che matcha il summary prefix ma per cui non esiste una rotta con longest prefix match tra le **contributing-routes** viene scartato (**reject**) e inviato un messaggio di notifica via ICMP
```
[edit routing-options]
user@R1# show
aggregate {
	defaults {
		community 1:888;
	}
	route 172.29.0.0/22;
	route 172.25.0.0/16 {
		community 1:999;
		discard;
	}
}
```
_Nota_: la sezione "default" raccoglie i parametri definiti di default per tutte le rotte aggregate per cui tali parametri non sono stati esplicitamente definiti.

| Options | Description |
| ----- | ----- |
| as-path | ... |
| community | ... |
| metric| ... |
| policy | Use all possible, more specific contributing routes to activate an aggregate route |
| preference | ... |

Per vedere le informazioni di dettaglio (preference, le singole contribuiting-route, next-hop) associate a una rotta aggregata è possible utilizzare tale comando:

`user@R1> show route 172.29.0.0/22 exact detail`

## Generated Routes

Allo stesso modo delle Aggregated Route, le Generated Route sono attive quando almeno una delle Contributing Route ad essa associate è attiva.
- La principale differenza è che per le Generated Route viene specificato un next-hop, o meglio, questo viene ereditato sulla base del next-hop del **_primary contributing route_** (ossia quella con preferenza più bassa o con numero di prefix più basso).
- Analogamente alle static route anche le "generated route" devono avere un next-hop valido e raggiungibile altrimenti vengono messe nello stato "_hidden_"

## Martian Addresses
Sono **host o network address** le cui informazioni di routing sono semplicemente ignorate.
- Martian Address non verranno mai installati nella routing table del device

Alcuni esempi di Martian addresses in IPv4 sono:
- 0.0.0.0/8 orlonger
- 127.0.0.0/8 orlonger
- 192.0.0.0/24 orlonger
- 240.0.0.0/4 orlonger
  
Altri martian address possono essere specificati nella configurazione a livello gerarchico `[edit routing-options]`.

_Nota_: "orlonger" è solo una delle keyword che rappresentano i "**matching type**":
- exact
- longer
- orlonger
- prefix-length-range
- through
- upto

Per verificare i martian address nella tabella IPv4 #0:
`show martian table inet.0`

