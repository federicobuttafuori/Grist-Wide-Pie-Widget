# Debug Learnings - Grist Wide Pie Widget

Tre filoni distinti: **flicker del grafico** (record parziali vs merge), **descrizioni colonna in legenda** (metadati `_grist_Tables_column` vs tabella attiva nel builder), e **toggle UI non rispondenti** (bootstrap `init` vs CSS in iframe).

## Riferimento rapido

- **Live Server / pagina senza `window.grist` sul primo tick:** il vecchio bootstrap chiamava `init()` (e quindi `wireInputs()`) solo quando `grist.ready` era già disponibile. Senza Grist, dopo il timeout del poll **`init()` non veniva mai eseguito** → nessun listener su ⚙/🐞. **Fix risolutivo:** chiamare comunque **`init()` subito** quando manca Grist, così la shell UI è cablata; usare **`state.uiShellWired`** per non duplicare tooltip/drag/listener input se `init()` viene richiamato quando Grist compare dopo; usare **`state.gristWireDone`** dopo un `grist.ready` riuscito così **`grist.ready` e gli handler Grist non si registrano due volte**.
- **Iframe Grist (produzione):** `position: fixed` fuori dal root del widget + `transform` / stacking sugli antenati può far sì che i controlli siano visibili ma **non ricevano** i click. **Fix strutturale (separato):** controlli con `position: absolute` **dentro** `.app`, `z-index` alto, selettori corretti (es. `.app.settings-hidden .debug-toggle` se il toggle è figlio di `.app`).

---

## Incident: toggle ⚙ / 🐞 non cliccabili (bug risolto)

### Problema originale

I pulsanti **impostazioni** (⚙) e **debug** (🐞) non reagivano ai click. Si riproduceva **nell’embed Grist** e, in modo decisivo per la diagnosi, anche con **Live Server** aprendo l’HTML senza host Grist. A volte la sidebar sembrava ancora “usabile”, il che portava a ipotizzare overlay o `pointer-events`, ma non era l’unica causa.

### Bug reale e causa (due problemi distinti, spesso confusi)

1. **Listener mai registrati (caso Live Server / assenza di `window.grist` al bootstrap):**  
   `waitForGristAndInit()` chiamava `init()` **solo** quando `window.grist` e `grist.ready` erano già presenti. Senza Grist, il timer andava in timeout e **`init()` non veniva mai eseguito** → **`wireInputs()` non girava mai** → i toggle **non avevano handler `click`**. Non era un problema solo di CSS: **non c’era nulla da “cliccare” a livello di logica.**

2. **Hit-test rotto nell’iframe (Grist in produzione):**  
   Controlli con `position: fixed` fuori dalla radice di stacking del widget, insieme a `transform` sugli antenati, possono rendere gli elementi visibili ma **non target di hit-test**. Qui servono DOM/CSS corretti (controlli **dentro** `.app`, `position: absolute`, stacking alto). Questo **non** risolve (1) se `init()` non parte.

### Cosa è stato provato prima del fix finale (non risolveva il caso “`init()` mai chiamato”)

- Retry e tempistiche su `grist.docApi`, `requiredAccess: "full"`, `grist.ready`.
- Aumento `z-index`, prove su `pointer-events`, spostamento dei controlli nel DOM.
- Spostamento di ⚙/🐞 e del pannello debug **dentro** `.app` e passaggio da **`position: fixed`** a **`position: absolute`** (utile per (2) in iframe; **non** sostituisce (1) se `init()` non viene invocato).
- Log nella finestra debug del widget e probe nella **console sviluppatore** (`elementFromPoint`, `pointerdown` in capture) per ipotesi overlay — utili per distinguere (1) vs (2), ma **non** sostituiscono il cablaggio di `init()`.

### Fix finale (quello che ha risolto il problema percepito dall’utente)

