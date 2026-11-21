Per far comunicare due componenti distribuite possiamo lavorare a livelli di astrazione differenti:
- con la **programmazione di rete esplicita** lo sviluppatore usa direttamente le primitive di comunicazione di rete (socket, HTTP, etc..) per gestire lo scambio di messaggi.  
Questo approccio richiede di occuparsi di molti dettagli della comunicazione (costruzione ed invio richieste, ricezione, interpretazione risposte...)
- l'alternativa **implicita** è utilizzare un **middleware di comunicazione** (collocato logicamente, come anticipato in 4, tra applicazione e SO) che offre un'interfaccia a livello applicativo e permette di gestire operazioni comuni nascondendo i dettagli della comunicazione, che altrimenti lo sviluppatore dovrebbe implementare da sé --> in questo modo lo sviluppatore può concentrarsi sulla logica applicativa.

# RPC
**RPC** sta per **Remote Procedure Call**: *un client chiede l'esecuzione di una procedura che però si trova presso un altro processo (in genere su una macchina remota)*.

Con RPC il client chiama la procedura, il server riceve la richiesta e la esegue, e infine il risultato viene restituito al client.  
L'obiettivo principale di RPC è la **trasparenza della comunicazione**: nonostante bisogni affrontare tutti i problemi legati alla comunicazione di rete e si deve eseguire la richiesta su un'altra macchina, si vuole far sì che (per quanto possibile) il client veda l'accesso alla procedura remota come se fosse locale.

Moltissime realizzazioni di RPC: **Sun RPC** (C), **Java RMI**, **RPyC** (Python), **Go** (che le supporta nativamente), **gRPC** (da Google, che permette di far interagire componenti scritte anche in linguaggi diversi).

## Client e Server Stub
In una chiamata locale, chiamante e procedura condividono lo stesso spazio di indirizzamento e possono usare meccanismi come la pila di esecuzione per passare parametri e risultati.  
In una RPC questo chiaramente non è possibile, dato che client e server sono processi distinti con spazi di memoria separati.

In RPC, per simulare una chiamata locale, vengono introdotti due componenti intermediari, detti **stub** o **proxy**.

Il **Client Stub** è un componente che si trova nel processo del client e ha la stessa interfaccia della procedura remota. Quando viene invocato:
1. Riceve i parametri della chiamata
2. Effettua il **marshalling**, preparando il messaggio di richiesta
3. Invia la richiesta al server tramite il sistema di comunicazione
4. Riceve il messaggio di risposta
5. Effettua l'**unmarshalling** del risultato
6. Restituisce il risultato al client

Il **Server Stub** è invece il componente che gestisce la comunicazione lato server. Quando riceve una richiesta:
1. Effettua l'**unmarshalling** dei parametri
2. Individua e invoca la procedura applicativa richiesta
3. Riceve il risultato della procedura
4. Effettua il **marshalling** del risultato
5. Invia la risposta al client

Il flusso completo della RPC può essere quindi schematizzato così:

<img src="img/5_1.png" alt="RPC" width="400"/>

Gli stub implementano il così detto **Proxy Pattern**: rappresentano localmente la procedura remota e nascondono diversi dettagli implementativi di comunicazione.

## Requisiti e problematiche implementative delle RPC
Tra le problematiche principali che deve affrontare il middleware per effettuare la chiamata remota:
1. **Scambio dei messaggi**: identificare la procedura richiesta, trasportare i parametri e risultati e associare ogni risultato alla relativa richiesta.
2. **Eterogeneità dei dati**: garantire che client e server possano interpretare i dati anche usando rappresentazioni differenti
3. **Passaggio dei parametri**: se un parametro è un riferimento (indirizzo) in memoria locale, come fare a passarlo al server remoto?
4. **Semantica dei fallimenti**: stabilire le garanzie offerte in caso di perdite di messaggi, ritardi, crash...
5. **Binding**: individuare l'endpoint del server che implementa la procedura richiesta.

Per quanto riguarda il punto 1, abbiamo già introdotto i concetti di *client stub* e *server stub*, che si occupano anche di associare ogni richiesta al relativo risultato. Iniziamo quindi ad analizzare più nel dettaglio gli altri punti.

### Eterogeneità dei dati
Client e server possono usare piattaforme e architetture diverse, che rappresentano i dati in modo differente.  
Tra le differenze principali l'*endianess* (ordine dei byte), *dimensione dei tipi* (es. un intero può essere rappresentato con 2, 4 o 8 byte), *codifica dei caratteri* (es. ASCII, UTF-8..), padding e allineamento dei dati in memoria...

Per questi motivi è necessario concordare una rappresentazione dei dati adatta alla trasmissione, detta *wire encoding*. Per fare ciò, RPC utilizza i concetti di:
- **Marshalling**: preparazione complessiva dei dati da inviare. 
- **Unmarshalling**: interpretazione dei dati ricevuti e ricostruzione della rappresentazione locale.

Un concetto da non confondere con il marshalling è quello di **serializzazione**, ossia la rappresentazione di dati e oggetti come una sequenza di byte --> la serializzazione è parte del marshalling.

