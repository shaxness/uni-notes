Il secondo soft fork molto importante in Bitcoin, dopo SegWit, è stato **Taproot** (2021). 

Taproot ha introdotto un insieme di miglioramenti alla struttura degli script e delle firme, e il ruolo centrale in ciò è stato svolto dall'implementazione delle **Firme di Schnorr**. Schnorr è uno schema di firma digitale considerato molto elegante ed affidabile, con proprietà matematiche particolarmente utili per Bitcoin soprattutto in quanto consente di **aggregare efficientemente più firme** in una singola firma, migliorando la privacy.

Quando Bitcoin fu creato, Satoshi Nakamoto non adottò Schnorr ma come sappiamo implementò ECDSA, lo schema basato sulle curve ellittiche già ampiamente utilizzato in crittografia. La motivazione principale era sicuramente che Schnorr all'epoca era ancora sotto brevetto, e Satoshi voleva evitare qualsiasi complicazione legale.

L'introduzione di Taproot ha beneficiato molto dell'update precedente SegWit, dal momento che quest'ultimo aveva già separato i dati di firma dal corpo principale della transazione spostandoli nella parte **witness**. Questo ha reso più semplice l'introduzione di nuove versioni di script e modi di verificare le condizioni di spesa senza dover modificare in modo invasivo la struttura base delle transazioni.

In particolare Taproot è implementato come una nuova versione di output SegWit, non si entra nei dettagli al riguardo. In questa lezione si introduce in particolare il funzionamento delle firme di Schnorr, e si spiega la questione dell'aggregazione delle firme.

### Firme di Schnorr
Le firme di Schnorr sono uno schema di firma digitale che può essere istanziato su un qualsiasi gruppo (ciclico) in cui **il problema del logaritmo discreto sia computazionalmente difficile**. 

In Bitcoin, Schnorr è applicato al gruppo dei punti della curva ellittica secp256k1. In questo gruppo:
- gli elementi del gruppo sono punti della curva ellittica
- l'operazione del gruppo è la somma di punti
- esiste un generatore pubblico $G$
- l'ordine del generatore è un numero primo $n$, poco inferiore a $2^{256}$. Con ordine del generatore si intende che il sottogruppo generato da $G$ ($\langle G \rangle$) ha esattamente $n$ elementi. Inoltre $nG = O$, dove $O$ è il punto all'infinito, che funge da elemento neutro per la somma di punti, per questo motivo quando prendiamo scalari random non includiamo $0$ e $n$ (O non è una chiave pubblica valida)

In quanto schema di firme, vediamo come funzionano KeyGen, Sign e Verify per Schnorr.

```text
KeyGen():

Public parameters:
    G = generator of the group
    n = order of G

1. choose x uniformly at random from {1, ..., n-1} #(secret key)
2. compute P = xG  #(public key)
3. return (x, P) 
```
Quindi la chiave privata è uno scalare $x$ scelto casualmente, e la chiave pubblica è il punto della curva $P$ ottenuto moltiplicando il generatore $G$ per $x$ secondo l'operazione somma definita per il gruppo.
```text
Sign(m, sk):

1. choose k uniformly at random from {1, ..., n-1}
2. compute R = kG
3. compute P = xG
4. compute h = H(R || P || m)
5. compute s = k + h x mod n
6. return sigma = (R, s)
```
Per firmare un messaggio $m$ con chiave privata $x$, Schnorr prevede di scegliere un nonce casuale $k \in [1, n-1]$, dopodiché si calcola $R = kG$ e poi una challenge tramite hash $h = H(R || P || m)$ (hash della concatenazione di $R$, $P$ e $m$). Infine si calcola la firma $s = k + h x \mod n$. La firma è quindi composta da $R$ e $s$.

La parte più interessante di Schnorr è che la firma $s$ ha una struttura **lineare** $s = k + h x$, e ciò che le consente di nascondere la chiave privata $x$ è l'aggiunta del nonce random $k$ (a chi osserva dall'esterno, $s$ sembra un numero casuale proprio perché $k$ è stato scelto random). 

Quindi qui la sicurezza dipende in modo cruciale (similmente ai ragionamenti fatti per ECDSA) dal fatto che $k$ sia segreto e imprevedibile. Se infatti si firmassero due messaggi diversi con lo stesso $k$, si potrebbe risalire facilmente ad $x$:
$$s_1 = k + h_1 x \mod n \qquad s_2 = k + h_2 x \mod n$$
Sottraendo membro a membro:
$$s_1 - s_2 = (h_1 - h_2) x \mod n \implies x = (s_1 - s_2)(h_1 - h_2)^{-1} \mod n$$
```text
Verify(m, sigma, pk):

1. compute h = H(R || P || m)
2. return sG == R + hP
```

