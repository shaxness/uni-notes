## Breve riepilogo su ECDSA e secp256k1
In Bitcoin lo schema di firma digitale usato originariamente è **ECDSA** (Elliptic Curve Digital Signature Algorithm).

Bitcoin utilizza una particolare curva ellittica chiamata **secp256k1**, definita sul campo discreto finito $\mathbb{F}_p$ con $p = 2^{256} - 2^{32} - 977$. La curva di ECDSA in generale è definita da un'equazione del tipo $y^2 = x^3 + ax + b$, ma per secp256k1 i parametri sono $a = 0$ e $b = 7$, portando all'equazione $y^2 = x^3 + 7$.

L'insieme dei punti della curva è 
$$\mathcal{C} = \{(x, y) \in \mathbb{F}_p^2 : y^2 \equiv x^3 + 7 \mod p\} \cup \{\mathcal{O}\}$$
dove $\mathcal{O}$ è il punto all'infinito, che svolge il ruolo di elemento neutro del gruppo.

L'operazione defnita sui punti della curva è l'**operazione somma**, grazie alla quale i punti della curva formano un gruppo. In particolare viene scelto un punto generatore $G$ della curva e si considera il sottogruppo generato da esso:
$$\langle G \rangle = \{kG : k \in \mathbb{N}\}$$
dove $kG = G + G + \ldots + G$ ($k$ volte). Calcolare questo sottogruppo è molto efficiente grazie all'algoritmo **douple and add**, simile all'esponenziazione modulare veloce. In breve l'idea dell'algoritmo è che se $k$ è pari allora $kG = (k/2)G + (k/2)G$, mentre se $k$ è dispari allora $kG = (k-1)G + G$, dimezzando ricorsivamente il problema --> **complessità $O(\log k)$** (logaritmico rispetto a $k$ == polinomiale rispetto al numero di bit di $k$).

Il gruppo generato da $G$ contiene $n$ elementi, dove nel caso secp256k1 $n$ è un numero primo molto grande, anch'esso vicino a $2^{256}$ ma ovviamente minore di $p$. 

In ECDSA, la **chiave privata** è scelta come un numero
$$\text{sk} \in [1, n-1]$$
mentre la **chiave pubblica** associata ad sk è il punto
$$\text{pk} = \text{sk} \cdot G$$
L'operazione per ottenere la pk a partire dalla sk è facile da calcolare (algoritmo double and add), mentre il problema inverso, cioè dato pk trovare sk, è noto come **problema del logaritmo discreto ellittico ECDLP** e non esiste attualmente un algoritmo efficiente per risolverlo.

Formalmente **ECDLP** è il seguente problema: dato $G$ e dato $\text{pk} = \text{sk} \cdot G$, trovare $\text{sk}$. La sicurezza di ECDSA si basa proprio sulla difficoltà di risolvere ECDLP.

## ECSDA come sistema di Firma Digitale
Uno schema di firme digitale è composto da tre algoritmi fondamentali: **KeyGen**, **Sign** e **Verify**. Nel caso di ECSDA, lavoriamo nel gruppo generato dal punto $G$ della curva ellittica secp256k1.

- **KeyGen() --> $(\alpha, \text{P})$**: 
  - si sceglie un valore $\alpha$ (la nostra sk) nell'intervallo $[1, n-1]$. Importante per questioni di sicurezza che la scelta di questo valore sia **u.a.r.** e avvenga tramite **un generatore pseudocasuale crittograficamente sicuro** (infatti essendo pseudocasuali, i generatori hanno natura deterministica e non sempre sono progettati per essere sicuri)
  - si calcola la chiave pubblica $\text{P} = \alpha \cdot G$ 
- **Sign($\alpha$, $m$) $\rightarrow \sigma$**: dove $m$ è una stringa (msg) e $\sigma$ è la firma digitale.
- **Verify($\text{P}$, $m$, $\sigma$) --> \{T, F\}**: restituisce True se la firma è valida, False altrimenti 

Affinché uno schema digitale sia considerato corretto e sicuro, deve soddisfare le proprietà:
- **Correttezza**: per ogni coppia di chiavi generate correttamente da **KeyGen()*, e per ogni messaggio $m \in \{0, 1\}^*$:
  $$\text{Verify}(\text{P}, m, \text{Sign}(\alpha, m)) = \text{True}$$
