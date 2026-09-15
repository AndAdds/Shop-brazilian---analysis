# Power BI — Lab alternativi "ShopBrasil"
## FILE 2 di 3 — SOLUZIONI

> Apri questo file **solo dopo** aver provato. Le soluzioni sono una delle possibili, non l'unica valida.

---

# LAB A1 — Soluzioni

### Task 2 — Connessione
**Perché cartella e non file singoli:** se domani arrivano nuovi CSV o i file cambiano nome/percorso, con la connessione a cartella aggiorni un solo passaggio. Con 9 connessioni separate ne aggiorni 9. In produzione la sorgente è quasi sempre una cartella o un DB, mai un file isolato.

**Perché escludere geolocation:** ~1 milione di righe per fornire lat/lng a livello di CAP. Il requisito di analisi arriva a livello Stato/Città. Importarla gonfia il modello senza aggiungere capacità analitica — viola il requisito non funzionale sulle prestazioni. Power BI geocodifica già Stato e Città dalla categoria dati.

**Modalità storage:** **Import**. I requisiti dicono esplicitamente "non serve real-time" e "minimizzare i tempi di caricamento". Import tiene i dati in VertiPaq compressi in memoria: risposte in millisecondi. DirectQuery servirebbe solo con requisito di freschezza o volumi che non stanno in memoria.

### Task 3 — Profilazione (risposte)
1. `order_delivered_customer_date` — vuota per gli ordini non ancora consegnati o annullati. È fisiologico: il dato non esiste ancora.
2. 8 stati: `delivered`, `shipped`, `canceled`, `unavailable`, `invoiced`, `processing`, `created`, `approved`.
3. 610 righe circa senza categoria.
4. **No**, `order_id` non è chiave in `order_items`: ~98.600 distinti su 112.650 righe. Un ordine con 3 prodotti genera 3 righe. La chiave è la coppia `order_id` + `order_item_id`.

### Task 4 — Power Query (codice M)

Filtro stato ordini:
```m
= Table.SelectRows(Origine, each [order_status] = "delivered")
```

Sostituzione categorie nulle:
```m
= Table.ReplaceValue(Origine, null, "sconosciuta", Replacer.ReplaceValue, {"product_category_name"})
```

Maiuscola su città:
```m
= Table.TransformColumns(Origine, {{"customer_city", Text.Proper, type text}})
```

Disabilitare il caricamento: tasto destro sulla query > togli la spunta da **Abilita caricamento**. La query resta disponibile per il merge ma non crea una tabella nel modello.

### Task 5 — Il tranello
Filtrare `orders` su `delivered` **non riduce** `order_items`. Sono due query indipendenti in Power Query: il filtro su una non si propaga all'altra. `order_items` mantiene tutte le 112.650 righe.

Due strade:
- **A (consigliata):** lascia `order_items` completa. La relazione nel modello farà sì che le righe orfane non compaiano nelle analisi filtrate per data. Ma occhio: i totali senza filtro data le includerebbero.
- **B (più pulita):** fai un merge di `order_items` con `orders` filtrata, join di tipo Inner, poi espandi nulla. Rimuove fisicamente le righe orfane. Costa un po' in refresh, guadagna in coerenza.

Se scegli A, aggiungi nella tabella misure un controllo:
```dax
Righe Orfane = 
COUNTROWS(
    FILTER( order_items, ISBLANK( RELATED( orders[order_id] ) ) )
)
```

---

# LAB A2 — Soluzioni

### Task 1 — Merge
**Tipo di join: Left Outer** (tutte le righe dalla prima tabella). Un Inner join perderebbe i ~610 prodotti senza traduzione, e con loro il fatturato associato — i totali non tornerebbero più con la sorgente.

```m
= Table.NestedJoin(products, {"product_category_name"}, translation, {"product_category_name"}, "trad", JoinKind.LeftOuter)
```

Pulizia nome categoria:
```m
= Table.TransformColumns(Origine, {{"Categoria", each Text.Proper(Text.Replace(_, "_", " ")), type text}})
```

### Task 3 — Relazioni

| Da | A | Cardinalità | Direzione filtro |
|---|---|---|---|
| order_items | orders | molti a uno | singola |
| order_items | products | molti a uno | singola |
| order_items | sellers | molti a uno | singola |
| orders | customers | molti a uno | singola |