> **Correttezza**:
>
> Vogliamo dimostrare che per ogni firma generata correttamente da Sign, questa sarà accettata da Verify.
>
> **Dimostrazione**:
>
> Da Sign abbiamo $s = k + h x \mod n$. Allora:
> $$sG = (k + h x)G = kG + hxG = R + hP$$
> che è esattamente la condizione che Verify controlla, quindi la firma sarà accettata.
>
> $\blacksquare$

La vera differenza tra Schnorr ed ECDSA non risiede solo nella sua semplicità, ma soprattutto **nel fatto che la sua struttura matematica è lineare**. 

Questo permette in particolare di **dimostrare in modo formale la sicurezza di Schnorr**, mostrando che se un attaccante riesce a produrre firme valide senza conoscere la chiave privata, ciò equivarrebbe a dire che l'attaccante è in grado di risolvere il problema del logaritmo discreto. Sarebbe a dire che **falsificare una firma Schnorr è difficile almeno quanto ricavare la chiave privata $x$ dalla chiave pubblica $P = xG$**.

Al contrario, in ECDSA (dal momento che la struttura della firma non è lineare) non esiste una dimostrazione formale di sicurezza, per questo Schnorr è considerato più elegante e affidabile.

Un altro punto fondamentale legato alla linearità della firma di Schnorr è la possibilità di **aggregare più firme** in una singola firma, e questo è un aspetto cruciale per Bitcoin.

### Aggregazione di firme
Torniamo ora al mondo LN e la vera utilità di Schnorr in Bitcoin, al di là della sicurezza. Immaginiamo di avere una funding transaction, sappiamo che questa è una transazione con output del tipo 2-of-2 multisig, quindi per spenderla è necessario che entrambe le parti (Alice e Bob) abbiano la firma della transazione.

Il problema con la costruizione multisig è che questa al momento della spesa (output) necessita che si rivelino diverse informazioni, tra cui **gli script della spesa**, le **chiavi pubbliche coinvolte**, le **firme dei partecipanti**.

Questo rappresenta un problema di privacy per molte ragioni: un osservatore della blockchain può facilmente capire se una transazione è un pagamento normale o una transazione di funding per un canale LN etc...

Con Schnorr la situazione cambia, per via del fatto che questo ha come proprietà fondamentale la **linearità, che permette di combinare chiavi pubbliche e firme**. Si vuole in particolare far sì che Alice e Bob siano in grado di firmare una transazione ma senza che entrambi conoscano la chiave privata aggregata. 

Ricordiamo che in Schnorr $P = xG$ e $s = k + h x \mod n$.  
Supponiamo che Alice abbia $x_A$ come chiave privata e $P_A = x_A G$ come chiave pubblica, e Bob $x_B$ e $P_B = x_B G$. Grazie alla linearità, possiamo costruire una chiave pubblica aggregata $P = P_A + P_B$, infatti:
$$P = P_A + P_B = x_A G + x_B G = (x_A + x_B) G$$
Quindi è come se esistesse una chiave privata aggregata $x = x_A + x_B \mod n$ tale per cui $P = xG$, ma nessuno dei due partecipanti conosce davvero tutto $x$.

Per firmare un messaggio $m$ con questa chiave aggregata senza che nessuno dei due partecipanti conosca $x$, si procede in questo modo:
1. Alice sceglie un nonce $k_A \in_u [1, n-1]$ e calcola $R_A = k_A G$, Bob sceglie $k_B \in_u [1, n-1]$ e calcola $R_B = k_B G$.
2. Alice e Bob si scambiano $R_A$ e $R_B$ e calcolano $R = R_A + R_B$.
3. Entrambi conoscono anche la chiave pubblica aggregata $P = P_A + P_B$, quindi si calcolano la challenge $h = H(R || P || m)$.
4. Alice produce la sua parte di firma $s_A = k_A + h x_A \mod n$, Bob produce $s_B = k_B + h x_B \mod n$, e la firma finale è ottenuta sommando le due parti $s = s_A + s_B \mod n$.

Quindi la firma aggregata finale sarà $\sigma = (R, s)$, e questa sarà una firma valida per il messaggio $m$ rispetto alla chiave pubblica aggregata $P$.

Verifichiamo il perché la firma aggregata è valida:
$$\begin{aligned}
sG &= (s_A + s_B)G = s_A G + s_B G = (k_A + h x_A)G + (k_B + h x_B)G \\
&= k_A G + k_B G + h(x_A G + x_B G) = R_A + R_B + h(P_A + P_B) = R + hP
\end{aligned}$$

