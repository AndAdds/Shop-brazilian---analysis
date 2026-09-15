# Power BI — Lab alternativi "ShopBrasil"
## FILE 3 di 3 — TEORIA

> Leggi la sezione corrispondente **prima** di provare il lab.
> Ogni sezione si legge in 10-15 minuti.

---

# Teoria per LAB A1 — ETL

## Cos'è l'ETL e perché occupa metà del lavoro

ETL sta per Extract, Transform, Load. In Power BI il motore che lo esegue si chiama **Power Query**, e il linguaggio dietro si chiama **M**.

L'ordine mentale corretto è: *prima decidi che domande vuoi rispondere, poi importi solo quello che serve a risponderle*. Chi importa tutto "per sicurezza" si ritrova con modelli lenti e illeggibili.

Le quattro cose che fai in ETL:
1. **Ridurre il volume** — righe e colonne che non useranno mai nessuna analisi
2. **Rinominare** — `order_purchase_timestamp` è un nome da database, `Data Ordine` è un nome da report
3. **Pulire** — valori nulli, formati incoerenti, duplicati
4. **Ristrutturare** — dare alle tabelle la forma che serve al modello, non quella che avevano nella sorgente

## Import vs DirectQuery

| | Import | DirectQuery |
|---|---|---|
| Dove stanno i dati | in memoria, compressi | restano nella sorgente |
| Velocità query | millisecondi | dipende dal DB |
| Freschezza | all'ultimo refresh | sempre attuale |
| Funzioni DAX | tutte | limitate |
| Limite dimensione | la RAM disponibile | nessuno |

**Regola pratica:** Import salvo requisito esplicito di real-time o volumi che non stanno in memoria. Nei lab Epicode la formula "non è necessario un report real-time / minimizzare i tempi di caricamento" è la firma di **Import**: è messa lì apposta come indizio.

C'è anche **Dual** (la tabella è sia Import che DirectQuery, il motore sceglie) e **Composite** (modalità miste nello stesso modello): argomento avanzato, ti basta sapere che esistono.

## La compressione VertiPaq — perché conta

Power BI non salva i dati come una tabella. Li salva **per colonna**, compressi con tre tecniche: dizionario, run-length encoding, value encoding.

Conseguenza pratica che cambia il modo in cui lavori:

**La cardinalità (numero di valori distinti) pesa più del numero di righe.**

Una colonna con 100 milioni di righe ma solo 8 valori distinti (es. `order_status`) occupa pochissimo. Una colonna con 100.000 righe tutte diverse (es. un timestamp al secondo, o un GUID) occupa moltissimo.

Da qui tre regole:
1. Elimina le colonne ID che non servono a relazioni
2. Separa data e ora in due colonne se ti serve solo la data (abbatti la cardinalità di un fattore 86.400)
3. Non importare colonne "per sicurezza"

## Profilazione dati

Power Query offre tre strumenti nella scheda Visualizza:
- **Qualità colonna** — % di valori validi, errori, vuoti
- **Distribuzione colonna** — quanti valori distinti e quanti unici
- **Profilo colonna** — statistiche complete sulla colonna selezionata

**Trappola:** di default profilano solo le **prime 1000 righe**. In basso a destra trovi l'interruttore per passare all'intero set. Su un dataset da 100k righe, profilare 1000 righe può nasconderti completamente un problema.

## Query Folding

Quando la sorgente è un database, Power Query cerca di tradurre i tuoi passaggi in SQL e farli eseguire **al database** anziché in locale. Si chiama query folding ed è la differenza tra un refresh di 10 secondi e uno di 10 minuti.

Il folding si **rompe** con certe operazioni (aggiunta di colonne personalizzate complesse, alcune funzioni M). Tasto destro su un passaggio > **Visualizza query nativa**: se è disponibile, il folding regge fino a lì.

