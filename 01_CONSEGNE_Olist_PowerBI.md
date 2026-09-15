# Power BI — Lab alternativi "ShopBrasil"
## FILE 1 di 3 — CONSEGNE

> Questo file contiene **solo i task**. Le soluzioni sono nel file 2, la teoria nel file 3.
> Non aprire il file 2 prima di aver provato.

---

## Setup iniziale (una volta sola — 15 minuti)

**Dataset:** Brazilian E-Commerce Public Dataset by Olist
**Download:** https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce

1. Scarica lo ZIP da Kaggle (serve account gratuito)
2. Estrai in una cartella dedicata, es. `C:\PowerBI\ShopBrasil\`
3. Verifica di avere questi 9 CSV:

| File | Righe ~ | Cosa contiene |
|---|---|---|
| `olist_orders_dataset.csv` | 99.441 | ordini, stato, 5 timestamp |
| `olist_order_items_dataset.csv` | 112.650 | riga d'ordine: prodotto, venditore, prezzo, spedizione |
| `olist_products_dataset.csv` | 32.951 | anagrafica prodotti |
| `olist_customers_dataset.csv` | 99.441 | clienti + CAP, città, stato |
| `olist_sellers_dataset.csv` | 3.095 | venditori + CAP, città, stato |
| `olist_order_payments_dataset.csv` | 103.886 | pagamenti, metodo, rate |
| `olist_order_reviews_dataset.csv` | 99.224 | recensioni 1-5 stelle |
| `product_category_name_translation.csv` | 71 | categorie PT → EN |
| `olist_geolocation_dataset.csv` | ~1M | CAP → lat/lng |

---

## Scenario (vale per tutti i lab)

**ShopBrasil** è un marketplace che mette in contatto piccoli venditori con clienti in tutto il Brasile. Il cliente ordina sulla piattaforma, il venditore spedisce, il cliente riceve e lascia una recensione.

La direzione vuole una soluzione di BI per analizzare lo storico e monitorare l'andamento corrente. Le metriche di interesse sono:

- **Fatturato** (importo totale delle righe d'ordine)
- **Numero Ordini** (ordini distinti)
- **Numero Transazioni** (righe d'ordine)
- **Costi di Spedizione**
- **Margine** (vedi Lab 3 per la definizione)

Le analisi devono essere esplorabili per:
- **gerarchia prodotto** (Categoria → Prodotto)
- **gerarchia geografica cliente** (Stato → Città)
- **venditore**
- **tempo** (anno, trimestre, mese)

**Requisiti non funzionali**
- Non serve un report real-time.
- Minimizzare il tempo di caricamento delle visualizzazioni.
- Il modello deve reggere l'aggiunta di nuovi mesi di dati senza rifare le relazioni.

---

# LAB A1 — ETL: acquisizione e pulizia
### Equivalente di: Lab 1 BikesWorld | Tempo stimato: 60-75 minuti

**Scopo:** acquisizione delle tabelle utili, profilazione, pulizia, ristrutturazione.

### Task 1 — Setup del report
1. Crea la cartella dove salverai i report
2. Apri Power BI Desktop, salva il file come `ShopBrasil - Analisi Vendite.pbix`
3. In **File > Opzioni e impostazioni > Opzioni > Caricamento dati**, disattiva il rilevamento automatico delle relazioni (le creerai tu, consapevolmente)
4. Sempre nelle opzioni, attiva la **profilazione colonna sull'intero set di dati** (default: solo prime 1000 righe)

### Task 2 — Connessione dati
1. Connettiti alla **cartella** contenente i CSV (non ai singoli file uno per uno). Motiva in un commento perché questa modalità è preferibile.
2. Importa queste 5 tabelle, che sono quelle utili al perimetro di analisi:
   - orders
   - order_items
   - products
   - customers
   - product_category_name_translation
3. **Non importare** geolocation. Scrivi in una nota perché (indizio: guarda il numero di righe e i requisiti non funzionali).
4. Scegli la modalità di storage corretta dati i requisiti. Motiva.

### Task 3 — Profilazione
Per ogni tabella importata, in Power Query attiva **Qualità colonna**, **Distribuzione colonna**, **Profilo colonna** e rispondi:
1. Quale colonna di `orders` ha la percentuale più alta di valori vuoti? Perché è normale che lo sia?
2. Quanti valori distinti ha `order_status`? Quali stati esistono?
3. In `products`, quante righe hanno la categoria mancante?
4. In `order_items`, il campo `order_id` è una chiave? Verifica contando distinti vs totali e spiega il risultato.

### Task 4 — Pulizia e trasformazione
1. In `orders`:
   - Tieni **solo** gli ordini con stato `delivered`. Motiva la scelta rispetto all'obiettivo di analisi vendite.
   - Imposta i tipi dati corretti sui 5 campi data/ora.
   - Rimuovi `order_approved_at` e `order_delivered_carrier_date` (non servono al perimetro).
   - Rinomina le colonne in italiano leggibile (es. `order_purchase_timestamp` → `Data Ordine`).
2. In `products`:
   - Rimuovi tutte le colonne relative a dimensioni e peso **tranne** `product_weight_g`.
   - Sostituisci le categorie nulle con `"sconosciuta"`.
3. In `order_items`:
   - Rimuovi `shipping_limit_date`.
   - Rinomina `price` → `Importo` e `freight_value` → `Spedizione`.
4. In `customers`:
   - Rimuovi `customer_zip_code_prefix`.
   - Metti in maiuscolo la prima lettera di `customer_city`.
5. Disabilita il caricamento della query `product_category_name_translation` (la userai in merge al Lab A2, non deve finire nel modello come tabella a sé).

### Task 5 — Verifica
1. Applica e chiudi. Quante righe ha `order_items` dopo il filtro sugli ordini `delivered`? (Attenzione: il filtro è su `orders` — questo è un tranello, rifletti su cosa succede davvero.)
2. Salva il report.

**Consegna:** file .pbix + un file di testo con le risposte ai Task 3 e 5.

---

# LAB A2 — ETL e modellazione dati
### Equivalente di: Lab 2 BikesWorld | Tempo stimato: 60-75 minuti

**Scopo:** arricchimento tabelle, relazioni, proprietà di colonne e tabelle, gerarchie.

### Task 1 — Arricchimento tabella prodotti
1. Riapri Power Query. Fai un **merge** tra `products` e `product_category_name_translation` per portare la categoria in inglese dentro `products`.
2. Usa il tipo di join corretto per **non perdere prodotti** privi di traduzione. Motiva la scelta.
3. Espandi solo la colonna della categoria inglese, rinominala `Categoria`.
4. Rimuovi la colonna della categoria originale in portoghese.
5. Metti in maiuscolo la prima lettera di ogni parola in `Categoria` e sostituisci gli underscore con spazi.

### Task 2 — Tabella venditori
1. Importa `olist_sellers_dataset.csv`.
2. Rimuovi `seller_zip_code_prefix`, rinomina i campi.
3. Applica e chiudi.

### Task 3 — Relazioni
Costruisci il modello collegando:
- `order_items` ↔ `orders`
- `order_items` ↔ `products`
- `order_items` ↔ `sellers`
- `orders` ↔ `customers`

Per **ogni** relazione documenta:
1. Cardinalità (1:molti, molti:1, 1:1, molti:molti)
2. Direzione del filtro incrociato
3. Qual è la tabella fatti e quali le dimensioni

Domanda: che forma ha il tuo schema? È una stella pura? Se no, perché e quale tabella crea il problema?

### Task 4 — Proprietà colonne
1. Su tutti gli ID (order_id, product_id, customer_id, seller_id): imposta **Riepiloga per: Non riepilogare** e nascondili dalla vista report.
2. Su `Importo` e `Spedizione`: imposta il formato valuta e 2 decimali.
3. Su `customer_state`: imposta la **categoria dati** appropriata per abilitare le mappe.
4. Su `customer_city`: idem.
5. Su `product_weight_g`: riepiloga per Media anziché Somma. Motiva.

### Task 5 — Gerarchie
1. Crea la gerarchia **Prodotto**: Categoria → Nome Prodotto (usa product_id come livello finale se non hai un nome).
2. Crea la gerarchia **Geografia Cliente**: Stato → Città.
3. Nascondi dalla vista report le colonne singole già incluse nelle gerarchie.

### Task 6 — Verifica del modello
1. Crea una matrice con la gerarchia Prodotto sulle righe e la somma di `Importo` sui valori. Funziona il drill-down?
2. Crea una tabella con `customer_state` e la somma di `Importo`. I numeri cambiano quando filtri per categoria? Se non cambiano, hai un problema di relazioni: trovalo.

**Consegna:** file .pbix + screenshot della vista Modello + documentazione delle relazioni.

---

# LAB A3 — Data Model e DAX
### Equivalente di: Lab 3 BikesWorld | Tempo stimato: 75-90 minuti

**Scopo:** colonne calcolate, tabella calendario, misure base.

### Task 1 — Colonne calcolate
1. Crea una **colonna chiave** nella tabella `order_items`. Individua da solo quali campi servono (indizio: `order_id` da solo non basta, verificalo).
2. Crea una colonna `Giorni Consegna` = giorni tra data ordine e data di effettiva consegna al cliente.
   - Attenzione: le due date stanno in tabelle diverse. Risolvi.
3. Crea una colonna `Fascia Consegna` che classifichi:
   - `<= 7 giorni` → **"Veloce"**
   - `da 8 a 15 giorni` → **"Standard"**
   - `> 15 giorni` → **"Lenta"**
   - date di consegna mancanti → **"Non consegnato"**
4. Crea una colonna `Ritardo su Stima` = differenza in giorni tra data consegna effettiva e data consegna stimata. Valori negativi = consegnato in anticipo.

**Focus:** per ognuna delle 4, chiediti se è davvero una colonna calcolata o dovrebbe essere una misura. Scrivi la motivazione. Una di queste sarebbe meglio farla in Power Query: quale e perché?

### Task 2 — Tabella calendario
1. Crea una tabella calendario in DAX che copra **esattamente** il range di date presenti negli ordini (non hardcodare gli anni).
2. Aggiungi le colonne: Anno, Trimestre, Mese (numero), Nome Mese, Anno-Mese (ordinabile), Giorno Settimana.
3. Ordina `Nome Mese` per numero mese.
4. Marca la tabella come **tabella data**. Spiega cosa cambia se non lo fai.
5. Collega la calendario a `orders` sulla data ordine.
6. Crea la gerarchia temporale Anno → Trimestre → Mese.

### Task 3 — Misure base
Crea una tabella misure dedicata (tabella vuota, senza colonne) e dentro:

1. `Fatturato` = somma degli importi
2. `Costi Spedizione` = somma delle spedizioni
3. `Numero Transazioni` = conteggio righe d'ordine
4. `Numero Ordini` = conteggio ordini **distinti**
5. `Scontrino Medio` = fatturato / numero ordini
6. `Pezzi per Ordine` = transazioni / ordini

### Task 4 — Margine
ShopBrasil non espone il costo d'acquisto. Per l'esercizio, assumiamo che il **costo del venduto sia il 62% dell'importo**.

1. Crea la misura `Costo Venduto`
2. Crea la misura `Margine` = fatturato − costo venduto − costi di spedizione
3. Crea la misura `Margine %` = margine / fatturato
4. Formatta `Margine %` come percentuale con 1 decimale
5. Gestisci la divisione per zero correttamente

### Task 5 — Verifica
Costruisci una matrice: gerarchia Prodotto sulle righe, tutte le misure sui valori. Individua:
1. La categoria con il fatturato più alto
2. La categoria con il margine % più basso
3. Perché le due non coincidono

**Consegna:** file .pbix + file di testo con le motivazioni del Focus Task 1 e le risposte al Task 5.

---

# LAB A4 — DAX e override di contesto
### Equivalente di: Lab 4 BikesWorld | Tempo stimato: 90 minuti

**Scopo:** time intelligence, funzioni di modifica del contesto di filtro.

### Task 1 — Confronto anno precedente
1. Crea le misure per il fatturato **nello stesso periodo dell'anno precedente**.
2. Fai lo stesso per numero transazioni e numero ordini.
3. Per ognuna, crea la misura di **variazione assoluta** e di **variazione percentuale** rispetto all'anno precedente.
4. Costruisci un grafico a linee: mesi sull'asse, anno corrente e anno precedente come due serie.

**Focus:** usa almeno due funzioni di time intelligence diverse per ottenere lo stesso risultato, poi confrontale. Quando danno risultati diversi? (indizio: prova a filtrare un singolo giorno, e prova con un mese incompleto)

### Task 2 — Cumulati
1. Crea la misura `Fatturato YTD` (da inizio anno alla data corrente del contesto).
2. Crea `Fatturato YTD Anno Precedente`.
3. Crea una misura di **media mobile a 3 mesi** del fatturato.

### Task 3 — Override di contesto
1. Crea `Fatturato Totale Categoria`: il fatturato dell'intera categoria, ignorando eventuali filtri sul singolo prodotto.
2. Crea `Incidenza Prodotto %`: quanto pesa il prodotto sulla sua categoria.
3. Crea `Incidenza Categoria %`: quanto pesa la categoria sul totale generale.
4. Verifica che le incidenze sommino a 100% a ogni livello della gerarchia.

**Focus:** quali funzioni rimuovono filtri e quali li sostituiscono? Perché in una matrice gerarchica la scelta sbagliata dà totali che non tornano?

### Task 4 — Classifiche
1. Crea una misura che dia il **rank** delle categorie per fatturato.
2. Crea una misura `Top 5 Categorie`: mostra il fatturato solo per le prime 5 categorie, altrimenti vuoto.
3. Gestisci il caso dei pari merito.

### Task 5 — Analisi performance consegne
1. Crea la misura `% Consegne Lente`.
2. Crea la misura `Giorni Medi Consegna`.
3. Costruisci una visualizzazione che risponda: **la performance di consegna è peggiorata o migliorata anno su anno?**

**Consegna:** file .pbix + risposta scritta ai due Focus.

---

# LAB A5 — Storytelling: pagina Overview
### Equivalente di: Lab 7 BikesWorld | Tempo stimato: 90 minuti

**Scopo:** costruzione report, relazioni molti-a-molti, incidenza.

### Task 1 — Lo scenario molti-a-molti
ShopBrasil vuole analizzare la performance dei **venditori rispetto agli stati di destinazione**.

Situazione reale nei dati: un venditore spedisce in più stati, e lo stesso stato è servito da più venditori.

1. Verifica nei dati che sia davvero così: trova un venditore che spedisce in almeno 5 stati e uno stato servito da almeno 50 venditori.
2. Costruisci nel modello la struttura corretta per gestire questa relazione. Non forzare una relazione diretta tra `sellers` e `customers`.
3. Crea la misura `Incidenza Venditore %`: quanto pesa un venditore sul totale realizzato negli stati in cui opera (non sul totale generale).

**Focus:** perché una relazione molti-a-molti diretta è sconsigliata? Cosa succede ai totali?

### Task 2 — Pagina Overview
Costruisci una pagina `Overview` con:

1. Sfondo e tema coerente (imposta un tema personalizzato, non usare il default)
2. Una riga di **KPI card** in alto: Fatturato, Numero Ordini, Scontrino Medio, Margine %
3. Ogni card deve mostrare anche la **variazione % vs anno precedente** con colore condizionale (verde/rosso)
4. Un grafico a linee: andamento fatturato mensile con confronto anno precedente
5. Un grafico a barre: Top 10 categorie per fatturato
6. Una mappa: fatturato per stato cliente
7. Uno slicer anno con selezione singola
8. Uno slicer categoria

### Task 3 — Interazioni
1. Configura le **interazioni tra oggetti**: cliccando sulla mappa, i grafici devono filtrare; cliccando sulle card, no.
2. Aggiungi **tooltip personalizzati** su almeno un grafico (pagina tooltip dedicata, non quelli di default).
3. Imposta l'ordine di tabulazione per l'accessibilità.

### Task 4 — Titoli dinamici
1. Il titolo della pagina deve cambiare in base allo slicer anno, es. "Performance vendite — 2017".
2. Aggiungi un testo dinamico che dica, a parole, se l'anno selezionato è andato meglio o peggio del precedente e di quanto.

**Consegna:** file .pbix + screenshot della pagina + risposta al Focus.

---

# LAB A6 — Storytelling: pagina Time Analysis
### Equivalente di: Lab 8 BikesWorld | Tempo stimato: 75 minuti

**Scopo:** pagina di analisi temporale, slicer avanzati.

### Task 1 — Struttura pagina
1. Crea una nuova pagina `Time Analysis`
2. Inserisci uno slicer con l'anno, **selezione singola obbligatoria**
3. Inserisci uno slicer categoria a discesa con ricerca

### Task 2 — Visualizzazioni
1. Grafico combinato: colonne = fatturato mensile, linea = margine %
2. Grafico a linee: fatturato YTD vs fatturato YTD anno precedente
3. Matrice: righe = gerarchia temporale (Anno → Trimestre → Mese), colonne = fascia consegna, valori = numero ordini
4. Grafico a nastri o waterfall: variazione di fatturato mese su mese

### Task 3 — Formattazione condizionale
1. Nella matrice, applica una scala colore sul numero ordini
2. Aggiungi icone che segnalino i mesi sotto la media annuale
3. Applica una linea di riferimento (media) sul grafico combinato

### Task 4 — Analisi assistita
1. Usa la funzionalità di **individuazione anomalie** o **previsione** sul grafico a linee del fatturato
2. Usa **Analizza > Spiega l'aumento/la diminuzione** su un mese con forte variazione. Riporta cosa ha trovato.
3. Aggiungi una **visualizzazione Domande e risposte** e testala con 3 domande in linguaggio naturale

### Task 5 — Navigazione
1. Aggiungi un pulsante per tornare alla pagina Overview
2. Configura il pulsante con azione di navigazione pagina
3. Sincronizza gli slicer tra le due pagine

**Consegna:** file .pbix + screenshot + output del punto Task 4.2.

---

# LAB A7 — Storytelling: pagina Dettaglio
### Equivalente di: Lab 9 BikesWorld | Tempo stimato: 90 minuti

**Scopo:** pagina di dettaglio, segnalibri, drill-through.

### Task 1 — Pagina di dettaglio
Crea una pagina `Dettaglio Transazioni` che consenta all'utente di esplorare, per ogni mese, le singole transazioni.

La **tabella** deve esporre:
- Data ordine
- Data consegna
- Fascia consegna
- Categoria
- Stato cliente
- Venditore
- Importo
- Spedizione
- Margine

### Task 2 — Alternativa tabella/matrice
L'utente vuole poter scegliere, con un **bottone**, se vedere i dati in formato tabella o in una matrice.

1. Crea la matrice equivalente: righe = categoria, colonne = fascia consegna, valori = importo e numero transazioni
2. Sovrapponi tabella e matrice nella stessa area
3. Crea due **segnalibri** che mostrino l'una e nascondano l'altra
4. Collega i segnalibri a due bottoni con stato selezionato/non selezionato visibile
5. Attenzione: i segnalibri non devono resettare gli slicer dell'utente. Configura le proprietà di conseguenza.

### Task 3 — Drill-through
1. Configura la pagina Dettaglio come **pagina di drill-through** filtrata per categoria
2. Dalla pagina Overview, cliccando col destro su una categoria si deve poter passare al dettaglio già filtrato
3. Aggiungi il pulsante di ritorno automatico
4. Aggiungi un titolo dinamico che mostri la categoria selezionata

### Task 4 — Rifiniture
1. Nascondi le pagine di servizio (tooltip) dalla barra pagine
2. Verifica che il report funzioni in modalità schermo intero
3. Controlla i tempi di risposta: usa **Performance Analyzer** e individua la visualizzazione più lenta
4. Ottimizza almeno una misura lenta e documenta il miglioramento in millisecondi

### Task 5 — Consegna finale
Esporta il report in PDF e scrivi mezza pagina di commento: quali sono le 3 evidenze principali che emergono dai dati di ShopBrasil?

**Consegna:** file .pbix + PDF + commento + output Performance Analyzer.

---

## Checklist di autovalutazione

Prima di considerare finito un lab, verifica:

- [ ] I totali nelle matrici tornano a ogni livello di gerarchia
- [ ] Nessuna colonna tecnica (ID) è visibile nella vista report
- [ ] Tutte le misure hanno un formato numerico esplicito
- [ ] Le divisioni gestiscono lo zero
- [ ] Il modello non ha relazioni bidirezionali non necessarie
- [ ] Il report si apre e si aggiorna in meno di 10 secondi

---

# LAB A8 — Visual avanzati e parametri
### Equivalente di: parti specifiche dei Lab 5, 8, 9 | Tempo stimato: 75 minuti

**Scopo:** albero di scomposizione, piccoli multipli, misuratore con soglia dinamica, relazione inattiva.

### Task 1 — Albero di scomposizione
1. Inserisci un **Albero di scomposizione** con `[Fatturato]` come Analisi
2. Aggiungi come dimensioni esplodibili: Anno, Categoria, Stato Cliente, Fascia Consegna
3. Espandi usando **Valore alto** e trova il percorso che concentra più fatturato
4. Prova poi **Valore basso** e il percorso automatico con IA
5. Scrivi cosa cambia tra esplosione manuale e assistita da IA, e quando ti fideresti dell'una o dell'altra

### Task 2 — Piccoli multipli
1. Crea un istogramma a colonne raggruppate: asse = Nome Mese, valori = `[Numero Transazioni]`
2. Rinomina il campo visualizzato da "Somma di..." a `Transazioni` (rinomina nel visual, non nel modello)
3. Aggiungi `products[Categoria]` nel campo **Piccoli multipli**
4. In Formato > Piccoli multipli > Layout, imposta 1 colonna e 3 righe
5. Aggiungi l'effetto **Ombreggiatura** al visual
6. Rifletti: a quante categorie smette di funzionare questa visualizzazione? Qual è il limite pratico?

### Task 3 — Misuratore con obiettivo dinamico
1. Crea una misura obiettivo: il fatturato dell'anno precedente maggiorato del 10%
2. Inserisci un **Misuratore**: valore = `[Fatturato]`, valore obiettivo = la misura appena creata
3. Problema: il massimo di default è il doppio del valore, e rende il misuratore inutile. Crea una misura che calcoli un massimo sensato in modo dinamico e impostala come valore massimo.

### Task 4 — Parametro what-if
L'utente vuole poter simulare uno scenario: "se aumentassimo il target del X%, lo raggiungeremmo?"

1. Crea un **parametro numerico** (Modellazione > Nuovo parametro > Intervallo numerico) da 0% a 50%, incremento 5%
2. Modifica la misura obiettivo perché usi il valore del parametro invece del 10% fisso
3. Metti lo slicer del parametro accanto al misuratore
4. Verifica che muovendo lo slicer il misuratore reagisca

### Task 5 — Molti a molti con relazione inattiva
Al Lab A5 hai risolto il molti-a-molti con la tabella ponte. Ora risolvilo con l'**approccio alternativo**, quello che usa il corso:

1. Mantieni la tabella ponte venditore-stato
2. Rendi **bidirezionale** la relazione tra ponte e dimensione Stato
3. Crea una relazione diretta venditore → fatti e rendila **inattiva**
4. Scrivi le misure che consumano forzatamente la relazione inattiva
5. Confronta i due approcci: quale preferisci e perché? Quale spiegheresti a un collega?

### Task 6 — Etichette dati per serie
Nel grafico a linee fatturato vs anno precedente:
1. Spegni le etichette solo sulla serie anno precedente
2. Sulla serie anno corrente, mostra come etichetta la **variazione percentuale** invece del valore
3. Formatta l'etichetta a 1 decimale con colore condizionale

**Consegna:** file .pbix + risposte ai punti riflessivi (Task 1.5, 2.6, 5.5).