Il vantaggio per Bitcoin è quindi che Alice e Bob possono costruire collaborativamente un output che, dall'esterno, appare associato a una **singola chiave pubblica** --> quando spendono cooperativamente, pubblicano una sola firma valida per quella chiave pubblica aggregata. Agli occhi esterni la transazione sembrerà una normale transazione con un output associato a una chiave pubblica, senza rivelare alcuna informazione su script o altro. Si parla in questo caso di **key path spending**.

### Nascondere gli script con Taproot
Sempre riguardo la privacy, Taproot introduce anche la possibilità di nascondere gli script di spesa (prima di Taproot, al momento della spesa era necessario rivelare tutto lo script di spesa con le varie condizioni, anche quelle non utilizzate).

Con Taproot l'output come visto è associato a una chiave pubblica aggregata $P$, e la spesa può avvenire in due modi:
1. **Caso cooperativo** --> **key path spending**: Alice e Bob collaborano e costruiscono la firma aggregata come visto prima, e spendono semplicemente con una firma valida per la chiave pubblica aggregata $P$. In questo caso non viene rivelato nulla sull'output, sembra una normale transazione con un output associato a una chiave pubblica.
2. **Caso non cooperativo** --> **script path spending**: se Alice e Bob non collaborano, allora non è possibile usare la firma aggregata --> è necessario ad una delle due parti riprendersi i soldi usando una delle due condizioni alternative previste nello script.  
Per migliorare la privacy, in Taproot, queste condizioni non sono messe in uno script piatto, ma organizzate nelle foglie di un **merkle tree**. Si ricorda che il merkle tree è una struttura ad albero in cui ogni nodo interno è l'hash dei suoi figli, e le foglie contengono i dati veri e propri (in questo caso le condizioni di spesa).  
Se volessi spendere ad esempio usando la condizione S1, non è necessario rivelare tutte le condizioni S2, S3, S4, ma solo la condizione S1 e il percorso di autenticazione (hash) che collega S1 alla radice del merkle tree, che è la condizione necessaria da verificare per effettuare la spesa. In questo modo, anche in caso di spesa non cooperativa, si rivela solo la condizione di spesa effettivamente utilizzata, e non tutte le altre condizioni alternative previste nello script.


### Accenni su Zero-Knowledge Proofs
Una **Zero Knowledge Proof** è un protocollo interattivo in cui un soggetto, detto **prover**, dimostra a un altro soggetto, detto **verifier**, che una certa affermazione è vera senza rivelare nient'altro oltre alla verità dell'affermazione.

es. voglio dimostrare di conoscere la dimostrazione di un teorema senza rivelare la dimostrazione stessa, o voglio dimostrare di conoscere una password senza rivelare la password.

Una Zero Knowledge Proof deve soddisfare tre proprietà fondamentali:
1. **Completeness**: se l'affermazione è vera e il prover onesto, allora il verifier accetterà la challenge
2. **Soundness**: se l'affermazione è falsa, un prover disonesto non dovrebbe riuscire a convincere il verifier che l'affermazione è vera (accettando la challenge), se non con probabilità trascurabile
3. **Zero Knowledge**: il verifier non impara nulla oltre al fatto che l'affermazione è vera

Nel metodo classico di autenticazione password, succede circa quanto segue: il client ha una password e il server non dovrebbe essere in grado di salvarla in chiaro. Salva quindi un valore derivato, ad esempio $H(password)$. Quando poi il client vuole autenticarsi, invia la password al server, che calcola $H(password)$ e lo confronta con il valore salvato.

Il problema di questo approccio è che il server entra comunque in contatto con la password in chiaro --> non è un protocollo zero knowledge. 

Una forma più vicina all'idea zero knowledge è l'autenticazione tramite **challenge**.  
Supponiamo che il client abbia una chiave segreta sk e che il server conosca la pk ad essa associata. Il protocollo di autenticazione potrebbe essere il seguente:
1. Il client invia una richiesta di autenticazione al server, dicendogli la sua pk 
2. Il server conosce la pk ma vuole testare che il client sia davvero lui e che quindi conosca la sk associata --> invia una challenge casuale $c$ al client
3. Il client firma $c$ con la sua sk e invia la firma al server
4. Il server verifica la firma usando la pk del client, se la verifica è corretta allora accetta l'autenticazione

In questo protocollo entra già più l'idea di zero knowledge, dal momento che il server non entra mai in contatto con la chiave segreta del client