**Tabella fatti:** `order_items` (è la granularità più fine: una riga = un prodotto in un ordine).
**Dimensioni:** products, sellers, customers.
**Ponte:** `orders` è ibrida — contiene date e stato (attributi dimensionali) ma sta tra fatti e customers.

**Forma dello schema: snowflake, non stella pura.** Il problema è la catena `order_items → orders → customers`: il cliente non è collegato direttamente ai fatti. Conseguenza pratica: il filtro dal cliente deve attraversare due relazioni per raggiungere i fatti. Funziona (la direzione è coerente), ma è un salto in più a ogni query.

**Come si normalizzerebbe in produzione:** si porterebbe `customer_state` e `customer_city` dentro `orders` con un merge in Power Query, eliminando la tabella customers. Oppure si costruirebbe una vera dimensione Cliente con chiave surrogata collegata direttamente ai fatti. Per l'esercizio va bene lo snowflake, ma devi sapere che è un compromesso.

### Task 4 — Proprietà colonne
- ID: **Riepiloga per = Non riepilogare**, poi nascondi. Se non lo fai, un doppio clic accidentale somma dei codici e produce numeri senza senso.
- `customer_state`: Categoria dati = **Provincia o Stato**
- `customer_city`: Categoria dati = **Città**
- `product_weight_g`: la somma dei pesi di tutti i prodotti non significa nulla (non stai pesando un carico, stai guardando un'anagrafica). La media sì.

### Task 6 — Se i numeri non cambiano
Se filtrando per categoria i valori per stato restano identici, il filtro non arriva. Controlla in ordine:
1. La relazione `order_items ↔ products` esiste ed è attiva (linea continua, non tratteggiata)?
2. La direzione del filtro va da products verso order_items?
3. Stai usando la colonna `Categoria` della tabella products o una copia orfana rimasta in un'altra tabella?

---

# LAB A3 — Soluzioni

### Task 1 — Colonne calcolate

**Chiave order_items:**
```dax
Chiave Riga = order_items[order_id] & "-" & order_items[order_item_id]
```

**Giorni Consegna** (le date stanno in `orders`, serve RELATED):
```dax
Giorni Consegna = 
VAR DataOrdine = RELATED( orders[Data Ordine] )
VAR DataConsegna = RELATED( orders[Data Consegna Cliente] )
RETURN
    IF(
        NOT ISBLANK( DataConsegna ),
        DATEDIFF( DataOrdine, DataConsegna, DAY )
    )
```

**Fascia Consegna:**
```dax
Fascia Consegna = 
VAR GG = order_items[Giorni Consegna]
RETURN
    SWITCH(
        TRUE(),
        ISBLANK( GG ),  "Non consegnato",
        GG <= 7,        "Veloce",
        GG <= 15,       "Standard",
                        "Lenta"
    )
```

`SWITCH(TRUE(), ...)` valuta in ordine e si ferma alla prima condizione vera. Più leggibile di IF annidati, e l'ordine ti evita di scrivere `GG >= 8 && GG <= 15`.

**Ritardo su Stima:**
```dax
Ritardo su Stima = 
DATEDIFF(
    RELATED( orders[Data Consegna Stimata] ),
    RELATED( orders[Data Consegna Cliente] ),
    DAY
)
```

**Focus — colonna o misura?**
Tutte e 4 sono correttamente **colonne calcolate**: sono attributi della singola riga, servono per raggruppare e filtrare, e non dipendono dal contesto di filtro del report. Una misura non può essere messa su un asse o in uno slicer.

**Quale andava in Power Query:** `Chiave Riga`. È una concatenazione statica che non usa il contesto del modello. Farla in Power Query significa che VertiPaq la comprime a monte anziché calcolarla al refresh in DAX. Regola generale: se puoi farla in Power Query, falla in Power Query.

### Task 2 — Tabella calendario

```dax
Calendario = 
VAR DataMin = MIN( orders[Data Ordine] )
VAR DataMax = MAX( orders[Data Ordine] )
VAR BaseCalendario = 
    CALENDAR( DATE( YEAR(DataMin), 1, 1 ), DATE( YEAR(DataMax), 12, 31 ) )
RETURN
    ADDCOLUMNS(
        BaseCalendario,
        "Anno",          YEAR( [Date] ),
        "Trimestre",     "Q" & QUARTER( [Date] ),
        "Mese Num",      MONTH( [Date] ),
        "Nome Mese",     FORMAT( [Date], "mmmm" ),
        "Anno-Mese",     FORMAT( [Date], "yyyy-mm" ),
        "Giorno Sett",   FORMAT( [Date], "dddd" ),
        "Giorno Sett Num", WEEKDAY( [Date], 2 )
    )
```

Nota: si parte dal 1° gennaio dell'anno minimo e si arriva al 31 dicembre dell'anno massimo, **non** dalla data esatta. Le funzioni di time intelligence richiedono anni civili completi, altrimenti YTD e confronti anno precedente danno risultati sbagliati sui bordi.

**Ordinamento Nome Mese:** seleziona la colonna `Nome Mese` > Strumenti colonna > **Ordina per colonna** > `Mese Num`. Senza questo passaggio i mesi appaiono in ordine alfabetico (Aprile, Agosto, Dicembre...).

**Marcare come tabella data:** tasto destro sulla tabella > **Contrassegna come tabella data** > scegli `Date`.
Cosa cambia: senza questa marcatura le funzioni di time intelligence usano una tabella data automatica nascosta (una per ogni campo data del modello), che gonfia il modello e ignora la tua tabella. Con la marcatura, Power BI sa che quella colonna è continua, senza buchi e senza duplicati, e ottimizza i calcoli.

**Disattiva anche la data automatica:** File > Opzioni > Caricamento dati > togli **Data/ora automatica**.

### Task 3 — Misure base

```dax
Fatturato = SUM( order_items[Importo] )

Costi Spedizione = SUM( order_items[Spedizione] )

Numero Transazioni = COUNTROWS( order_items )

Numero Ordini = DISTINCTCOUNT( order_items[order_id] )

Scontrino Medio = DIVIDE( [Fatturato], [Numero Ordini] )

Pezzi per Ordine = DIVIDE( [Numero Transazioni], [Numero Ordini] )
```

**Perché DIVIDE e non `/`:** DIVIDE gestisce la divisione per zero restituendo BLANK (o un terzo argomento a tua scelta) invece di un errore che rompe la visualizzazione.

**Tabella misure:** Home > Immetti dati > tabella vuota chiamata `_Misure`. Crea le misure lì dentro, poi elimina la colonna fittizia. L'underscore la fa ordinare in cima all'elenco campi.

### Task 4 — Margine

```dax
Costo Venduto = [Fatturato] * 0.62

Margine = [Fatturato] - [Costo Venduto] - [Costi Spedizione]

Margine % = DIVIDE( [Margine], [Fatturato] )
```

Formato: seleziona `Margine %` > Strumenti misura > formato **Percentuale**, 1 decimale.

**Nota metodologica:** un costo come percentuale fissa del prezzo è una semplificazione didattica. Nella realtà il costo varia per categoria e per fornitore, e starebbe in una colonna della dimensione prodotto. Se vuoi complicarti la vita in modo utile, crea una tabella di costi per categoria e usa `RELATED` o `LOOKUPVALUE`.

### Task 5 — Perché fatturato alto e margine basso non coincidono
Il margine assorbe i **costi di spedizione**, che sono un valore assoluto per riga, non proporzionale al prezzo. Una categoria con prodotti economici e pesanti (es. mobili, articoli per la casa) paga proporzionalmente molto di più in spedizione. Su un prodotto da 20 R$ con 15 R$ di spedizione il margine è negativo; su uno da 500 R$ con 20 R$ di spedizione è ampio.

---

# LAB A4 — Soluzioni

### Task 1 — Anno precedente

```dax
Fatturato AP = 
CALCULATE( [Fatturato], SAMEPERIODLASTYEAR( Calendario[Date] ) )

Transazioni AP = 
CALCULATE( [Numero Transazioni], SAMEPERIODLASTYEAR( Calendario[Date] ) )

Ordini AP = 
CALCULATE( [Numero Ordini], SAMEPERIODLASTYEAR( Calendario[Date] ) )

Var Fatturato = [Fatturato] - [Fatturato AP]

Var Fatturato % = DIVIDE( [Fatturato] - [Fatturato AP], [Fatturato AP] )
```

**Focus — due funzioni a confronto:**

```dax
Fatturato AP v2 = 
CALCULATE( [Fatturato], DATEADD( Calendario[Date], -1, YEAR ) )
```

`SAMEPERIODLASTYEAR(x)` è esattamente `DATEADD(x, -1, YEAR)`: stesso identico risultato, la prima è solo più leggibile.

La differenza vera è con **`PARALLELPERIOD`**:
```dax
Fatturato AP v3 = 
CALCULATE( [Fatturato], PARALLELPERIOD( Calendario[Date], -1, YEAR ) )
```
PARALLELPERIOD restituisce **l'intero periodo** (tutto l'anno precedente), non il periodo corrispondente. Se sei su marzo 2018, SAMEPERIODLASTYEAR ti dà marzo 2017, PARALLELPERIOD ti dà tutto il 2017.

