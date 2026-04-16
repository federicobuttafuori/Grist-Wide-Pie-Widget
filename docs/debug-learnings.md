# Debug Learnings - Grist Wide Pie Widget

## Problema originale
Il widget mostrava spesso `No numeric columns detected for this record` o rimaneva bloccato su `Waiting for active record...` anche con una riga attiva valida in Grist.

## Bug reale
Nel contesto `custom-widget-builder` di Grist, gli eventi `onRecord`/`onRecords` e alcune interfacce RPC (`setOptions`, mapping/options APIs) non erano affidabili in modo consistente.  
In parallelo, i dati arrivavano talvolta in formato `record.fields` e non sempre come oggetto flat.

## Causa principale
- Dipendenza forte dagli eventi runtime (`onRecord`/`onRecords`) in un host che a volte non li emette.
- Assunzione iniziale che il record fosse sempre in un solo formato.
- Rumore RPC nel builder (`Unknown interface` / `Unknown reqId`) che rendeva fuorviante la diagnosi.

## Modifiche fatte PRIMA del fix finale (non risolutive da sole)
- Hardening generale di `setOptions/onOptions` con debounce e anti-loop.
- Rimozione script esterno Grist API e init differita.
- Disattivazione parziale di alcune API in builder mode.
- Vari pannelli/log temporanei di debug in UI.
- Fallback iniziali non sufficienti basati solo su callback eventi.

Queste modifiche hanno ridotto i crash, ma non garantivano il caricamento del record quando gli eventi non partivano.

## Fix finale (risolutivo)
- Mantenuti `onRecord`/`onRecords` quando disponibili.
- Aggiunto fallback robusto in builder mode: polling di `grist.docApi.fetchSelectedRecord()` ogni ~1.2s.
- Aggiornamento record solo quando cambia il contenuto, per evitare render inutili.
- Normalizzazione robusta lettura valori (`record[col]` + fallback `record.fields[col]`).

## Pulizia finale
- Rimossi log e pannelli debug temporanei.
- Rimossi watchdog/probe diagnostici non necessari in produzione.
- Lasciata solo la soluzione stabile necessaria al funzionamento.

## Nota di apprendimento per agenti futuri
Nel `custom-widget-builder` non assumere che gli eventi record siano sempre affidabili.  
Se il widget dipende dal record attivo, prevedere subito una strategia duale:
1) subscription eventi, 2) fallback `docApi.fetchSelectedRecord()` con polling leggero e confronto delta.
