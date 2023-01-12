# Filter-Based Forwarding FBF

Ossia quella funzionalità che permette al router di effettuare il forwarding dei pacchetti non più soltanto in base al **Destination Prefix**.
Di seguito gli step per la configurazione di queste funzionalità:

1. Creare un **matching filter criteria** sotto la gerarchia _[edit firewall]_ e permette l'associazione tra i pacchetti e le routing instance target
2. Creare una routing instance per creare path di forwarding unici per flussi di traffico che matchano particolari condizioni
3. Creare un RIB - Routing instance group 

## Creazione del filtro

Nel primo step va definita una regola firewall, **da applicare come _input rule_ sul traffico ingress del router**.
_Nota_: questa è definita da termini **from** e **then** esattamente come le altre regole firewall ma contiene tra le **action** nella sezione then l'azione "_routing-instance_" per indicare la routing instance a cui affidare i traffico che matcha i criteri di filtering impostati. Tutto quello che non è matchato dalla regola di firewall by default viene scartato!

    [edit firewall]
    user@R1# show
    family inet {
        filter <filter-name> {
            term <term-one> {
                from {
                    <matching-conditions>
                }
                then {
                    routing-instance <routing-instance-name>
                }
            }
        }
    }

<mark>routing-instance-name</mark> è l'istanza a cui faremo riferimento di seguito.

## Associazione alla routing instance
Una volta trovato il match tra il pacchetto ed il filter, il primo viene valutato rispetto alla RIB associata alla Routing-Instance specificata poco prima. In alternativa alla valutazione è possibile definire:
- una rotta statica indicando il next-hop(ip-address)
- consegnare il pacchetto ad un'altra routing instance per ulteriore valutazione

        [edit routing-instances]
        user@R1# show
        instance-name {
            instance-type forwarding;
            routing-options {
                static {
                    route 0.0.0.0/0 next-hop ip-address;
                }
            }
        }
        instance-name2 {
            instance-type forwarding;
            routing-options {
                static {
                    route 0.0.0.0/0 next-table inet.0;
                }
            }
        }

Il parametro distintivo per FBF in questo passaggio della configurazione è rappresentato dalla **_instance-type_** che deve essere <mark>forwarding</mark>.

## Creazione del RIB group

    [edit routing-options]
    user@R1# show
    interface-routes {
        rib-group inet group-name;
    }
    rib-groups {
        group-name {
            import-rib [inet.0 instance-name.inet.0];
        }
    }

Tramite la precedente configurazione andiamo ad importare tutte le interface route all'interno delle tabelle di routing facenti parte il RIB-group definito più sotto.

Le interface-route sono le rotte direttamente connesse, ossia quelle destinazioni raggiungibili in base all'indirizzo ip configurato sull'interfaccia e in base a cosa è collegato fisicamente l'apparato.

**_Nota_**: è fondamentale la creazione del RIB-group per poter sfruttare anche queste direct-route, le quali altrimenti verrebbero installate esclusivamente nella _master_ routing instance del router.