Con un **mese incompleto** (es. sei a metà agosto 2018): SAMEPERIODLASTYEAR ti dà l'1-15 agosto 2017, quindi il confronto è corretto giorno per giorno. Questo è il comportamento che vuoi quasi sempre.

### Task 2 — Cumulati

```dax
Fatturato YTD = 
TOTALYTD( [Fatturato], Calendario[Date] )

Fatturato YTD AP = 
CALCULATE( [Fatturato YTD], SAMEPERIODLASTYEAR( Calendario[Date] ) )

Media Mobile 3M = 
AVERAGEX(
    DATESINPERIOD( Calendario[Date], MAX( Calendario[Date] ), -3, MONTH ),
    [Fatturato]
)
```

Attenzione alla media mobile: `AVERAGEX` su `DATESINPERIOD` a livello giorno farebbe la media dei giorni, non dei mesi. La versione sopra funziona se il contesto è mensile. Versione robusta:
```dax
Media Mobile 3M = 
VAR Periodo = DATESINPERIOD( Calendario[Date], MAX( Calendario[Date] ), -3, MONTH )
VAR Totale = CALCULATE( [Fatturato], Periodo )
RETURN DIVIDE( Totale, 3 )
```

### Task 3 — Override di contesto

```dax
Fatturato Totale Categoria = 
CALCULATE( [Fatturato], ALLEXCEPT( products, products[Categoria] ) )

Incidenza Prodotto % = 
DIVIDE( [Fatturato], [Fatturato Totale Categoria] )

Fatturato Totale Generale = 
CALCULATE( [Fatturato], ALL( products ) )

Incidenza Categoria % = 
DIVIDE( [Fatturato Totale Categoria], [Fatturato Totale Generale] )
```

