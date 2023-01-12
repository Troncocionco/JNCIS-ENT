#routing #juniper 
# 1. Load Balancing

Equal-cost multipath per traffico accumunato dal medesimo prefix-number destination.

## 1.1 Per-Packet Load Balancing
Pacchetti sono inviati in modalità **round-robin** su path con medesimo costo verso la destinazione.
- equal distribution
- potrebbe necessitare il riordinamento dei pacchetti **deteriorando le performance**

<mark>Nota</mark>: apparati più recenti rilasciati da Juniper non implementano questa modalità

## 1.2 Per-Flow Load Balancing
A differenza del primo, i <mark>pacchetti che fanno parte dello stesso flusso</mark> vengono forwardati attraverso la <mark>medesima interfaccia egress</mark>.

- generalmente garantisce l'arrivo nell'ordine in cui sono stati emessi i pacchetti (non è necessario riordino);
- più facile applicazione dei meccanismi di QoS

By default Junos interpreta come facenti parte dello stesso flusso, pacchetti che:
- condividono la ingress port
- source packet adddress
- protocol

Questo aspetto ovviamente può essere modificato per discriminare traffico anche in base ad altri parametri di L3 e L4. 

**IGP** - Il concetto è che Junos sceglie uno tra i next-hop con medesimo costo (equal-cost) per un certo _destination prefix_. **Solo questo next-hop selezionato viene installato nella FIB (Forward Information Base o forwarding table)**.

_Nota_: chiaramente il processo di selezione del next-hop è dipendente dallo specifico protocollo (IGP)

**BGP** - <mark>Per-Prefix Load Balancing</mark> ossia quando le rotte vengono ricevute da un internal peer BGP e queste condividono il medesimo attributo "_BGP next-hop_" (ossia un indirizzo IP). Tutte le rotto che matchano il prefix indicato saranno distribuite lungo il network-path.

# L3 Load Balancing
Per alterare il comportamento del load balancer è possibile utilizzare le policy:

- In questo caso stiamo applicando la policy di load balancing a tutte le rotte (**_perché non è stato specificato un termine <mark>from</mark>_**) presenti nella routing table
```
[edit policy-options]
user@R1# show
policy-statement load-balance-all {
	then {
		load-balance per-packet;
	}
}
```
- Se avessimo voluto matchare delle rotte specifiche avremmo potuto specificare, come nell'esempio:
```
[edit policy-options]
user@R1# show
policy-statement load-balance-some {
	from {
		router-filter 172.24.0.0/24 exact;
		router-filter 172.24.1.0/24 exact;
	}
	then {
		load-balance per-packet;
	}
}
```
<mark>load-balance per-packet</mark> tale istruzione non significa che il load balancing è di tipo _per-packet_, questo dipende esclusivamente dal tipo di tecnologia supportata dal device. 
- Generalmente per le piattaforme nuove è sempre per-flow (fino a 64 equal cost path balancing)
- Solo nelle piattaforme vecchie è veramente per-packet (fino a 8 equal cost path balancing)

Perché questa policy sortisca qualche effetto/venga applicata deve essere configurata come una export-policy all'interno della gerarchia `[edit routing-options]`. Questo fa si che che le equal cost paths vengano installate di conseguenza sulla **forwarding table** di conseguenza.
```
[edit routing-options]
user@R1# show
forwarding-table {
	export <policy-name>;
}
```
Junos abbiamo già detto che considera un flusso sulla base dei soli parametri livello 3. Per far si che questo contempli anche parametri di livello 4 va ampliata la configurazione del meccanismo di hashing (_hash-key_) a livello gerarchico `[edit forwarding-options]`:
```
[edit forwarding-options]
user@R1# show
hash-key {
	family inet {
		layer-3;
		layer-4;
	}
}
```
_Nota_: va tenuto anche 'layer-3' se si vogliono ispezionare comunque anche parametri di livello 3. Esistono anche le famiglie 'multiservice' e 'mpls'.


### Step procedura per implementare load balancing
- creare la policy
- applicare la policy alla forwarding table

_installazione delle multiple next-hop per la matching prefix destination nella forwarding table_

- modificare eventualmente i parametri di hashing-key per il controllo dei flussi
- per verifica controllare i next-hop installati nella forwarding table tramite il comando `user@R1> show route forwarding-table`

_Nota_: la medesima configurazione ottenuta con gli step precedenti deve essere replicata ad ambo gli estremi del collegamento dove insistono i link multipli da bilanciare.