# Debug Learnings - Grist Wide Pie Widget

Tre filoni distinti: **flicker del grafico** (record parziali vs merge), **descrizioni colonna in legenda** (metadati `_grist_Tables_column` vs tabella attiva nel builder), e **toggle UI non rispondenti** (bootstrap `init` vs CSS in iframe).

---

## Incident (cross-project): ordini Stripe non creati su Grist

### Problema originale

Pagamento Stripe completato ma:
- nessun ordine creato in Grist (`T91_Ordini`)
- nessuna email di conferma/notifica admin (dipendevano dal ramo "ordine creato")

### Causa reale del bug

Il backend inviava in `Prodotti` una RefList con record `T9_Magazzino` già referenziati in ordini precedenti.  
In questo schema Grist scattava:
- `UniqueReferenceError UNIQUE reference constraint violated`

Quindi il create ordine falliva con `400` nel webhook.

### Evidenza runtime decisiva

Nei log:
- `duplicates: 0` su `product_record_ids` (quindi non era duplicazione interna payload)
- `existingOrdersWithSamePaymentIntent: 0` (quindi non era replay stesso payment intent)
- `existingOrdersUsingSameProducts: [{ orderId: 97, ... }]` (riuso effettivo di record già collegati)
- errore create persistente: `UniqueReferenceError`

### Fix finale applicato

Nel calcolo carrello server-side:
- prima del create ordine legge `T91_Ordini`
- costruisce il set dei `Prodotti` già collegati
- seleziona solo record prodotto non ancora linkati
- se i record liberi non bastano, fallisce in modo esplicito con errore chiaro (invece di inviare payload invalido a Grist)

### Apprendimento operativo

Quando si usa una colonna `RefList` in Grist, non assumere che i record possano essere riutilizzati infinite volte: validare sempre i vincoli reali dello schema con evidenza runtime e non solo con i tipi colonna dichiarati.

### Follow-up: mismatch carrello vs magazzino (r_uid)

#### Problema
- In checkout alcuni prodotti risultavano "non disponibili" anche se in magazzino esisteva una variante disponibile dello stesso gusto/peso.
- Log tipico: `Prodotti non disponibili ... codici_candidati: ...`.

#### Causa
- Il carrello passava un `r_uid` storico (es. lotto/versione precedente ricetta).
- Il backend cercava prima per `gusto+peso+r_uid` e trovava solo record vecchi (spesso già venduti/occupati), escludendo la variante nuova disponibile con stesso `gusto+peso` ma `r_uid` diverso.

#### Fix
- In `validateAndCalculateCart`, quando non ci sono record disponibili nel match iniziale, il backend allarga il match a tutti i record con stesso `gusto+peso` (ignorando `r_uid`) e poi applica i vincoli di disponibilità/record già collegati.
- Se anche dopo l'allargamento non ci sono pezzi liberi, ritorna errore esplicito (nessun ordine sporco).

#### Learnings
- `r_uid` è utile come preferenza, ma non deve bloccare la vendita se il prodotto commerciale è ancora lo stesso (gusto+peso) e la disponibilità reale è su una variante di lotto/versione diversa.

## Riferimento rapido

- **Widget servito come URL (es. Pages / hosting proprio):** senza `<script src="https://docs.getgrist.com/grist-plugin-api.js">` (o equivalente sullo stesso host Grist) **`window.grist` non esiste** → `hasGrist=false`, nessun `grist.ready`, nessuna colonna. La documentazione Grist richiede questo script nel markup del custom widget.
- **Live Server / pagina senza `window.grist` sul primo tick:** il vecchio bootstrap chiamava `init()` (e quindi `wireInputs()`) solo quando `grist.ready` era già disponibile. Senza Grist, dopo il timeout del poll **`init()` non veniva mai eseguito** → nessun listener su ⚙/🐞. **Fix risolutivo:** chiamare comunque **`init()` subito** quando manca Grist, così la shell UI è cablata; usare **`state.uiShellWired`** per non duplicare tooltip/drag/listener input se `init()` viene richiamato quando Grist compare dopo; usare **`state.gristWireDone`** dopo un `grist.ready` riuscito così **`grist.ready` e gli handler Grist non si registrano due volte**.
- **Chrome UI (⚙ / 🐞 / pannello debug):** layout come versione “originale”: **`position: fixed`** al **viewport** (pulsanti sempre in basso a sinistra anche se `.app` è più alto dello schermo; con **`absolute`** rispetto a `.app` scorrevano via col contenuto). DOM: pulsanti e `#debugWindow` **sibling** di `.app` (non dentro), selettore **`.app.settings-hidden ~ .debug-toggle`**. `z-index` 1100/1200. In iframe rari casi di stacking/`transform` sugli antenati possono ancora influenzare il hit-test: se succede, diagnosticare per quel host.

---

## Incident: toggle ⚙ / 🐞 non cliccabili (bug risolto)

### Problema originale

I pulsanti **impostazioni** (⚙) e **debug** (🐞) non reagivano ai click. Si riproduceva **nell’embed Grist** e, in modo decisivo per la diagnosi, anche con **Live Server** aprendo l’HTML senza host Grist. A volte la sidebar sembrava ancora “usabile”, il che portava a ipotizzare overlay o `pointer-events`, ma non era l’unica causa.

### Bug reale e causa (due problemi distinti, spesso confusi)