- **Sicurezza**: non basta che dalla chiave pubblica non si riesca a ricavare con un algoritmo efficiente la chiave privata (ECDLP), si vuole qualcosa di più forte: anche a patto che un attaccante abbia a disposizione un oracolo di firme che gli permetta di ottenere la firma di messaggi arbitrari $(msg_i, \sigma_i)$, non deve essere per lui possibile generare "facilmente" una firma valida per un messaggio $m$ non firmato, ossia una coppia $(\hat{msg}, \hat{\sigma})$ tale che $\text{Verify}(\text{P}, \hat{msg}, \hat{\sigma}) = \text{True}$. Questa proprietà è nota come **unforgeability == non falsificabile** (EUF-CMA, existential unforgeability under chosen message attack) 

Come vedremo la correttezza di ECSDA è facilmente dimostrabile, tuttavia la **sicurezza** è molto più complessa per via del fatto che la struttura algebrica di ECSDA è "artificiale" e quindi difficile da ridurre direttamente a ECDLP. Esistono schemi molto più puliti matematicamente, come le **Firme di Schnorr**, che è considerato teoricamente migliore e più semplice da analizzare di ECDSA. Tuttavia quando Bitcoin è stato progettato, le firme di Schnorr erano coperte da brevetto e quindi non utilizzabili liberamente, per questo Satoshi optò per ECSDA. 

Ad oggi il brevetto di Schnorr è scaduto e infatti Bitcoin supporta anche questo schema grazie all'upgrade **Taproot**.

### Sign e Verify di ECDSA

```text
Sign(msg, α):

1. h ← Hash(msg)
   if h ≥ n:
       h ← most significant log2(n) bits of h

2. scegli k ∈ {1, ..., n−1} uniformemente a caso

3. R ← kG
   R = (x_R, y_R)

4. r ← x_R mod n

5. s ← k^(-1) (h + rα) mod n

6. return σ = (r, s)
```

Per prima cosa quindi l'algoritmo di firma calcola $h = \text{Hash}(msg)$, dove Hash è una funzione di hash crittografica (in Bitcoin si usa SHA256). Questo valore deve essere $\lt n$ perché in ECSDA le operazioni sul gruppo avvengono modulo $n$ (ordine del gruppo generato da $G$). Dal momento che SHA256 restituisce un output di 256 bit (praticamente u.a.r.) e che $n$ è un numero primo molto vicino a $2^{256}$, le probabilità che $h \ge n$ sono praticamente zero, però se dovesse succedere l'algoritmo prevede di prendere primi $\log(n)$ bit di $h$ più significativi per ottenere un valore $h$ valido.

Poi calcola $k$ u.a.r in $[1, n-1]$ e calcola il punto $R = kG$, di cui ci interesserà solo la coordinata $x_R$ per calcolare $r = x_R \mod n$. Infine calcola $s = k^{-1} (h + r\alpha) \mod n$ e restituisce la firma $\sigma = (r, s)$. Tale firma è composta da due numeri ognuno da 32 byte, per un totale di 64 byte (512 bit).

```text
Verify(msg, P, σ = (r,s)):

1. h ← Hash(msg)
   if h ≥ n:
       h ← most significant log2(n) bits of h

2. u1 ← s^(-1) h mod n

3. u2 ← s^(-1) r mod n

4. Q ← u1 G + u2 P
   Q = (x_Q, y_Q)

5. return (x_Q mod n == r)
```

Si noti che il punto cruciale nel verificatore ECSDA sta nel fatto che é possibile ricostruire $kG$ senza conoscere né $k$ né la chiave privata $\alpha$. Inoltre i calcoli effettuati in entrambi gli algoritmi possono essere fatti in tempo efficiente.

