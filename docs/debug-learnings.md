# Debug Learnings - Grist Wide Pie Widget

## Problema originale
Nel builder di Grist il grafico funzionava solo per un istante dopo il cambio riga: appariva correttamente e subito dopo tornava vuoto.

## Bug reale e causa
Il bug era una race tra due sorgenti dati:
- `onRecord` consegnava un record completo (grafico corretto).
- il polling `fetchSelectedRecord()` a volte consegnava subito dopo un record "parziale" con molte colonne `undefined`.
- quel record parziale sovrascriveva lo stato buono e il rendering trovava `0` valori validi.

Effetto visibile: flicker "appare e scompare".

## Modifiche fatte prima del fix finale (non risolutive da sole)
- Hardening `setOptions/onOptions` (debounce, anti-loop, guardie in builder).
- Retry/init più difensivo per `grist.ready`.
- Logging esteso in UI e console per tracciare pipeline selezione/parse/render.
- Polling builder introdotto per coprire i casi in cui `onRecord` non arrivava.
- Parsing numerico e lettura record resi più robusti (`record` e `record.fields`).
- Fallback automatico "all numeric columns" per evitare grafico vuoto.

Questi cambiamenti aiutavano a diagnosticare o mitigare, ma non eliminavano la sovrascrittura con record incompleto.

## Fix finale
- Introduzione merge record per stessa riga (`mergeGristRecords`):
  - se `id` riga è uguale, i campi `undefined` del nuovo snapshot non sovrascrivono i valori già validi.
  - i campi definiti continuano ad aggiornarsi normalmente.
- `adoptActiveRecord` usa sempre il merge, anche per eventi/polling.
- Polling builder mantenuto come fallback affidabile, senza più degradare lo stato.
- Rimosso comportamento non desiderato di auto-selezione iniziale "tutte le colonne" e relativo fallback render.

## Nota di apprendimento per agenti futuri
Nel `custom-widget-builder` non basta "avere un fallback": bisogna anche evitare che il fallback degradi lo stato già valido.
Regola pratica: quando arrivano snapshot multipli della stessa entità, fare merge conservativo e non permettere a valori `undefined` di cancellare dati validi.