**Focus — rimuovere vs sostituire:**

| Funzione | Cosa fa |
|---|---|
| `ALL(tabella)` | rimuove **tutti** i filtri dalla tabella |
| `ALL(colonna)` | rimuove i filtri solo da quella colonna |
| `ALLEXCEPT(tab, col)` | rimuove tutti i filtri **tranne** quelli sulle colonne indicate |
| `ALLSELECTED()` | rimuove i filtri interni alla visualizzazione ma **rispetta gli slicer** |
| `REMOVEFILTERS()` | sinonimo moderno di ALL usato come modificatore |
| `KEEPFILTERS()` | intersecа invece di sostituire |

**Perché in una matrice gerarchica sbagliare rompe i totali:** se usi `ALL(products)` per l'incidenza di prodotto, il denominatore diventa il fatturato totale di tutte le categorie anziché della sua. Le percentuali di prodotto sommeranno a 100% solo sul totale generale, non dentro ogni categoria — e la riga di subtotale mostrerà un numero che non corrisponde alla somma delle sue righe figlie.

**Variante importante:** se vuoi che l'incidenza rispetti lo slicer utente (es. l'utente ha selezionato 3 categorie e vuoi le percentuali su quelle 3), usa `ALLSELECTED` invece di `ALL`.

### Task 4 — Classifiche

```dax
Rank Categoria = 
IF(
    HASONEVALUE( products[Categoria] ),
    RANKX( ALL( products[Categoria] ), [Fatturato], , DESC, DENSE )
)

Top 5 Categorie = 
VAR Posizione = [Rank Categoria]
RETURN IF( Posizione <= 5, [Fatturato] )
```

- `HASONEVALUE` evita che il rank appaia sulla riga totale (dove non ha senso).
- `DENSE` gestisce i pari merito senza saltare posizioni: due primi posti, poi il secondo. Con `SKIP` avresti due primi e poi il terzo.

### Task 5 — Performance consegne