>**Teorema: Correttezza di ECDSA**
>
>Dimostriamo ora la correttezza di ECSDA, ossia che se $\sigma$ è una firma valida per $msg$ con chiave privata $\alpha$ e chiave pubblica $P = \alpha G$, allora la verifica restituisce True.
>
> **Dimostrazione**
>
> La correttezza deriva direttamente dall'equazione usata per costruire $s = k^{-1} (h + r\alpha)$. Moltiplicando tutto per $k$ otteniamo $k s = h + r\alpha$ e quindi
> $$k = s^{-1} (h + r \alpha)$$
> Analizziamo ora il valore calcolato da Verify $u_1 G + u_2 P$, sostituendo con le definizioni di $u_1$ e $u_2$ otteniamo
> $$u_1 G + u_2 P = s^{-1} h G + s^{-1} r P$$
> Poiché $P = \alpha G$, possiamo riscrivere:
> $$u_1 G + u_2 P = s^{-1} h G + s^{-1} r \alpha G = s^{-1} (h + r \alpha) G$$
> ma dalla definizione di $s$ sappiamo che $s^{-1} (h + r \alpha) = k$, quindi
> $$u_1 G + u_2 P = k G = R$$
> Quindi il punto ricostruito da Verify coincide proprio con il punto generato in fase di firma -> $x_Q == r$ restituisce True.
> 
> $\blacksquare$

### L'importanza della scelta di $k$ in ECDSA
In ECSDA il valore casuale $k$ (detto nonce effimero) è **estremamente critico** per la sicurezza dello schema. Se infatti vengono generate due firme usando lo stesso $k$ allora **è possibile recuperare la chiave privata $\alpha$**.

Supponiamo di avere due messaggi distinti $m_1$ e $m_2$ e che per entrambi venga usato lo stesso $k$. Allora avremo due firme $\sigma_1 = (r, s_1)$ e $\sigma_2 = (r, s_2)$, dove $r$ è lo stesso perché dipende solo da $k$. Dalla definizione di $s$ otteniamo:
$$s_1 = k^{-1} (h_1 + r\alpha) \qquad s_2 = k^{-1} (h_2 + r\alpha)$$
moltiplicando entrambe per $k$:
$$k s_1 = h_1 + r\alpha \qquad k s_2 = h_2 + r\alpha$$
sottraendo membro a membro evitiamo la necessità di conoscere $r$ ed $alpha$, possiamo calcolare $k$ come:
$$k = \frac{h_1 - h_2}{s_1 - s_2}$$
Una volta ottenuto $k$, possiamo sostituirlo in una delle due equazioni per ottenere $\alpha$:
$$\alpha = \frac{k s_1 - h_1}{r}$$

Questa vulnerabilità è stata sfruttata in passato in diverse occasioni, una eclatante è stata quella del caso PS3 di Sony, dove ECSDA veniva usata per firmare software: la console eseguiva solo codice con firma valida. L'implementazione di Sony riutilizzava lo stesso valore di $k$ nelle firme, permettendo agli hacker di recuperare la chiave privata e quindi poter produrre codice che sembrava firmato da Sony, permettendo quindi di far girare qualsiasi copia di giochi piratati.

Quindi è fondamentale che ci sia una sorgente random nella scelta di $k$ ogni volta che si firma. Qui però nasce un **problema**: le firme vengono generate continuamente --> è necessario produrre continuamente nuovi valori casuali $k$ --> è necessario avere una sorgente PRNG (pseudo random number generator) molto forte, ma questo è difficile. 

Per evitare di dipendere continuamente da randomness "fresca", la soluzione moderna in questo senso è il **deterministic ECSDA**. Invece di scegliere $k$ casualmente ogni volta, si genera $k = \text{H}(\alpha, m)$ in modo deterministico ma pseudocasuale, sfruttando più precisamente la funzione hash HMAC-SHA256. 

Questa soluzione permette di evitare di scegliere $k$ usando un generatore di numeri casuali, ma allo stesso tempo garantisce che $k$ sia diverso per ogni messaggio (perché dipende da $m$) e che agli occhi di chi non conosce $\alpha$ appaia comunque pseudocasuale (perché HMAC-SHA256 è una funzione di hash crittografica).

Una conseguenza di deterministic ECDSA è che chiaramente **se firmo due volte lo stesso msg ottengo lo stesso k e quindi la stessa firma**, ma questo non rappresenta un problema.