- **Chiamare `init()` anche quando Grist non c’è al primo tick:** così `wireInputs()`, `wireTooltip()` e `wireDebugWindowDrag()` vengono eseguiti (Live Server e anteprima locale funzionano).
- **Evitare doppie registrazioni:**  
  - **`state.uiShellWired`:** cabla una sola volta input + tooltip + drag della finestra debug; se `init()` viene richiamato quando Grist compare dopo, non si duplicano listener sulla canvas o sul drag.  
  - **`state.gristWireDone`:** dopo un `grist.ready` riuscito, non si ripete la registrazione di `grist.ready` e degli handler.
- **Mantenere** la correzione CSS/DOM per l’embed: controlli dentro `.app`, `position: absolute`, stacking alto (per (2)).

### Nota per agenti futuri

- **Sintomi simili a “layer invisibile sopra i pulsanti”** possono nascondere il fatto che **non esistono listener**. Prima di investire ore in hit-test e `pointer-events`, verificare che **`init()` / `wireInputs()` siano eseguiti** nell’ambiente che fallisce (es. HTML servito senza `window.grist`).
- **Non bloccare tutto il cablaggio UI sull’API Grist** se pannello e controlli devono essere usabili anche senza host: cablare prima la shell; agganciare Grist quando disponibile.
- **Separare le ipotesi:** “CSS iframe” vs “bootstrap che non chiama `init`” richiedono fix diversi; se la riproduzione include **sia embed sia Live Server**, considerare **entrambe** le cause.

---

## 1. Flicker del grafico (record parziali)

### Problema originale
Nel builder di Grist il grafico funzionava solo per un istante dopo il cambio riga: appariva correttamente e subito dopo tornava vuoto.

### Bug reale e causa
Il bug era una race tra due sorgenti dati:
- `onRecord` consegnava un record completo (grafico corretto).
- il polling `fetchSelectedRecord()` a volte consegnava subito dopo un record "parziale" con molte colonne `undefined`.
- quel record parziale sovrascriveva lo stato buono e il rendering trovava `0` valori validi.

Effetto visibile: flicker "appare e scompare".

### Modifiche fatte prima del fix finale (non risolutive da sole)
- Hardening `setOptions/onOptions` (debounce, anti-loop, guardie in builder).
- Retry/init più difensivo per `grist.ready`.
- Logging esteso in UI e console per tracciare pipeline selezione/parse/render.
- Polling builder introdotto per coprire i casi in cui `onRecord` non arrivava.
- Parsing numerico e lettura record resi più robusti (`record` e `record.fields`).
- Fallback automatico "all numeric columns" per evitare grafico vuoto.

Questi cambiamenti aiutavano a diagnosticare o mitigare, ma non eliminavano la sovrascrittura con record incompleto.

### Fix finale
- Introduzione merge record per stessa riga (`mergeGristRecords`):
  - se `id` riga è uguale, i campi `undefined` del nuovo snapshot non sovrascrivono i valori già validi.
  - i campi definiti continuano ad aggiornarsi normalmente.
- `adoptActiveRecord` usa sempre il merge sugli eventi (`onRecord` / `onRecords`).
- In passato era presente anche polling `fetchSelectedRecord` nel builder; è stato rimosso in favore del solo modello a eventi (meno RPC/console).
- Rimosso comportamento non desiderato di auto-selezione iniziale "tutte le colonne" e relativo fallback render.

### Nota di apprendimento per agenti futuri
Nel `custom-widget-builder` non basta "avere un fallback": bisogna anche evitare che il fallback degradi lo stato già valido.
Regola pratica: quando arrivano snapshot multipli della stessa entità, fare merge conservativo e non permettere a valori `undefined` di cancellare dati validi.

---

## 2. Descrizioni colonna sulla legenda (attributo `title` / hover)

### Problema originale
Le descrizioni delle colonne definite in Grist non comparivano al passaggio del mouse sulla **legenda** (tooltip nativo del browser). Il problema era evidente soprattutto nel **custom widget builder**, dove i log di debug mostravano `metaWithDescription: 0` e colonne del grafico senza testo associato.