```dax
% Consegne Lente = 
DIVIDE(
    CALCULATE( [Numero Transazioni], order_items[Fascia Consegna] = "Lenta" ),
    CALCULATE( [Numero Transazioni], order_items[Fascia Consegna] <> "Non consegnato" )
)

Giorni Medi Consegna = 
AVERAGE( order_items[Giorni Consegna] )

Giorni Medi Consegna AP = 
CALCULATE( [Giorni Medi Consegna], SAMEPERIODLASTYEAR( Calendario[Date] ) )
```

Nota sul denominatore: escludere i "Non consegnato" è una scelta metodologica. Se li includi nel denominatore, la percentuale di consegne lente scende artificialmente. Documenta sempre questa scelta.

**Visualizzazione consigliata:** grafico combinato con mesi sull'asse, colonne = numero ordini, linea = giorni medi consegna, più una linea di riferimento sulla media del periodo.

---

# LAB A5 — Soluzioni

### Task 1 — Molti a molti

**Verifica nei dati:** crea una matrice temporanea con `seller_id` sulle righe e `DISTINCTCOUNT(customers[customer_state])` sui valori. Ordina decrescente.

**Struttura corretta — tabella ponte:**

```dax
Ponte Venditore Stato = 
SUMMARIZE(
    order_items,
    order_items[seller_id],
    customers[customer_state]
)
```

Poi:
1. Collega `Ponte[seller_id]` → `sellers[seller_id]` (molti a uno)
2. Collega `Ponte[customer_state]` → una dimensione Stato (creala con `DISTINCT(customers[customer_state])`)
3. Imposta la direzione del filtro incrociato su **Entrambe** solo sulle relazioni ponte-dimensione

**Incidenza venditore:**
```dax
Fatturato Stati del Venditore = 
CALCULATE(
    [Fatturato],
    REMOVEFILTERS( sellers ),
    VALUES( customers[customer_state] )
)

Incidenza Venditore % = 
DIVIDE( [Fatturato], [Fatturato Stati del Venditore] )
```

**Focus — perché non la relazione m:m diretta:**
Power BI la supporta dal 2018, ma internamente genera comunque una tabella ponte nascosta su cui non hai controllo. I problemi concreti:
1. I **totali non sono la somma delle righe** — sono ricalcolati sul contesto complessivo, e questo sorprende gli utenti
2. `RELATED` non funziona attraverso una m:m
3. Il filtraggio bidirezionale che richiede può creare **ambiguità di percorso** se il modello ha altri anelli
4. Le prestazioni degradano perché il motore deve risolvere la corrispondenza a runtime

Con la tabella ponte esplicita vedi cosa succede e puoi controllarlo.

### Task 2 — Card con variazione

Per la variazione colorata sulle card, la via più pulita è la **card nuova** (visual "Scheda nuova" da Power BI 2024) che supporta più valori. In alternativa, misura con formattazione condizionale:

```dax
Fatturato con Var = 
VAR Var = [Var Fatturato %]
VAR Freccia = IF( Var >= 0, "▲", "▼" )
RETURN
    FORMAT( [Fatturato], "#,##0,,\M" ) & " " & Freccia & " " & FORMAT( ABS(Var), "0.0%" )
```

Colore condizionale:
```dax
Colore Variazione = IF( [Var Fatturato %] >= 0, "#2E7D32", "#C62828" )
```
Applicalo in: Formato visual > Callout value > Colore > **fx** > Formato per = Valore campo > scegli `Colore Variazione`.

### Task 4 — Titoli dinamici

```dax
Titolo Pagina = 
"Performance vendite — " & 
IF( HASONEVALUE( Calendario[Anno] ), 
    SELECTEDVALUE( Calendario[Anno] ), 
    "tutti gli anni" )
```

```dax
Commento Andamento = 
VAR V = [Var Fatturato %]
VAR Anno = SELECTEDVALUE( Calendario[Anno] )
RETURN
    SWITCH(
        TRUE(),
        ISBLANK( [Fatturato AP] ), "Primo anno disponibile: nessun confronto possibile.",
        V > 0.05,  "Il " & Anno & " chiude in crescita del " & FORMAT(V, "0.0%") & " sul " & Anno-1 & ".",
        V < -0.05, "Il " & Anno & " chiude in calo del " & FORMAT(ABS(V), "0.0%") & " sul " & Anno-1 & ".",
        "Il " & Anno & " è sostanzialmente stabile rispetto al " & Anno-1 & "."
    )
```