Per la conversione dei dati, esistono quattro strategie principali:
1. **Codifica indicata nel messaggio** == il mittente specifica il formato utilizzato e il destinatario effettua la conversione --> il limite di questo approccio è l'**overhead** nella dimensione del messaggio. 
2. **Conversione nel formato destinatario** == il mittente converte direttamente i dati nel formato del ricevente --> il limite è che il ricevente **deve conoscere tutti i formati degli altri sistemi**
3. **Broker** == un componente intermediario si occupa delle conversioni --> il limite è che il broker diventa quindi SPOF e collo di bottiglia
4. **Formato comune** == sia mittente che destinatario convertono i dati da e verso una codifica condivisa 

Di questi quattro approcci, il secondo non è ben scalabile: supponendo di avere $N$ formati diversi nella rete (uno per ogni nodo) --> ogni nodo deve essere in grado di convertire i propri dati in $N-1$ formati diversi --> $O(N^2)$ conversioni totali.

Il formato comune, al contrario, rappresenta una soluzione **scalabile**: ogni sistema deve saper convertire solo dal proprio formato a quello comune e viceversa --> $O(N)$ conversioni totali in una rete di $N$ sistemi di formati differenti.  
Inoltre, la soluzione a formato comune **riduce anche il coupling** dato che i vari nodi sono dipendenti solo dal formato comune e non dai formati degli altri nodi --> per queste ragioni **la soluzione a formato comune è quella prevalente nelle implementazioni RPC**.

### Passaggio dei parametri per riferimento
In una chiamata locale sappiamo distinguersi tre modalità principali di passaggio dei parametri:
- **per valore** se la procedura riceve direttamente una copia del dato --> le modifiche al parametro non modificano l'originale
- **per riferimento** se la procedura accede direttamente al dato originale tramite un riferimento (es. un puntatore) 
- **copy-restore**: qui la procedura chiamata riceve una copia del valore (*copy-in*), ma al termine della chiamata il valore modificato viene copiato nuovamente nel parametro originale (*copy-out*) --> la procedura lavora su una copia, ma il risultato finale modifica effettivamente la memoria del chiamante.

Nelle RPC chiaramente **un riferimento a un indirizzo di memoria del client non può essere usato direttamente dal server remoto**, dato che i due processi hanno spazi di indirizzamento diversi.  
Una possibile soluzione adottata nelle RPC consiste quindi nel *simulare il passaggio per riferimento tramite un* **copy-restore**:
1. Il client stub copia e traspette il dato al server (*copy-in*)
2. il server esegue la procedura lavorando sulla copia ricevuta
3. il server restituisce il dato eventualmente modificato
4. il client stub aggiorna il dato originale (*copy-out*)

Se la struttura contiene altri puntatori --> si procede ricorsivamente, copiando anche i dati puntati dai puntatori etc...

### Semantica dei fallimenti
Ci si ricollega al riassunto 4.  
Qui ci limitiamo a ricordare come **at-least-once** sia la semantica più adatta nel caso si debbano eseguire operazioni **idempotenti** (è sufficiente meccanismo RR1), mentre **at-most-once** è più complessa (RR1, RR2, DF) ma garantisce che se il server ha ricevuto la richiesta, allora la eseguirà al più una volta.

Ricordiamo che un'operazione non idempotente può essere protetta da duplicazioni introducendo identificativi univoci delle richieste e un meccanismo applicativo di deduplicazione, trasformando il receiver in un Idempotent Receiver; oppure trasformando dove possibile l'operazione in un'operazione idempotente (es. *incrementa di 47 di 1* --> *imposta 48*).

Per RPC non analizzeremo la semantica **exactly-once**, che sappiamo richiedere meccanismi avanzati di persistenza e recovery (la vedremo nelle MOM con Kafka), ricordiamo comunque che exactly-once serve per operazioni critiche dove si vuole garantire che la richiesta venga completata anche in caso di crash del server mid-processing.

### Binding
Prima di effettuare una RPC il client deve conoscere l'**endpoint** del server (i.e. le informazioni necessarie per contattarlo), tipicamente un indirizzo IP e una porta.  
Il processo attraverso il cui il client identifica l'indirizzo remoto è detto **binding**.

