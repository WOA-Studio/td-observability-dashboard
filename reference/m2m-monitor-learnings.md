# M2M Monitor — Learnings v1

## Stack

**Firebase Firestore** (non Realtime DB) è la scelta corretta per questo use case.
`onSnapshot` su una collection dà aggiornamenti push WebSocket-like, zero polling,
funziona sul piano Spark (gratuito). Realtime DB sarebbe ridondante.

Il file HTML è completamente self-hosted e standalone: Firebase SDK via CDN,
zero build step, apribile anche da `file://` in locale o deployabile su qualsiasi
static host (Cloudflare Pages, Netlify, ecc.).

---

## Schema Firestore — campi minimi validati

```
installations / {docId}
  id        : string   — nome leggibile del touchpoint (es. "DO WHAT YOU LOVE_0")
  group     : string   — chiave di raggruppamento (es. "DO WHAT YOU LOVE")
  fps       : string   — TouchDesigner manda FPS come stringa, non number
  status    : boolean  — true/false (a volte arriva come stringa "true"/"True")
  winopen   : string   — "True" / "False" (non boolean)
  lastSeen  : timestamp — Firestore Timestamp, usare .toDate() per convertire
```

**Attenzione ai tipi misti**: TouchDesigner tende a mandare boolean e numeri
come stringhe. Il client deve normalizzare esplicitamente, non fare affidamento
sul tipo nativo del campo.

---

## Logica di status derivato

Lo status display non viene letto direttamente dal campo `status`, ma è calcolato
lato client combinando due segnali:

```
offline  → lastSeen più vecchio di N minuti (default: 2)
warning  → lastSeen recente MA status === false
online   → lastSeen recente E status === true
```

Questo rende il monitor robusto a crash silenziosi: un processo TD che smette
di scrivere viene correttamente marcato offline anche se l'ultimo valore scritto
era `status: true`.

La soglia `OFFLINE_THRESHOLD_MIN` deve essere calibrata sul rate di scrittura
reale di TouchDesigner. Con aggiornamenti ogni ~30s, 2 minuti dà margine sufficiente.

---

## Regole Firestore

Per un monitor interno, read pubblica sulla sola collection `installations` è accettabile:

```
match /installations/{docId} {
  allow read: if true;
}
```

Il resto del database rimane protetto. Se il monitor va su un dominio pubblico,
valutare autenticazione Firebase (Anonymous auth è il minimo overhead).

---

## Architettura UI — pattern consolidati

**Accordion per gruppo** è il layout vincente quando i gruppi sono N variabile
e ogni gruppo ha M touchpoint variabili. Permette di tenere visibili tutti i
gruppi in alto e drillare solo su quello che interessa.

**Stato open/closed dell'accordion va preservato** tra i re-render (Firestore
triggera `renderAll()` ad ogni cambio). Salvare `{ [groupName]: bool }` prima
di svuotare il DOM e riapplicarlo dopo.

**Semafori collassati vs counter espanso**: quando il gruppo è chiuso, i dot
colorati danno uno scan visivo immediato senza aprire. Quando è aperto, il
counter numerico `N/N` è più informativo. I due elementi si scambiano via CSS
(`.group-block .tp-dots { display: flex }` / `.group-block.open .tp-dots { display: none }`),
nessun JS aggiuntivo.

**Pulse sul bordo** (non flash sul background) è la notifica di update giusta
per una dashboard sempre visibile: poco invasivo, immediatamente localizzato
sulla card che cambia. Implementato con `@keyframes` + `requestAnimationFrame`
dopo il re-render per garantire che il reflow sia avvenuto prima di aggiungere
la classe.

---

## Scalabilità — dimensioni testate/stimate

- 3 touchpoint, 1 gruppo → testato, funziona
- ~30 touchpoint, ~10 gruppi → stimato gestibile senza ottimizzazioni
- Oltre ~100 touchpoint attivi con aggiornamenti frequenti → valutare
  di non fare `renderAll()` completo ad ogni snapshot, ma aggiornare
  solo le card dei `docChanges()` di tipo `modified` in-place

---

## Next implementation — punti aperti

- **Autenticazione**: anche solo Firebase Anonymous per evitare read pubbliche
- **Notifiche offline**: alert visivo/sonoro quando un touchpoint passa online→offline
- **Storico**: Firestore non ha TTL nativo, valutare Firestore + Cloud Function
  per archiviare snapshot periodici su una sub-collection `history`
- **Multi-tenant**: se gruppi diversi appartengono a progetti diversi,
  considerare una collection `projects` padre e `installations` come sub-collection
- **Rate di aggiornamento TD**: documentare e standardizzare lato TouchDesigner
  (ogni quanto scrive, quale DAT, su crash cosa succede al lastSeen)
- **Campo `version`**: aggiungere versione del progetto/patch TD al documento
  per debug remoto