Applicalo a una **casella di testo dinamica** (Inserisci > Casella di testo > pulsante **fx**) oppure al titolo di una card.

---

# LAB A6 — Soluzioni

### Task 2 — Visualizzazioni
1. **Combinato:** visual "Grafico a linee e istogramma a colonne raggruppate". Asse = `Calendario[Nome Mese]`, Valori colonna = `[Fatturato]`, Valori riga = `[Margine %]`.
2. **YTD:** grafico a linee, asse = Nome Mese, valori = `[Fatturato YTD]` e `[Fatturato YTD AP]`.
3. **Matrice:** righe = gerarchia Calendario, colonne = `order_items[Fascia Consegna]`, valori = `[Numero Ordini]`.
4. **Variazione mese su mese:**
```dax
Var Fatturato MoM = 
VAR Precedente = CALCULATE( [Fatturato], DATEADD( Calendario[Date], -1, MONTH ) )
RETURN [Fatturato] - Precedente
```
Usa il visual **Grafico a cascata** con `Var Fatturato MoM` come valore.

### Task 3 — Formattazione condizionale
- Scala colore: Formato visual > Elementi della cella > seleziona la misura > **Colore sfondo** > fx > Formato per = **Scala colori**.
- Icone: stessa sezione, **Icone** > fx > Formato per = **Regole**. Regola: se valore < `[Fatturato Medio Mensile]` mostra icona rossa.

```dax
Fatturato Medio Mensile = 
AVERAGEX( VALUES( Calendario[Anno-Mese] ), [Fatturato] )
```

- Linea di riferimento: riquadro **Analisi** (icona lente) > Linea media.

### Task 4 — Analisi assistita
- **Previsione:** riquadro Analisi > Previsione. Disponibile solo su grafico a linee con asse continuo di tipo data. Se non appare, il tuo asse è categorico: usa `Calendario[Date]` invece di `Nome Mese`.
- **Individuazione anomalie:** riquadro Analisi > Trova anomalie. Stessa condizione sull'asse.
- **Spiega l'aumento:** tasto destro su un punto del grafico > **Analizza** > Spiega la diminuzione. Power BI cerca automaticamente quali dimensioni spiegano lo scostamento. Nel dataset Olist tipicamente trovi che il calo di fine 2018 è dovuto al **troncamento dei dati** (la raccolta si ferma a ottobre 2018, non è un calo reale). Questo è un ottimo esempio da citare: lo strumento trova il pattern ma l'interpretazione resta tua.

### Task 5 — Sincronizzare gli slicer
Seleziona lo slicer > menu **Visualizza** > **Sincronizza slicer** > spunta le pagine su cui sincronizzare. Puoi scegliere se sincronizzare solo il filtro o anche la visibilità.

---

# LAB A7 — Soluzioni

### Task 2 — Segnalibri

Procedura:
1. Crea tabella e matrice, sovrapponile
2. Apri **Visualizza > Riquadro selezione** per gestire la visibilità dei singoli oggetti
3. Nascondi la matrice, apri **Visualizza > Segnalibri** > Aggiungi > rinomina "Vista Tabella"
4. Mostra la matrice, nascondi la tabella > Aggiungi segnalibro > "Vista Matrice"
5. **Cruciale:** su ogni segnalibro, tasto destro > togli la spunta da **Dati** e da **Filtri**, lascia solo **Visualizzazione corrente** e **Elementi visibili**

Senza il punto 5 il segnalibro salva anche lo stato degli slicer e, quando l'utente clicca il bottone, gli resetta le selezioni. È l'errore più comune e più fastidioso.

6. Inserisci due bottoni (Inserisci > Pulsanti > Vuoto), imposta testo e azione = **Segnalibro**
7. Per lo stato selezionato: Formato pulsante > **Stile** > applica un colore di sfondo diverso allo stato "Premuto"

### Task 3 — Drill-through
1. Nella pagina Dettaglio, riquadro **Filtri** > area **Drill-through** > trascina `products[Categoria]`
2. Power BI aggiunge automaticamente il pulsante di ritorno (freccia in alto a sinistra)
3. Titolo dinamico:
```dax
Titolo Dettaglio = 
"Dettaglio transazioni — " & SELECTEDVALUE( products[Categoria], "Tutte le categorie" )
```