Si distingue tra:
- **Binding Statico** se l'endpoint viene stabilito prima dell'esecuzione, ad es. nel codice o nella configurazione del client.  
Questo approccio è **semplice dato che non richiede meccanismo dinamico di discovery**, ma anche **poco flessibile** (se l'indirizzo cambia, il client deve aggiornare le informazioni utilizzate) e quindi **poco trasparente**
- **Binding Dinamico** se invece il client individua l'endpoint durante l'esecuzione. Questo approccio permette quindi di **modificare la posizione** e **scegliere tra più repliche senza dover modificare il codice applicativo**, introducendo però ***overhead aggiuntivo per l'individuazione del server**.

Il Binding Dinamico distingue due concetti:
1. **Naming** == stabilisce quale servizio vuole sfruttare il client
2. **Addressing** == individuazione dell'endpoint concreto attraverso cui raggiungere quel servizio

ossia si deve distinguere tra il nome del servizio e l'indirizzo concreto del server che lo implementa.  
Nell'**Addressing Esplicito**, il client cerca direttamente il server, ad esempio tramite broadcast o multicast, attendendo una risposta.  
Nell'**Addressing Implicito** il client utilizza invece un componente dedicato detto **Service Registry** (o *Binder*/*Service Directory*/*Name Server*) per individuare il server.

Il **Service Registry** mantiene una tabella che mappa i nomi logici dei servizi agli endpoint disponibili.  
Il funzionamento tipico è il seguente:
1. Il server registra il proprio servizio e relativo endpoint nel registry
2. Il client interroga il registry, specificando il nome logico del servizio richiesto
3. Il registry restituisce l'endpoint, necessario per raggiungerlo
4. Il client invia la richiesta al server individuato

Il registry può inoltre supportare la rimozione e l'aggiornamento delle registrazioni, ad es. sulla base di timeout o heartbeat.  

<img src="img/5_2.png" alt="RPC" width="400"/>

Per ridurre l'overhead del binding dinamico tipicamente si utilizza una **cache** per evitare di ripetere il lookup ad ogni invocazione.  
Il binding dinamico favorisce quindi la **location transparency**, ma questa non è necessariamente completa: alcune implementazioni richiedono comunque la conoscenza di un hostname oppure dell'indirizzo del registry...

### Ulteriori problematiche
Ci sono anche altre problematiche oltre alle 5 appena viste.

**RPC sincrone e asincrone**: alcune RPC implementano comunicazione asincrona, che permette al client di inviare la richiesta senza attendere la risposta, continuando l'esecuzione --> si può ottenere **decoupling di sincronizzazione** tra client e server.  
Per quanto riguarda il **decoupling temporale**, le principali implementazioni RPC non lo supportano --> **comunicazione transiente**, per cui il middleware RPC non conserva normalmente le richieste inviate se il server non è disponibile.

**Limiti della trasparenza**: per quanto l'interfaccia RPC possa somigliare a una chiamata locale, il comportamento è diverso sotto molti aspetti:
- **prestazioni**: costi aggiuntivi in termini di serializzazione, copy-restore, attraversamento del SO e soprattutto latenza di rete (0.1-1 ms in LAN vs nanosecondi di una chiamata locale, milioni di volte peggio!)
- **fallimenti**: come visto con le semantiche client e server possono fallire indipendentemente, e i messaggi possono andare persi. Anche l'uso di TCP non garantisce che la procedura venga eseguita o che il client ne conosca il risultato... l'affidabilità del lvl di trasporto non risolve i guasti applicativi
- **concorrenza e replicazione**: non tutti i middleware RPC gestiscono automaticamente richieste concorrenti o repliche del server
- **sicurezza**: i middleware possono dover offrire meccanismi di autenticazione, autorizzazione, confidenzialità, integrità e protezione dagli attacchi di replay.

---

## IDL Interface Definition Language
Un problema importante delle RPC è definire un'interfaccia comune che client e server possano utilizzare, anche quando sono sviluppati in linguaggi diversi.

Una soluzione è rappresentata dall'**IDL Interface Definition Language**, un linguaggio che permette di descrivere formalmente l'interfaccia del servizio remoto.  
In particolare, un IDL definisce:
- nome del servizio
- operazioni disponibili
- parametri e relativi tipi
- tipi dei valori restituiti

L'IDL rappresenta quindi a tutti gli effetti il **contratto del servizio**, ma non contiene direttamente l'implementazione delle procedure.

Es.
```idl
service KeyValueStore {
    i32 put(1: string key, 2: string value);
    string get(1: string key);
}
```
Qui l'interfaccia descrive un servizio `KeyValueStore` che espone due operazioni:
- `put` riceve una chiave e un valore, entrambi stringhe, e restituisce un intero a 32 bit
- `get` riceve una chiave e restituisce una stringa

I numeri associati ai parametri rappresentano identificatori univoci dei campi, che vengono utilizzati per il marshalling e l'unmarshalling dei dati.

A partire dall'IDL, un apposito **compilatore IDL** può generare automaticamente i componenti necessari per utilizzare il servizio.  
In particolare, **il compilatore genera tutto il codice di supporto come gli stub client e server** (che ricordiamo implementano il Proxy Pattern occupandosi di marshalling, unmarshalling e gestione della comunicazione) --> **lo sviluppatore si limita a scrivere**:
- **la definizione IDL** del servizio
- **il codice applicativo del client**
- **l'implementazione delle procedure sul server**

*Chiaramente da distinguere tra compilatore IDL e eventuale compilatore del linguaggio di programmazione*, quest'ultimo compila solo il codice applicativo insieme ai componenti generati dal compilatore IDL e alle librerie necessarie.

<img src="img/5_3.png" alt="RPC" width="400"/>

Non tutte le implementazioni RPC usano un IDL separato: alcune sfruttano direttamente i meccanismi del linguaggio di programmazione.

---

# RPC use-cases