Con sorgenti CSV il folding non esiste (non c'è un motore query a monte), quindi in questi lab non è un problema — ma sappilo per i colloqui.

---

# Teoria per LAB A2 — Modellazione

## Schema a stella: il concetto più importante del corso

Un modello dati ben fatto ha due tipi di tabella:

**Tabella dei fatti** — contiene le misure numeriche e le chiavi esterne. È lunga (molte righe) e stretta (poche colonne). Nel nostro caso: `order_items`, una riga per ogni prodotto in ogni ordine.

**Tabelle dimensione** — contengono gli attributi con cui filtri e raggruppi. Sono corte e larghe. Nel nostro caso: prodotti, clienti, venditori, calendario.

Disegnate su carta, le dimensioni circondano i fatti come i raggi di una stella. Da qui il nome.

**Perché non basta una tabella unica (flat table):**
1. Ripeti gli stessi attributi milioni di volte → modello gonfio
2. Gli slicer diventano lentissimi (devono scansionare i fatti per trovare i valori distinti)
3. Non puoi avere dimensioni condivise tra più tabelle fatti
4. Le gerarchie diventano ambigue

**Snowflake** è quando una dimensione è collegata non ai fatti ma a un'altra dimensione (`order_items → orders → customers`). Funziona, ma ogni salto costa. Si accetta quando normalizzare costerebbe più di quanto rende.

## Cardinalità e direzione

**Cardinalità** descrive quante righe di una tabella corrispondono a quante dell'altra:
- **Molti a uno (\*:1)** — il caso normale: molti order_items, un prodotto
- **Uno a uno (1:1)** — raro, di solito significa che le due tabelle andrebbero unite
- **Molti a molti (\*:\*)** — da evitare quando puoi, gestire con tabella ponte quando devi

**Direzione del filtro incrociato:**
- **Singola** — il filtro va dalla dimensione ai fatti. È il default e va bene nel 95% dei casi.
- **Entrambe** — il filtro va anche dai fatti alla dimensione.

Il bidirezionale sembra comodo ("così filtra tutto") ed è una delle cause più frequenti di modelli rotti. Problemi che crea:
1. **Ambiguità di percorso** — se esistono due strade per arrivare alla stessa tabella, il motore non sa quale scegliere e può rifiutare la relazione
2. **Prestazioni** — ogni query deve considerare più propagazioni
3. **Risultati sorprendenti** — i filtri si propagano in modi che non avevi previsto

Usalo solo quando hai una tabella ponte per un molti-a-molti, o quando devi filtrare uno slicer in base a un altro.

## Relazioni attive e inattive

Tra due tabelle può esistere **una sola relazione attiva** (linea continua). Le altre sono inattive (tratteggiate) e si attivano solo dentro una misura con `USERELATIONSHIP`.

Caso tipico: gli ordini hanno data ordine e data consegna. Vuoi analizzare per entrambe? Crei due relazioni verso il calendario, una attiva e una inattiva:

```dax
Ordini per Data Consegna = 
CALCULATE(
    [Numero Ordini],
    USERELATIONSHIP( Calendario[Date], orders[Data Consegna Cliente] )
)
```

## Proprietà delle colonne che cambiano il comportamento

- **Riepiloga per** — se una colonna numerica non va sommata (un codice, un anno, un peso anagrafico), imposta "Non riepilogare". Altrimenti Power BI ci mette la sigma e produce numeri privi di senso.
- **Categoria dati** — dice a Power BI che quella colonna è un indirizzo, una città, uno stato, un URL, un'immagine. Serve per le mappe e per rendere cliccabili i link.
- **Ordina per colonna** — la colonna testuale si ordina secondo una colonna numerica. Indispensabile per i nomi dei mesi.
- **Nascondi** — non elimina, toglie dalla vista report. Nascondi tutti gli ID e tutte le colonne già dentro una gerarchia.

## Gerarchie

Una gerarchia è un raggruppamento ordinato di colonne che abilita il drill-down nelle visualizzazioni. Si crea con tasto destro su una colonna > Crea gerarchia, poi trascini dentro le altre.

Regola: l'ordine va dal **meno granulare al più granulare**. Categoria → Sottocategoria → Prodotto, mai il contrario.

---

# Teoria per LAB A3 — DAX di base

## Colonna calcolata vs misura: la distinzione che sblocca tutto

| | Colonna calcolata | Misura |
|---|---|---|
| Quando si calcola | al refresh | al momento della query |
| Dove si salva | in memoria, riga per riga | non si salva |
| Contesto | contesto di riga | contesto di filtro |
| Si può mettere in uno slicer | sì | no |
| Si può mettere su un asse | sì | no |
| Costo in memoria | alto | nullo |
| Costo in CPU | nullo a runtime | ad ogni interazione |

**La domanda da farsi:** *questo valore cambia in base a cosa l'utente seleziona?*
- Se **no** (la fascia di consegna di quell'ordine è quella, punto) → colonna calcolata
- Se **sì** (il fatturato dipende da quali filtri sono attivi) → misura

**Regola di default:** quando hai dubbio, fai una misura. Le misure non pesano in memoria e sono più flessibili.

**Regola aggiuntiva:** se puoi fare la colonna in Power Query invece che in DAX, falla in Power Query. Si comprime meglio e non allunga il tempo di refresh del modello.

## I due contesti

Questo è il cuore di DAX. Se lo capisci, il resto viene da sé.

**Contesto di riga** — esiste dentro una colonna calcolata e dentro le funzioni iteratrici (`SUMX`, `AVERAGEX`, `FILTER`...). Significa: "in questo momento sto guardando una riga specifica, e posso riferirmi ai suoi valori".

**Contesto di filtro** — esiste dentro una misura. È l'insieme di tutti i filtri attivi in quel momento: gli slicer, la riga della matrice in cui sto, la colonna, i filtri a livello di pagina.

La stessa misura `[Fatturato]` restituisce numeri diversi in celle diverse **non perché cambia la formula**, ma perché cambia il contesto di filtro in cui viene valutata.

**`RELATED`** funziona in contesto di riga e segue una relazione dal lato "molti" verso il lato "uno". Da `order_items` posso fare `RELATED(products[Categoria])` perché ogni riga ha un solo prodotto. Il contrario richiede `RELATEDTABLE`.

## CALCULATE: la funzione che modifica il contesto

`CALCULATE(espressione, filtro1, filtro2, ...)` fa due cose:
1. Applica i filtri che gli passi
2. Valuta l'espressione nel nuovo contesto

```dax
Fatturato Lenta = CALCULATE( [Fatturato], order_items[Fascia Consegna] = "Lenta" )
```

Il filtro che passi **sostituisce** il filtro esistente su quella colonna, non si aggiunge. Se l'utente ha già selezionato "Veloce" nello slicer, questa misura mostra comunque "Lenta". Per intersecare invece che sostituire, usi `KEEPFILTERS`.

`CALCULATE` fa anche una terza cosa importante: la **transizione di contesto**. Se lo chiami dentro un contesto di riga, trasforma quella riga in un filtro. È il motivo per cui `[Misura]` dentro un `SUMX` funziona.

## Variabili: usale sempre

```dax
Margine % = 
VAR Ricavi = [Fatturato]
VAR Costi = [Costo Venduto] + [Costi Spedizione]
RETURN DIVIDE( Ricavi - Costi, Ricavi )
```

Tre vantaggi:
1. **Leggibilità** — la formula si legge come un ragionamento
2. **Prestazioni** — una variabile si calcola una volta sola, anche se la usi tre volte
3. **Debug** — puoi fare `RETURN Ricavi` per vedere un passaggio intermedio

Attenzione: una variabile **cattura il contesto nel punto in cui è definita**. Se la usi dentro un `CALCULATE` che cambia il contesto, la variabile mantiene il valore vecchio. È una fonte di bug sottili — e anche uno strumento potente quando è quello che vuoi.

## La tabella calendario

Ogni modello con analisi temporali ne ha bisogno. Requisiti non negoziabili:
1. **Una riga per ogni giorno**, senza buchi
2. **Anni civili completi** (dal 1/1 al 31/12), altrimenti YTD e confronti anno precedente sbagliano ai bordi
3. **Marcata come tabella data**
4. **Nessuna relazione ad altre dimensioni**, solo ai fatti

Perché non usare la data automatica di Power BI: ne crea una nascosta per **ogni campo data del modello**, gonfia il file, e non puoi personalizzarla. Disattivala sempre (File > Opzioni > Caricamento dati > Data/ora automatica).

`CALENDAR(inizio, fine)` e `CALENDARAUTO()` sono le due funzioni base. `CALENDARAUTO` scansiona tutto il modello e prende il range più ampio — comodo ma può sorprenderti se hai una data anomala (una data di nascita del 1920 ti crea 100 anni di calendario).

---

# Teoria per LAB A4 — Time intelligence e override di contesto

## Le funzioni di time intelligence

Tutte richiedono una tabella data marcata come tale. Tutte restituiscono una **tabella di date**, che passi come filtro a `CALCULATE`.

| Funzione | Cosa restituisce |
|---|---|
| `SAMEPERIODLASTYEAR(date)` | lo stesso periodo, un anno prima |
| `DATEADD(date, n, unità)` | il periodo spostato di n unità |
| `PARALLELPERIOD(date, n, unità)` | l'**intera** unità spostata di n |
| `DATESYTD(date)` | da inizio anno alla data corrente |
| `TOTALYTD(expr, date)` | scorciatoia per `CALCULATE(expr, DATESYTD(date))` |
| `DATESINPERIOD(date, inizio, n, unità)` | un intervallo arbitrario |
| `PREVIOUSMONTH/QUARTER/YEAR` | il periodo precedente completo |

La differenza fondamentale è tra funzioni che **rispettano la granularità del contesto** (SAMEPERIODLASTYEAR, DATEADD) e quelle che **restituiscono il periodo intero** (PARALLELPERIOD, PREVIOUSYEAR). Se sei su marzo, le prime ti danno marzo scorso, le seconde tutto l'anno scorso.

## Le funzioni di rimozione filtro

| Funzione | Effetto |
|---|---|
| `ALL(tabella)` | rimuove ogni filtro dalla tabella |
| `ALL(colonna)` | rimuove i filtri da quella colonna |
| `ALLEXCEPT(tab, col1, col2)` | rimuove tutto **tranne** le colonne indicate |
| `ALLSELECTED()` | rimuove i filtri della visualizzazione, **rispetta gli slicer** |
| `REMOVEFILTERS()` | sinonimo moderno di ALL |
| `KEEPFILTERS()` | interseca invece di sostituire |

**Come scegliere per un calcolo di incidenza percentuale:**

Domanda da farsi: *rispetto a quale totale voglio la percentuale?*

- Rispetto al totale generale sempre → `ALL()`
- Rispetto al totale del livello superiore della gerarchia → `ALLEXCEPT()`
- Rispetto a quello che l'utente ha selezionato → `ALLSELECTED()`

Sbagliare qui produce il sintomo classico: **le percentuali non sommano a 100%** nei subtotali. Se lo vedi, sai dove guardare.

## RANKX

```dax
RANKX( tabella, espressione, [valore], [ordine], [ties] )
```

Due trappole:
1. Il primo argomento deve essere una tabella **senza il filtro corrente** su cui vuoi classificare — quasi sempre `ALL(colonna)`. Se passi la tabella filtrata, ogni riga si classifica contro se stessa e ottieni tutti 1.
2. Sulla riga del totale il rank non ha senso. Proteggilo con `HASONEVALUE` o `ISINSCOPE`.

`DENSE` vs `SKIP` per i pari merito: con due primi posti, DENSE dà 1,1,2 mentre SKIP dà 1,1,3.

---

# Teoria per LAB A5 — Molti a molti e design del report

## Molti a molti: il problema e le soluzioni

Il caso: un venditore serve più stati, uno stato è servito da più venditori. Non esiste un lato "uno".

**Soluzione con tabella ponte (bridge):**
Crei una tabella che contiene le combinazioni valide venditore-stato. Ora hai due relazioni molti-a-uno invece di una molti-a-molti. La direzione del filtro sulle relazioni ponte va impostata su "Entrambe" perché il filtro deve attraversare il ponte.

**Perché preferirla alla m:m nativa:**
1. Vedi la tabella, la puoi ispezionare, capisci cosa succede
2. Controlli tu la direzione dei filtri
3. `RELATED` continua a funzionare sulle singole relazioni
4. I totali si comportano in modo prevedibile

**Il problema dei totali non additivi:** in una relazione molti-a-molti il totale **non è** la somma delle righe visibili, perché una stessa transazione può appartenere a più righe. Questo è matematicamente corretto ma controintuitivo per l'utente finale. Quando costruisci un report su una m:m, o spieghi il comportamento o nascondi i totali.

## Principi di data visualization che il corso valuta

**Gerarchia visiva** — l'occhio legge in Z: alto a sinistra prima. I KPI principali vanno lì, il dettaglio in basso a destra.

**Scelta del grafico:**
| Domanda | Grafico |
|---|---|
| Come cambia nel tempo? | linee |
| Chi è più grande? | barre (orizzontali se le etichette sono lunghe) |
| Come si compone il totale? | barre impilate o waterfall, **non torta** |
| Che relazione c'è tra due misure? | dispersione |
| Dove? | mappa |
| Quanto vale adesso? | card |

**Cosa evitare:**
- Grafici a torta con più di 4 fette (l'occhio non confronta angoli)
- Assi che non partono da zero nei grafici a barre (esagerano le differenze)
- Più di 5-6 colori in una visualizzazione
- Colori che non significano nulla (usa il colore per codificare, non per decorare)
- 3D, sempre

**Accessibilità:** non affidare mai il significato al solo colore. Aggiungi icone, etichette o pattern. Circa l'8% dei maschi ha una qualche forma di daltonismo — in una presentazione aziendale ci sarà.

## Misure che restituiscono testo

Una misura può restituire una stringa, e quella stringa può alimentare un titolo dinamico o una casella di testo. È il modo per fare **narrazione automatica**: invece di lasciare che l'utente interpreti, gli dici tu cosa sta guardando.

`SELECTEDVALUE(colonna, alternativa)` restituisce il valore se ce n'è uno solo selezionato, altrimenti l'alternativa. È la funzione base per i titoli dinamici.

---

# Teoria per LAB A6 e A7 — Interattività e ottimizzazione

## Segnalibri

Un segnalibro fotografa lo stato corrente della pagina. Puoi scegliere **cosa** fotografa:

| Proprietà | Cosa salva | Quando tenerla |
|---|---|---|
| **Dati** | stato di slicer e filtri | quasi mai per i bottoni vista |
| **Visualizzazione** | quali oggetti sono visibili | sempre |
| **Pagina corrente** | su quale pagina sei | solo per navigazione |
| **Tutti gli oggetti / Selezionati** | ambito del segnalibro | "Selezionati" è più preciso |

L'errore classico: lasciare **Dati** attivo su un bottone che serve solo a cambiare vista. Risultato: l'utente seleziona un filtro, clicca il bottone, e il filtro sparisce.

## Drill-through vs drill-down vs tooltip

- **Drill-down** — scendere in una gerarchia dentro la stessa visualizzazione
- **Drill-through** — passare a un'altra pagina portandosi dietro il contesto di filtro
- **Tooltip pagina** — una pagina intera mostrata al passaggio del mouse

## Performance: come si ottimizza davvero

**Performance Analyzer** (scheda Visualizza) scompone il tempo di ogni visualizzazione in:
- **Query DAX** — il motore calcola
- **Visualizzazione oggetto** — il rendering
- **Altro** — attesa, sincronizzazione

Se il tempo è in Query DAX il problema è nel modello o nelle misure. Se è nel rendering, hai troppi oggetti o troppe righe in una tabella.

**Le cause di lentezza, in ordine di frequenza:**
1. `FILTER` usato dove bastava un predicato in `CALCULATE`
2. Colonne ad alta cardinalità importate senza motivo (timestamp, GUID)
3. Relazioni bidirezionali che moltiplicano i percorsi
4. `DISTINCTCOUNT` su milioni di valori distinti
5. Troppe visualizzazioni sulla stessa pagina (ognuna è una query)
6. Colonne calcolate DAX dove bastava Power Query

**Soglia di riferimento:** una visualizzazione sotto i 100 ms è ottima, sotto i 500 ms accettabile, sopra i 2 secondi va rivista.

## Power BI Service (Lab 5-6 mancanti, ma serve saperlo)

Il corso prevede anche la pubblicazione. Concetti chiave:
- **Workspace** — il contenitore condiviso dove pubblichi
- **Dataset semantico** — il modello, separato dai report che lo usano
- **App** — il pacchetto che distribuisci agli utenti finali
- **Aggiornamento pianificato** — fino a 8 volte al giorno in Pro, 48 in Premium
- **Gateway** — il ponte necessario per aggiornare dati che stanno on-premise
- **Row-Level Security (RLS)** — filtri applicati in base all'utente che apre il report

RLS si definisce in Desktop (Modellazione > Gestisci ruoli) con un'espressione DAX tipo `[customer_state] = "SP"`, oppure dinamica con `USERPRINCIPALNAME()`, e si assegna agli utenti nel Service.

---

## Glossario rapido

| Termine | Significato |
|---|---|
| **VertiPaq** | il motore di archiviazione colonnare in memoria |
| **Cardinalità** | numero di valori distinti in una colonna |
| **Granularità** | livello di dettaglio di una tabella fatti |
| **Contesto di riga** | "sto guardando questa riga" |
| **Contesto di filtro** | "questi sono i filtri attivi ora" |
| **Transizione di contesto** | quando CALCULATE trasforma una riga in un filtro |
| **Query folding** | Power Query traduce i passaggi in SQL |
| **Misura implicita** | l'aggregazione automatica su una colonna (evitala) |
| **Misura esplicita** | una misura che hai scritto tu (usa queste) |
| **Star schema** | fatti al centro, dimensioni intorno |
| **Tabella ponte** | tabella che risolve un molti-a-molti |

---

# Teoria per LAB A8 — Visual avanzati e parametri

## Albero di scomposizione

È un visual di **analisi esplorativa**: parte da una misura aggregata e la scompone per dimensioni successive, una alla volta, a scelta dell'utente.

Tre modalità di espansione:
- **Valore alto / Valore basso** — deterministiche, seguono il massimo o il minimo
- **Split ad alta variabilità** — algoritmo IA che cerca la dimensione più "esplicativa"
- **Manuale** — scegli tu la dimensione a ogni livello

Uso corretto: strumento di scoperta durante l'analisi, non oggetto da mettere in un report direzionale. Un dirigente che apre un albero di scomposizione e ci gioca dieci minuti trova cose interessanti; lo stesso dirigente che lo trova in una pagina di sintesi si chiede cosa deve guardare.

**Limite:** non funziona bene con dimensioni ad altissima cardinalità (migliaia di valori), perché i livelli diventano illeggibili.

## Piccoli multipli (small multiples)

Un solo tipo di grafico ripetuto in una griglia, uno per ogni valore di una dimensione. Tutti i pannelli condividono la stessa scala.

**Perché funzionano:** l'occhio confronta forme molto meglio di quanto confronti numeri o colori. Sei linee affiancate raccontano un pattern in un secondo; le stesse sei serie sovrapposte in un unico grafico sono spaghetti.

**Perché falliscono:** oltre ~12 pannelli compare lo scroll, e appena devi scorrere hai perso il confronto simultaneo, che era l'unico vantaggio. Se hai 71 categorie, filtri alle prime 6.

Regola: piccoli multipli quando vuoi **confrontare l'andamento** tra pochi gruppi. Grafico unico con serie multiple quando vuoi confrontare **i livelli** tra pochi gruppi.

## Misuratore (gauge)

Mostra un valore singolo rispetto a un obiettivo su una scala ad arco.

**Il problema del massimo:** il default è il doppio del valore corrente. Conseguenza: l'ago sta sempre esattamente a metà, e il visual non comunica nulla. Devi sempre impostare un massimo esplicito, e in un report dinamico quel massimo deve essere una misura.

**Critica onesta:** il misuratore è tra i visual meno efficienti in termini di dati per centimetro quadrato. Occupa molto spazio per comunicare un numero e una soglia — che una card con formattazione condizionale comunica in un quarto dello spazio. Il corso te lo fa fare perché è nei requisiti aziendali tipici, e perché saperlo configurare bene (massimo dinamico, parametro) dimostra padronanza. Ma in un report tuo, valuta un'alternativa.

## Parametri what-if

Un parametro what-if crea una **tabella disconnessa**: una tabella di valori che non ha relazioni con il resto del modello.

Perché funziona: la tabella non filtra niente, ma `SELECTEDVALUE` legge cosa l'utente ha selezionato nello slicer, e quel valore entra nella formula di una misura. È il meccanismo generale per rendere interattivo qualsiasi parametro di calcolo.

```dax
Valore Parametro = SELECTEDVALUE( 'Tabella'[Colonna], valore_default )
```

Il default nel secondo argomento è obbligatorio in pratica: senza, quando lo slicer non ha selezione la misura restituisce BLANK e tutto ciò che ne dipende sparisce.

Altri usi dello stesso pattern:
- **Selettore di misura** — l'utente sceglie quale metrica vedere su un grafico
- **Simulazioni di scenario** — sconto, cambio valuta, inflazione
- **Soglie configurabili** — l'utente decide cosa conta come "consegna lenta"

Il selettore di misura si fa così:
```dax
Misura Dinamica = 
SWITCH(
    SELECTEDVALUE( 'Selettore'[Metrica] ),
    "Fatturato",  [Fatturato],
    "Margine",    [Margine],
    "Ordini",     [Numero Ordini],
    [Fatturato]
)
```
(Dal 2023 esistono anche i **parametri di campo**, che fanno la stessa cosa nativamente: Modellazione > Nuovo parametro > Campi.)

## Relazione inattiva vs tabella ponte: quando usare quale

Sono due soluzioni allo stesso problema, con compromessi diversi.

**Relazione inattiva + USERELATIONSHIP**
- Vantaggio: DAX semplice, il modello resta compatto
- Svantaggio: il comportamento è nascosto dentro le misure. Chi guarda il modello non capisce cosa succede.
- Rischio: ogni tabella che aggiungi può creare nuove ambiguità di percorso

**Tabella ponte esplicita**
- Vantaggio: autoesplicativo, controllabile, prevedibile
- Svantaggio: una tabella in più, DAX un po' più verboso
- Rischio: se la generi con SUMMARIZE, va rigenerata al refresh

**Il principio del cammino minimo:** quando esistono più percorsi tra due tabelle, Power BI sceglie il più corto. Non te lo chiede, non te lo segnala. È il motivo per cui bisogna disattivare esplicitamente la relazione diretta quando si vuole forzare il percorso lungo. Se non lo sai, ti ritrovi con numeri corretti ma che rispondono a una domanda diversa da quella che credevi di aver posto.

## Nota sul rinominare i campi

Due livelli diversi:
- **Rinomina nel modello** (vista Dati o Modello) — cambia ovunque, è la scelta giusta per nomi sbagliati
- **Rinomina per questo oggetto visivo** (doppio clic nel riquadro Visualizzazioni) — locale al singolo visual

Il secondo serve quando lo stesso campo ha senso con nomi diversi in contesti diversi, o per togliere il prefisso "Somma di" senza creare una misura esplicita. Detto questo: se ti trovi spesso a togliere "Somma di", il problema vero è che stai usando **misure implicite**. Crea misure esplicite e il problema sparisce.