### Task 4 — Performance Analyzer
1. Scheda **Visualizza** > Performance Analyzer > Avvia registrazione > Aggiorna oggetti visivi
2. Leggi la colonna **Query DAX** in millisecondi
3. Tipicamente il collo di bottiglia è la matrice con molte righe o una misura con iteratori annidati

**Ottimizzazioni più frequenti, in ordine di resa:**
1. Sostituisci `FILTER(tabella, ...)` con un predicato diretto in `CALCULATE` — il primo materializza tutta la tabella, il secondo usa l'ottimizzazione interna
   - Lento: `CALCULATE([Fatturato], FILTER(order_items, order_items[Fascia Consegna]="Lenta"))`
   - Veloce: `CALCULATE([Fatturato], order_items[Fascia Consegna]="Lenta")`
2. Rimuovi le colonne inutilizzate dal modello (ogni colonna occupa memoria anche se nascosta)
3. Evita `DISTINCTCOUNT` su colonne ad alta cardinalità quando puoi usare `COUNTROWS(VALUES(...))`
4. Riduci le relazioni bidirezionali
5. Non usare colonne calcolate DAX dove basta Power Query

Documenta: "La matrice X impiegava 1.240 ms, dopo la riscrittura della misura Y impiega 310 ms."

### Task 5 — Tre evidenze tipiche dal dataset Olist
Spunti se ti blocchi (verificali tu, non copiarli):
1. **Concentrazione geografica estrema** — San Paolo da solo vale una quota enorme del fatturato; la strategia logistica dovrebbe riflettere questo squilibrio
2. **La spedizione erode il margine sui prodotti economici** — esistono categorie dove il costo di spedizione supera il margine lordo
3. **Il crollo di fine 2018 non è un crollo** — è la fine della finestra di raccolta dati. Confondere un artefatto del dataset con un fenomeno di business è l'errore classico, e saperlo riconoscere vale più di qualsiasi misura DAX

---

## Errori comuni, riassunti

| Sintomo | Causa quasi sempre |
|---|---|
| Totali di percentuale che non fanno 100% | `ALL` invece di `ALLEXCEPT` o `ALLSELECTED` |
| Time intelligence restituisce vuoto | tabella calendario non marcata come tabella data, o anni non completi |
| Mesi in ordine alfabetico | manca "Ordina per colonna" |
| Il filtro non si propaga | direzione relazione, o relazione inattiva |
| Report lentissimo | `FILTER` dentro `CALCULATE`, o geolocation importata |
| I bottoni resettano gli slicer | segnalibro salvato con "Dati" spuntato |
| Numeri diversi dalla sorgente | Inner join dove serviva Left |

---

# LAB A8 — Soluzioni

### Task 1 — Albero di scomposizione
Inserisci > Albero di scomposizione. Campo **Analisi** = `[Fatturato]`, campo **Spiega per** = le dimensioni.

**Manuale vs IA:**
- **Valore alto / Valore basso** sono deterministici: scelgono il ramo col valore massimo/minimo. Prevedibili e riproducibili.
- **Split ad alta variabilità (IA)** usa un algoritmo che cerca la dimensione che *spiega meglio* la varianza. Può sorprenderti positivamente ma non è riproducibile e non spiega il criterio.

**Quando fidarsi:** l'IA in fase esplorativa, per generare ipotesi. Il manuale in fase di presentazione, perché devi poter difendere il percorso davanti a chi ti fa domande.

Nel dataset Olist tipicamente il primo split rivelatore è sullo Stato: San Paolo domina in modo che nessun'altra dimensione replica.

### Task 2 — Piccoli multipli
- Rinomina nel visual: doppio clic sul campo nel riquadro Visualizzazioni > **Rinomina per questo oggetto visivo**. Non tocca il modello.
- Formato > **Piccoli multipli** > Layout > Righe 3, Colonne 1.
- Ombreggiatura: Formato > Generale > **Effetti** > Ombreggiatura.