Esiste in questo senso un protocollo di identificazione di Schnorr del tutto zero knowledge, che permette al verifier di essere convinto che il prover conosce la chiave segreta associata a una chiave pubblica, senza che il verifier impari nulla sulla chiave segreta stessa. Il protocollo è il seguente:
```text
1. Prover sceglie k casuale.
2. Prover calcola R = kG.
3. Prover invia R al verifier.
4. Verifier sceglie una challenge casuale h.
5. Verifier invia h al prover.
6. Prover calcola s = k + hx mod n.
7. Prover invia s al verifier.
8. Verifier controlla:
       sG = R + hP
```
Il meccanismo è quindi praticamente lo stesso della firma di Schnorr, ma con la differenza è che in questo protocollo interattivo la challenge $h$ è scelta dal verifier (mentre nella firma di Schnorr è calcolata come hash di $R$, $P$ e $m$). 

Il fatto che sia zero knowledge dipende dal fatto che il verifier non impara assolutamente nulla su $x$, e questo è chiaro in quanto questa stessa interazione poteva benissimo essere fatta da parte del verifier stesso, che non conosce $x$, come segue:
```text
1. Sceglie h casuale.
2. Sceglie s casuale.
3. Calcola:
       R = sG - hP
4. Restituisce la conversazione:
       (R, h, s)
```
In questo modo la verifica passa perché chiaramente $sG = R + hP$ --> il verifier può produrre da solo una conversazione apparentemente valida, senza conoscere $x$ --> la conversazione originale non può contenere informazioni su $x$ perché si può simulare anche senza (il punto è che nel caso dell'interazione il prover calcola $s$ usando la comnbinazione lineare $s = k + hx$, ma il verifier non può risalire a $x$ da $s$ proprio perché $k$ è scelto casualmente e quindi $s$ sembra un numero del tutto casuale, per questo poi la simulazione è possibile scegliendo $s$ u.a.r.)

Il passaggio dal protocollo interattivo (dove $h$ è challenge scelta dal verifier) alla classica firma di Schnorr che conosciamo (dove $h$ è calcolata come hash di $R$, $P$ e $m$) è chiamato **trasformata di Fiat-Shamir**.

Più in generale si parla di **protocolli $\Sigma$** quando si ha una struttura a tre fasi (commitment da parte del prover, challenge da parte del server, response da parte del prover ed eventualmente Accept/Reject) come quella del protocollo di identificazione di Schnorr (commit R, challenge h, response s). La trasformata di Fiat-Shamir rappresenta un metodo generico per trasformare un protocollo $\Sigma$ interattivo in una firma digitale non interattiva, sostituendo la challenge scelta dal verifier con una challenge calcolata come hash dei dati coinvolti nella firma.

### Alcuni cenni sulle reti P2P Bitcoin e Lightning
**Rete P2P Bitcoin**: Bitcoin si basa su una rete peer-to-peer per cui come ben sappiamo non esiste un server centrale che coordina tutti. Ogni nodo mantiene connessioni con altri nodi suoi peer, attraverso i quali riceve e propaga blocchi, transazioni e indirizzi di altri nodi.

In questo senso si può vedere la rete Bitcoin come un grafo fatto dai nodi (peer) e archi (connessioni TCP tra i peer) dove però ogni nodo **vede solo una parte locale della rete, ossia i suoi vicini diretti**. Non esiste di fatto una lista pubblica e completa di tutte le connessioni attive tra i nodi.

Gli studi sulla topologia dela rete Bitcoin dimostrano che le connessioni tra i nodi sono deliberatamente difficili da ricostruire. In passato ogni volta che qualcuno era riuscito a trovare un metodo per ricostruire la topologia gli sviluppatori di Bitcoin hanno modificato il protocollo introducendo adeguatamente randomness per non rendere più possibile quel tipo di ricostruzione.

**Perché non rendere pubblica la topologia?** Moltissimi motivi, tra cui:
1. **Ridurre il rischio di attacchi DDoS e attacchi mirati per sconnettere certi nodi**, che potrebbero essere più efficaci se un attaccante conoscesse la topologia completa della rete
2. **Migliorare la privacy dei nodi** --> se un osservatore conoscesse la topologia completa, potrebbe provare ad inferire da dove è partita una certa transazione (es. se una transazione appare prima vicino a un certo nodo/gruppo di nodi --> l'attaccante potrebbe ricondurla a un indirizzo IP o a una zona della rete)
3. **Evitare attacchi mirati alla propagazione**: Bitcoin dipende dalla propagazione rapida e affidabile di blocchi e transazioni --> se l'attaccante conosce la topologia, potrebbe provare a colpire nodi strategici per rallentare la diffusione di informazioni
4. **Evitare centralizzazione**: se topologia pubblica --> alcuni soggetti potrebbero sfruttarla per ottimizzare la propria posizione nella rete e ottenere vantaggi informativi

**Modello teorico studiato dal Prof**: nonostante la topologia Bitcoin non sia nota, si può provare a inferire alcune delle sue proprietà simulando con un grafo che si comporti similmente a lei. Immaginiamo un grafo con:
- $n$ nodi
- $d$ target degree (numero di connessioni che ogni nodo cerca di mantenere)
- $c \gt 1$ fattore di tolleranza ai guasti (ogni nodo cerca di mantenere $c \cdot d$ connessioni per essere sicuro di avere sempre almeno $d$ connessioni attive)

Quindi ogni nodo cerca di mantenere un numero di vicini compreso tra $d$ e $c \cdot d$. Si procede nella simulazione in round successivi, con le seguenti regole. Per ogni round:
1. Se il nodo ha meno di $d$ connessioni, cerca di connettersi a nuovi nodi scelti casualmente finché non raggiunge almeno $d$ connessioni
2. Se il nodo ha $k \gt c \cdot d$ connessioni, ne elimina $k - c \cdot d$ connessioni scelte casualmente per riportarsi a $c \cdot d$ connessioni
3. Se il nodo ha tra $d$ e $c \cdot d$ connessioni, non fa nulla

Il processo si ferma quando tutti i nodi arrivano a un numero di connessioni compreso tra $d$ e $c \cdot d$. Secondo gli studi del Prof, se $c \gt 1$ allora il processo converge in $O(logn)$ round (molto efficiente e scalabile). 

Inoltre la cosa interessante è che, anche senza coordinamento centrale, il grafo risultante risulta essere molto ben connesso, un **ottimo expander** (dove un expander è un grafo sparso ma molto connesso, sparso pochi nodi, molto connesso significa che non è facile separare il grafo in due parti senza tagliare molti archi).

**Crawler**: un crawler è un programma che cerca di scoprire quali nodi sono attivi nella rete Bitcoin. Per farlo può contattare i nodi Bitcoin suoi vicini e usare messaggi del tipo `getaddr` per chiedere loro di restituire una lista di alcuni indirizzi IP di altri nodi Bitcoin che conoscono. In questo modo, partendo da un nodo iniziale, il crawler può esplorare la rete e scoprire nuovi nodi.

Attenzione al fatto che scoprire nuovi nodi non significa necessariamente scoprire la topologia: con `getaddr` scopro nuovi indirizzi, ma non so effettivamente chi è connesso a chi.

**Rete Lightning**: nel caso Lightning si distinguono due reti:
1. **Rete peer-to-peer privata**: è la rete delle connessioni effettive dei nodi Lightning. Un nodo infatti volendo può aprire una connessione Lightning con chi vuole, senza necessariamente renderla pubblica e far sì che gli altri possano usare il suo canale come parte della rete. Questa rete non è pubblica però sicuramente riflette in parte il channel graph pubblico, dal momento che spesso i nodi comunicano con peer con cui hanno anche canali pubblici.
2. **Channel Graph pubblico**: necessariamente pubblico per far sì che chiunque voglia instradare un pagamento possa tentare di costruire il path.

Il Channel Graph è **molto più centralizzato** rispetto alla rete Bitcoin, quindi pochi nodi con grado molto alto e tanti con grado basso. Il motivo è abbastanza intuitivo: in Lightning aprire e mantenere canali costa, e solo per grandi rivenditori conviene mettere queste risorse a disposizione.

In questo senso entra in gioco la misura di **assortatività** di un grafo, ossia quanto nodi simili tendano a connettersi tra loro. Nel caso del grado, si è osservato che il channel graph ha **assortatività negativa** proprio perché i nodi con grado alto tendono a connettersi con nodi con grado basso, e viceversa. 

Un altro problema in LN è il fatto che la **liquidità è direzionale**: se ho un canale con Bob di capacità 1 BTC, ma il bilancio è 1 BTC da parte mia verso Bob e 0 da parte sua verso di me, allora Alice può inviare denaro o essere usata per instradarne, ma Bob non può essere usato per instradarne verso alice. Se la mia unica connessione è con Bob in questo modo, allora nessuno può instradare pagamenti verso di me.

In questo senso esistono servizi per risolvere il problema, come Triangoli: un triangolo è quando tre nodi aprono canali in modo coordinato, ad es. A -> B -> C -> A, in questo modo si crea un ciclo che permette di distribuire la liquidità in modo efficiente creando capacità sia in entrata che in uscita. 




