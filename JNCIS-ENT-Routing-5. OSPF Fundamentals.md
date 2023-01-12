#routing #juniper #igp #ospf
#   Fundamentals OSPF

E' un protocollo di tipo **IGP - Interior Gateway Protocol**, ossia da utilizzarsi all'interno di un unicum rappresentato dal **AS - Autonomous System** (rete proprietaria).
OSPF è un protocollo di tipo **link state** e quindi meno suscettibile rispetto ai protocolli _distance vector_.
- scambia con altri router (che eseguono a loro volta OSPF) **LSA - Link State Advertisement** ossia informazioni relative alle rotte/reti note al presente router e al loro status;
- con gli LSA ricevuti ogni router costruisce una propria "vista" sulla topologia della rete memorizzando queste informazioni nel **LSDB - Link State Database**. Tra le info memorizzate sotto forma di record:
    - router ID che ha inviato l'LSA
    - rete e router noti
    - costo associato per raggiungere le destinazioni sopra riportate

_Nota_: perché l'algoritmo funzioni stabilmente e fornisca informazioni di routing affidabili tutti i _router che eseguono OSPF e fanno parte della stessa **Area** devono avere un LSDB identito fra loro_.

- altri router che eseguono OSPF formano tra loro delle **adiacenze**;
- le info fornite tramite gli LSA ricevuti alimentano l'algoritmo per il calcolo del miglior percorso per ogni destinazione nota, detto anche **SPF - Shortest Path First** o **Dijkstra Algorithm**
    - crea un albero di percorsi con cui raggiungere ogni destinazione che viene "potato" man mano fino a raggiungere il percorso con costo minore totale associato;
    - tale rotta viene direttamente consegnata alla RIB del router per entrare a far parte della tabella di routing


- Network Mask valutato solo in casi di mezzo trasmissivo comune tipo Ethernet con un'unico dominio di broadcast. Serve perché tutti i router OSPF concordino sulla **subnet mask** da utilizzare.
- Hello Interval - by default sarebbe ogni 10s ma ogni router che forma adiacenza deve concordare su questo tempo
- Tempo che si attende per ricevere un Hello Packet prima di rimuovere un'adiacenza
- Options - campo da 8 bit fondamentale per il corretto funzionamento di OSPF


#

OSPF definisce 5 tipologie di messaggi all'interno della sua RFC:
## <mark>HELLO</mark> 
Ogni **10s** invia questo messaggio su tutti i suoi link in formato **multicast**.
    
- 224.0.0.5 indirizzo multicast per tutti i router che eseguono OSPF 

- In aggiunta al header OSPF i messaggi Hello contengono i seguenti campi:


| Network Mask** | Hello Interval* | Dead Interval* | Options* |
| ----- | ----- | ----- | ----- |
| **Router Priority** | **Designated Route** | **Backup Designated Route** | **Neighbor** |

**_Nota_**: <mark>I campi marcati con * sono campi importanti per la formazione di adiacenze tra router. Due router OSPF devono condividere tali parametri per poter formare un'adiacenza su di un mezzo broadcast. Nel caso di link **P2P non è necessario** che sia condivisa anche la **Network Mask**</mark>

#
## <mark>DATABASE DESCRIPTION</mark>
Scambiato solo **al momento della negoziazione di un adiacenza OSPF**. Allo scopo di determinare chi è responsabile della sincronizzazione del LSDB e per scambiarsi LSA Header.
- Descrive il contenuto del LSDB;
- Il messaggio consiste del **OSPF Header** + **Sequence Number** + **LSA Headers**;
- Il router con RID (Router ID) più grande è automaticamente nominato come il router responsabile del processo di sincronizzazione del LSDB tra i due router formanti l'adiacenza;
    - Questo perché nel formare un'adiacenza, abbiamo già detto che, due **router devono condividere le informazioni nel LSDB** --> sincronizzare i due DB secondo un processo master slave (che prende luogo solo in questa fase)
    - Il sequence number è scelto/mantenuto da router primario al fine di cadenzare il processo di sincronizzazione
    - L'altro router riceve ed eventualmente agisce come Backup primary router


- <mark>LINK-STATE REQUEST</mark>
- <mark>LINK-STATE UPDATE</mark>
- <mark>LINK-STATE ACKNOLEDGEMENT</mark>