**Limite pratico:** oltre 9-12 pannelli diventa illeggibile e compare lo scroll, che uccide il confronto visivo (non vedi più tutti i pannelli insieme, che è l'unico motivo per cui usi i piccoli multipli). Con 71 categorie in Olist **devi** filtrare: usa un filtro Top N sulle prime 6 categorie per fatturato.

```dax
Rank per Piccoli Multipli = 
RANKX( ALL( products[Categoria] ), [Fatturato], , DESC )
```
Poi filtro a livello di oggetto visivo: `Rank per Piccoli Multipli <= 6`.

### Task 3 — Misuratore

```dax
Obiettivo Fatturato = [Fatturato AP] * 1.1

Massimo Misuratore = 
VAR Valore = [Fatturato]
VAR Target = [Obiettivo Fatturato]
RETURN MAX( Valore, Target ) * 1.2
```

Logica: il massimo deve essere poco sopra il maggiore tra valore e obiettivo. Se lasci il default (doppio del valore), quando superi l'obiettivo l'ago resta schiacciato a sinistra e non si legge nulla.

Impostazione: trascina `Massimo Misuratore` nel campo **Valore massimo** del visual.

### Task 4 — Parametro what-if

1. Modellazione > **Nuovo parametro** > Intervallo numerico
   - Nome: `Incremento Target`
   - Min 0, Max 0.5, Incremento 0.05
   - Spunta "Aggiungi filtro dei dati a questa pagina"

Power BI crea automaticamente una tabella e una misura:
```dax
Valore Incremento Target = SELECTEDVALUE( 'Incremento Target'[Incremento Target], 0.1 )
```

2. Riscrivi l'obiettivo:
```dax
Obiettivo Fatturato = [Fatturato AP] * ( 1 + [Valore Incremento Target] )
```

Il secondo argomento di `SELECTEDVALUE` è il default quando lo slicer non ha selezione — mettilo sempre, altrimenti l'obiettivo diventa vuoto e il misuratore sparisce.

### Task 5 — Molti a molti con relazione inattiva

Questo è l'approccio del corso. Passaggi:

1. **Relazione ponte → Stato bidirezionale.** Serve perché il filtro deve risalire dal ponte alla dimensione Stato e poi ridiscendere ai fatti. Con direzione singola il percorso si interrompe.

2. **Relazione venditore → fatti inattiva.** Dopo il punto 1 esistono due percorsi da venditore a fatti: quello diretto e quello via ponte/stato. Power BI risolve l'ambiguità col **principio del cammino minimo**, cioè sceglie sempre il diretto — che non è quello che vuoi per l'analisi sulle regioni. Disattivando il diretto forzi il percorso lungo.

3. **Misure che riattivano la relazione al bisogno:**
```dax
Fatturato Venditore = 
CALCULATE(
    [Fatturato],
    USERELATIONSHIP( order_items[seller_id], sellers[seller_id] )
)

Transazioni Venditore = 
CALCULATE(
    [Numero Transazioni],
    USERELATIONSHIP( order_items[seller_id], sellers[seller_id] )
)

Incidenza Venditore % = 
DIVIDE( [Fatturato Venditore], [Fatturato] )
```

La chiave concettuale: `[Fatturato Venditore]` usa il percorso **diretto** (le vendite di quell'agente), `[Fatturato]` al denominatore usa il percorso **via ponte** (il totale delle regioni in cui opera). Il rapporto è esattamente l'incidenza richiesta.

**Confronto tra i due approcci:**

| | Ponte esplicito (A5) | Relazione inattiva (A8) |
|---|---|---|
| Leggibilità del modello | alta, vedi tutto | media, devi leggere le misure |
| Bidirezionali necessarie | 1 | 1 |
| Complessità DAX | media (REMOVEFILTERS/VALUES) | bassa (USERELATIONSHIP) |
| Rischio di ambiguità | basso | alto se aggiungi tabelle |
| Manutenibilità | migliore | peggiore |

**Cosa spiegare a un collega:** il ponte. È autoesplicativo — apri la vista Modello e capisci la struttura. Con la relazione inattiva devi aprire ogni misura per capire cosa succede, e chi eredita il report dopo di te non lo capirà.

**Cosa aspettarsi all'esame:** il corso insegna il secondo. Impara entrambi, presenta quello del corso, cita l'altro come alternativa — quella è la risposta che dimostra di aver capito e non solo copiato.

### Task 6 — Etichette per serie
1. Formato > **Etichette dati** > in alto seleziona **Applica impostazioni a: Serie** > scegli la serie anno precedente > interruttore Off
2. Seleziona la serie anno corrente > apri **Personalizza serie** > campo **Dettaglio etichetta** > trascina `[Var Fatturato %]`
3. Colore condizionale sull'etichetta: fx > Formato per = Valore campo > usa la misura `Colore Variazione` già creata al Lab A5