### Bug reale e causa
1. **`fetchSelectedTable().tableId` nel builder** può essere un numero, un array di numeri, o una stringa tipo `T2_Ricette`. Quei numeri sono riferimenti a `_grist_Tables.id` (parent delle righe in `_grist_Tables_column`), non sempre la stessa “tabella logica” di cui `fetchSelectedTable().columns` elenca le colonne. Filtrare `_grist_Tables_column` con un `parentId` ricavato solo da `tableId` poteva puntare alla **tabella sbagliata** (es. poche colonne tipo `manualSort`), mentre `columnsMeta` descriveva un’altra tabella con decine di campi (`Costo_*`, ecc.): nessun match su `colId`, quindi nessuna descrizione in meta.
2. **UI**: il tooltip era applicato solo al contenitore / nome; spesso l’utente passava sul **valore numerico** o sulla **swatch**, dove `title` mancava.

### Modifiche fatte prima del fix finale (utili ma non sufficienti da sole)
- Leggere descrizioni dall’API colonne (`description` / `help` / `fields`) e da `_grist_Tables_column` tramite `grist.docApi.fetchTable`.
- **`resolveGristTableParentIds` / coercizione** (`tableId` array vs stringa, stringhe numeriche `"1"`): corretto l’interpretazione di `tableId`, ma **non** garantiva che il `parentId` risolto fosse la tabella delle colonne mostrate in `fetchSelectedTable().columns`.
- **Merge per label** (oltre a `colId`) tra meta e righe interne: utile come integrazione, non risolve se le righe considerate sono ancora della tabella sbagliata.
- **`scheduleColumnDescriptionEnrich`** (debounce su cambio record): ragionevole per aggiornare meta costruita solo dal record; va mantenuto; non era la radice del mismatch tabella/colonne.
- Log temporanei (`DEBUG_COLUMN_DESC`, `DEBUG_LEGEND_TITLE`, `legendDebug`, ecc.) per confrontare `internalColIds` vs `metaIds`.

### Fix finale
- **Niente affidamento a `parentId` da `sel.tableId`** per decidere quali righe di `_grist_Tables_column` leggere. Si fa una scansione di **tutte** le righe di `_grist_Tables_column` e si tengono solo quelle il cui **`colId`** (e, in seconda battuta, **`label`** normalizzato) compare nell’insieme degli id/label già presenti in `columnsMeta`. Così la descrizione viene associata alla colonna giusta **indipendentemente** dalla tabella interna segnalata dal builder.
- In **legenda**, impostare `title` (e cursore `help` dove serve) su **riga, swatch, nome e valore** così l’hover copre tutta la riga.

### Nota di apprendimento per agenti futuri
- In Grist, **`fetchSelectedTable().tableId` e l’elenco `columns` non sono una garanzia univoca** di quale `parentId` in `_grist_Tables_column` sia “la stessa tabella” in tutti gli host (in particolare nel builder). Per metadati colonna **per `colId`**, è più affidabile **filtrare per identificatore colonna** (e opzionalmente label) piuttosto che per tabella interna dedotta da `tableId`.
- Se i log mostrano descrizioni internal su colonne che non compaiono nel grafico e **meta** con tutt’altri `id`, sospettare subito **disallineamento di tabella/parent**, non solo rename o assenza di testo in Grist.
- **Traffico WebSocket / console:** non schedulare `fetchTable("_grist_Tables_column")` (tabella molto grande) ad ogni cambio riga o tick di polling. Le descrizioni sono **schema**, non valori di riga: basta arricchire dopo `fetchSelectedTable().columns`, al primo bootstrap da record, o quando compaiono **nuovi** `colId` in meta. In cache la risposta per la vita del widget (invalidare se ricarichi le colonne da API).

### Perché altri widget grafici “non aggiornano sempre” e vanno lo stesso
Molti widget si limitano a **`grist.onRecord`** (o equivalente) e ridisegnano quando **cambia la riga o i dati**. Il Wide Pie ora segue lo stesso modello (**solo eventi**, niente `setInterval` + `fetchSelectedRecord`). Non accoppiare fetch ripetuti su **tabelle interne** grosse al refresh riga: la console e il server si riempiono di RPC e compaiono warning tipo `RPC_UNKNOWN_REQID` (race tra risposte e nuove richieste).