1. **Listener mai registrati (caso Live Server / assenza di `window.grist` al bootstrap):**  
   `waitForGristAndInit()` chiamava `init()` **solo** quando `window.grist` e `grist.ready` erano già presenti. Senza Grist, il timer andava in timeout e **`init()` non veniva mai eseguito** → **`wireInputs()` non girava mai** → i toggle **non avevano handler `click`**. Non era un problema solo di CSS: **non c’era nulla da “cliccare” a livello di logica.**

2. **Aspetto / posizione dei FAB:** con **`position: absolute`** in basso rispetto a `.app`, se il contenuto supera l’altezza dello schermo i pulsanti **escono dalla vista** (stesso blocco di scroll). La UI “migliore” usa **`position: fixed`** al viewport e (opzionale) DOM sibling di `.app`. Questo è **indipendente** da (1): senza `init()` i click non partono comunque.

### Cosa è stato provato prima del fix finale (non risolveva il caso “`init()` mai chiamato”)

- Retry e tempistiche su `grist.docApi`, `requiredAccess: "full"`, `grist.ready`.
- Aumento `z-index`, prove su `pointer-events`, spostamento dei controlli nel DOM.
- Variante intermedia: controlli **dentro** `.app` con **`position: absolute`** (migliora stacking in alcuni iframe ma i FAB possono **scrollare** via se `.app` è alto; **non** sostituisce (1) se `init()` non viene invocato).
- Log nella finestra debug del widget e probe nella **console sviluppatore** (`elementFromPoint`, `pointerdown` in capture) per ipotesi overlay — utili per distinguere (1) vs (2), ma **non** sostituiscono il cablaggio di `init()`.

### Fix finale (quello che ha risolto il problema percepito dall’utente)

- **Chiamare `init()` anche quando Grist non c’è al primo tick:** così `wireInputs()`, `wireTooltip()` e `wireDebugWindowDrag()` vengono eseguiti (Live Server e anteprima locale funzionano).
- **Evitare doppie registrazioni:**  
  - **`state.uiShellWired`:** cabla una sola volta input + tooltip + drag della finestra debug; se `init()` viene richiamato quando Grist compare dopo, non si duplicano listener sulla canvas o sul drag.  
  - **`state.gristWireDone`:** dopo un `grist.ready` riuscito, non si ripete la registrazione di `grist.ready` e degli handler.
- **Chrome UI:** ripristinare **`position: fixed`**, sibling di `.app`, e `z-index` coerenti (comportamento FAB sul viewport).

### Nota per agenti futuri

- **Sintomi simili a “layer invisibile sopra i pulsanti”** possono nascondere il fatto che **non esistono listener**. Prima di investire ore in hit-test e `pointer-events`, verificare che **`init()` / `wireInputs()` siano eseguiti** nell’ambiente che fallisce (es. HTML servito senza `window.grist`).
- **Non bloccare tutto il cablaggio UI sull’API Grist** se pannello e controlli devono essere usabili anche senza host: cablare prima la shell; agganciare Grist quando disponibile.
- **Separare le ipotesi:** “CSS iframe” vs “bootstrap che non chiama `init`” richiedono fix diversi; se la riproduzione include **sia embed sia Live Server**, considerare **entrambe** le cause.

---

## Incident: nessuna colonna / `hasGrist=false` in produzione (URL ospitato)

### Problema originale

In **produzione** (widget caricato come URL, es. `/Grist-Wide-Pie-Widget/?access=full…`): **nessuna colonna** in meta, grafico **“waiting for row”**, sidebar senza elenco colonne utile. Nei log compariva **`grist: hasGrist=false`**, **`wireDone=false`**, **`docApi: skipped_no_grist`**. L’utente aveva già concesso **accesso** al documento; sembrava un problema di permessi o di bootstrap.

### Bug reale e causa

Il file HTML **non includeva** lo script ufficiale che crea **`window.grist`**. La [documentazione Grist per custom widget](https://support.getgrist.com/widget-custom/) richiede esplicitamente:

`https://docs.getgrist.com/grist-plugin-api.js`

Senza quel file, nel contesto del widget **non esiste mai** l’oggetto `grist` → nessun `grist.ready`, nessun `onRecord`, nessun `docApi.fetchSelectedTable`, **indipendentemente** da `access=full` nell’URL o dalle impostazioni del documento. Non era un bug di permessi: era **API assente nella pagina**.

### Cosa è stato provato prima del fix finale (non risolveva la causa radice)

- Ritentativi su `docApi`, `requiredAccess: "full"`, ordine di `init` / `wireGrist`.
- Blocco **Diagnostics** nel pannello debug (contatori eventi, UA, viewport) — utile per **vedere** `hasGrist=false`, ma **non** sostituisce la presenza dello script plugin.
- Ipotesi “permessi” o “tabella non collegata” senza verificare prima se `window.grist` esiste nel frame del widget.

### Fix finale

Aggiungere **nel markup**, **prima** dello script principale del widget, una riga del tipo:

`<script src="https://docs.getgrist.com/grist-plugin-api.js"></script>`

(Caricamento sincrono in coda al `body` va bene: lo script viene eseguito prima dell’IIFE del widget.)  
**Self-hosted:** se la CSP blocca `docs.getgrist.com`, usare l’URL del `grist-plugin-api.js` servito dal proprio server Grist.

### Nota per agenti futuri

- Se **`hasGrist=false`** e il widget è servito come **file/URL proprio**, la prima verifica è: **è incluso `grist-plugin-api.js`?** — non solo “è in Grist?”.
- **`access=full` nell’URL** non inietta l’API: serve solo lo script documentato.
- Dopo il fix, i sintomi attesi sono `hasGrist=true`, `wireDone=true`, colonne da `fetchSelectedTable` quando la tabella è collegata.

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
