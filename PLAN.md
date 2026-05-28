# TD Observability Dashboard — Piano

Evoluzione dell'MVP `reference/monitor.html` verso una dashboard live, ospitata online, per monitorare ~15 installazioni TouchDesigner del museo immersivo **POSTOLOGY**.

Tre fasi: **Fase 0** (tutto free tier, niente carta di credito), **Fase 1** (storico server-side, richiede attivazione Blaze), **Fase 2** (alerting attivo e quality-of-life).

---

## Contesto

- **Team utente**: 4-5 persone interne WOA Studio. Tutti con email `@woacreativecompany.com`.
- **Cliente / scopo**: monitoring continuo delle installazioni POSTOLOGY (un solo cliente, museo immersivo, ~15 macchine TD distribuite tra le stanze/esperienze).
- **Gerarchia dati**: esperienza/stanza (= `group`) → touchpoint (= singolo PC con TouchDesigner).
- **Vincolo principale**: basso attrito d'uso. Deve essere apribile da mobile in autobus, da un proiettore in studio, da casa, senza step di login complessi.
- **Filosofia**: partire gratis, aggiungere costi solo quando portano valore concreto. Tutti gli step verso Fase 1/2 sono additivi: nessun rework.

---

## Stack scelto

| Layer | Tecnologia | Piano | Note |
|---|---|---|---|
| Hosting dashboard | Cloudflare Pages | Free | Deploy automatico da GitHub repo, CDN globale |
| Database | Firebase Firestore | Spark (free) in Fase 0, Blaze in Fase 1 | Real-time via `onSnapshot` |
| Auth utenti | Firebase Auth — Google provider + check dominio `@woastudio.<tld>` | Free | Niente Cloudflare Access, niente account terzi |
| Write da TD | Firestore Admin SDK con service account | Free | Credenziali in DAT del TOX, mai esposte al client |
| Storico eventi | Cloud Functions + collection `events` con TTL 30g | **Fase 1**, richiede Blaze | Free tier Blaze copre questo volume |
| Alert | Telegram bot via Cloud Function | **Fase 2** | Hook predisposto già in Fase 1 |

---

## Fase 0 — Production-ready, tutto free tier

Obiettivo: portare l'MVP online, protetto, robusto, su tutte le 15 installazioni reali. Senza storico server-side, senza alert. Si sostituisce all'HTML locale attuale.

### Step 0.1 — Setup infrastruttura

- [ ] Nuovo Firebase project `woa-td-monitor` (production) e `woa-td-monitor-dev` (staging). Piano: **Spark**.
- [ ] Firestore attivato in *production mode* su entrambi.
- [ ] Restrizione apiKey al dominio finale dell'app (Google Cloud Console → Credentials).
- [ ] Security rules Firestore:
  ```
  rules_version = '2';
  service cloud.firestore {
    match /databases/{database}/documents {
      match /installations/{docId} {
        allow read: if request.auth != null
                    && request.auth.token.email.endsWith('@woacreativecompany.com');
        allow write: if false;  // solo service account scrive
      }
    }
  }
  ```
- [ ] Cloudflare Pages connesso a questa GitHub repo. Deploy automatico su push a `main`. URL: `woa-td-monitor.pages.dev`.
- [ ] Firebase Auth → abilitare provider Google. Authorized domain include `woa-td-monitor.pages.dev`.
- [ ] Schermata login e activity log in-session (A+B): dashboard live + log delle transizioni di stato tenuto in memoria/localStorage per la sessione corrente.

### Step 0.2 — Schema dati

**`installations/{docId}`** (un doc per touchpoint, sovrascritto ad ogni heartbeat):

| Campo | Tipo | Note |
|---|---|---|
| `id` | string | Nome leggibile del touchpoint, es. `"DO_WHAT_YOU_LOVE_0"` |
| `group` | string | Esperienza/stanza |
| `fps` | number | Normalizzato lato TD (non più stringa) |
| `status` | boolean | TD vivo & operativo |
| `winopen` | boolean | Normalizzato |
| `lastSeen` | timestamp | Heartbeat |
| `version` | string | **NEW** — versione patch TD |
| `host` | string | **NEW** — nome PC / hostname |

### Step 0.3 — Dashboard (evoluzione del file `monitor.html`)

Mantengo il design attuale (è già buono). Modifiche:

- [ ] Aggiunta `firebase-auth-compat.js` allo stack.
- [ ] Schermata "Login con Google" se utente non autenticato. Dopo il login, check client-side che l'email termini con `@woacreativecompany.com`; altrimenti sign-out + messaggio "non autorizzato".
- [ ] Cards mostrano i nuovi campi `version` (piccolo, in alto) e `host` (footer).
- [ ] **Activity log in-session** (collassabile, fondo pagina): il client osserva le transizioni di stato di ogni touchpoint dal momento in cui apri la dashboard, tiene un log delle ultime ~50 transizioni in memoria + localStorage. Si perde quando si svuota lo storage, ma copre la sessione di chi sta guardando in tempo reale. Sarà sostituito dalla versione server-side in Fase 1.
- [ ] **Web Notifications API**: al primo load chiedo permesso; quando una card passa a offline mentre il tab è in background, notifica desktop nativa.
- [ ] La regola `nodeStatus()` (online/warning/offline da `status` + `lastSeen`) viene estratta in una funzione singola, riusata identica per il rendering e per la detection delle transizioni del log.

### Step 0.4 — TOX TouchDesigner (via MCP)

Da fare quando attivi l'MCP server. Review del TOX esistente prima, poi:

- [ ] Le write su Firestore passano da **service account** (Admin SDK), non più dal client-side apiKey. Credenziali in un DAT del TOX, fuori dal version control delle patch.
- [ ] Aggiunta dei campi `version` e `host` al payload scritto.
- [ ] **Normalizzazione tipi** prima della write: `fps` come number, `status` e `winopen` come boolean (non più stringhe "True"/"False"). Toglie un layer di fragilità che oggi gestiamo client-side.
- [ ] **Cadenza scrittura: 60 secondi** (le installazioni sono accese ~14h/day, non 24/7 → 12.600 write/day, dentro Spark con margine). Soglia offline lato client: 3 minuti.
- [ ] **Heartbeat separato dal render loop**: timer Python indipendente dentro TD. Se la GPU si inchioda l'heartbeat continua, e davvero se l'heartbeat si ferma è perché TD è morto.
- [ ] **Retry con backoff** se la write fallisce (max 3 tentativi).
- [ ] **Config esternalizzato**: `id`, `group`, `version` letti da un DAT separato. Onboardare una nuova installazione = copiare il TOX standard + cambiare 3 righe di config, niente patch custom per ogni macchina.

### Step 0.5 — Test & rollout

- [ ] Test su staging (`woa-td-monitor-dev`) con 1-2 macchine fisiche (anche dummy).
- [ ] Verifica scenari: heartbeat normale, disconnessione rete temporanea, crash TD (kill -9 simulato), riavvio macchina, finestra chiusa, FPS sotto soglia.
- [ ] Documentazione `docs/install-procedure.md` con i 4-5 step per onboardare una nuova installazione.
- [ ] Switch al project prod. Le 15 installazioni vanno live insieme.

### Stima Fase 0

~2 giornate di lavoro effettive, gating principale sull'MCP attivo per il TOX.

### Cosa NON c'è in Fase 0 (esplicito)

- Storico server-side persistente.
- Alert email/Telegram.
- Grafici time-series FPS.

---

## Fase 1 — Storico server-side (richiede Blaze)

Trigger per partire: tu hai 5 minuti per aggiungere la carta su Google Cloud, e attiviamo budget alert a $5/mese. Costo previsto reale: **sotto $1/mese**, probabilmente $0 perché Blaze include lo stesso free tier di Spark.

### Step 1.1 — Attivazione Blaze

- [ ] Upgrade Firebase project da Spark a Blaze.
- [ ] **Budget alert** Google Cloud: notifica email a $5, $10, $20/mese.
- [ ] Cadenza scrittura TD può tornare a **60 secondi** (rate di scrittura più aggressivo = detection più rapida).
- [ ] Soglia offline lato client può scendere a 2-3 minuti.

### Step 1.2 — Schema aggiunto

**`events/{autoId}`** (append-only, TTL 30g):

| Campo | Tipo | Note |
|---|---|---|
| `touchpoint` | string | ref a `installations.id` |
| `group` | string | denormalizzato per query veloci |
| `from` | string | `"online"` \| `"warning"` \| `"offline"` |
| `to` | string | idem |
| `fps` | number | snapshot al momento della transizione |
| `timestamp` | timestamp | quando è successo |
| `expiresAt` | timestamp | per Firestore TTL policy (= timestamp + 30g) |

- [ ] TTL policy Firestore su `events.expiresAt`.

### Step 1.3 — Cloud Functions

- [ ] `onInstallationStatusChange` (trigger: `onDocumentUpdated` su `installations/{docId}`).
  - Calcola `prevStatus` dal before-snapshot e `newStatus` dall'after, usando la **stessa identica regola** del client (lastSeen + status, soglia 2 min). La funzione condivide byte-per-byte la logica di `nodeStatus()`.
  - Se i due stati differiscono, scrive un doc in `events`.
  - Hook predisposto per Fase 2 (chiama `notifyTelegram()` ma per ora è no-op).
- [ ] `scheduledOfflineCheck` (cron: ogni 60 secondi).
  - Risolve il caso edge: quando un touchpoint smette di scrivere, nessun `onUpdate` viene mai triggerato → la transizione `online → offline` non viene registrata.
  - Per ogni doc in `installations`, calcola lo stato corrente. Se è `offline` ma l'ultimo evento in `events` per quel touchpoint era `online`/`warning`, scrive l'evento mancante.
  - Volume: 1.440 invocazioni/giorno, ampiamente dentro free tier Functions.

### Step 1.4 — Dashboard

- [ ] Sostituire l'activity log in-session con una sottoscrizione real-time a `events` (`orderBy('timestamp', 'desc').limit(50)`).
- [ ] Eventuale pulsante "Carica altri" per andare indietro (paginazione query Firestore).

### Stima Fase 1

~1 giornata di lavoro effettiva.

---

## Fase 2 — Quality of life

Da fare solo se/quando si sente l'esigenza, dopo settimane di dashboard a regime.

- [ ] **Alert Telegram**: bot dedicato, gruppo WOA. Cloud Function `notifyTelegram()` chiamata da `onInstallationStatusChange` quando un touchpoint passa a `offline` da più di N minuti. Configurabile per touchpoint (alcune installazioni possono essere "silenziabili" durante test).
- [ ] **Pannello dettaglio touchpoint**: click su una card → vista con timeline degli ultimi 7 giorni di eventi per quel singolo touchpoint.
- [ ] **Grafici FPS time-series**: aggiungere snapshot ogni 5 minuti in una collection `metrics`, visualizzare con sparkline nelle cards o grafico nel pannello dettaglio. Solo se serve davvero.
- [ ] **Controllo remoto** (out-of-scope per ora ma fattibile): pulsante "restart" che scrive in una collection `commands`, il TOX la legge e si auto-restarta. Richiede pensare bene la sicurezza.

---

## Logica di status — fonte unica di verità

Da mantenere identica byte-per-byte tra client (`monitor.html`) e Cloud Function (`functions/index.ts` in Fase 1).

```
inputs:
  status   : boolean
  lastSeen : timestamp
  now      : timestamp
  threshold: minutes (default: 4 in Fase 0, 2 in Fase 1)

output: "online" | "warning" | "offline"

logic:
  if not lastSeen          → "offline"
  if (now - lastSeen) > threshold → "offline"
  if status is true        → "online"
  else                     → "warning"
```

---

## Calcolo costi e quote — verifica numerica

**Fase 0 (Spark)** — cadenza scrittura 60s, installazioni accese ~14h/day:
- Write/day: 15 × (14 × 3600 / 60) = **12.600** → dentro 20k Spark ✓
- Read/day (worst case, 5 client aperti tutte le 14h): 12.600 × 5 = 63.000 → *leggermente fuori* 50k Spark
- Read/day (realistico, 2-3 client): ~25-38k → dentro ✓
- Se 5 persone guardano la dashboard tutto il giorno tutti i giorni, è il segnale per passare a Fase 1 / Blaze.

**Fase 1 (Blaze)** — cadenza 60s, free tier Blaze (= 50k read, 20k write/day gratis, poi $0.06 / 100k write, $0.06 / 100k read):
- Write: 15 × 1440 = **21.600/day** → 1.600 sopra free = $0.001/day = **$0.03/mese**
- Read (5 client): 108.000/day → 58.000 sopra free = $0.035/day = **$1.05/mese**
- Storage (events 30g a ~100 byte): ~3 MB → gratis
- Functions: ~22.000 invocazioni/day → ben dentro 125k/day free
- **Totale realistico: ~$1-2/mese.**

---

## Decisioni che servono da te per partire

1. **Crea i due Firebase project** (`woa-td-monitor` prod e `woa-td-monitor-dev` staging), condividi le config (apiKey, projectId, ecc.) — poi procedo io con tutto il resto.
2. **Cloudflare Pages**: crea account (o usane uno esistente) e connetti questa repo — poi imposto io le build settings.
3. **Quando attivi l'MCP TouchDesigner**, partiamo dallo Step 0.4 (lavoro sul TOX) — può succedere in parallelo agli altri step.
4. **Niente da decidere su Blaze per ora**: si attiva quando si attiva, e gli Step 1.x partono allora.

---

## Cosa manca ancora (rischi noti)

- **Reliability rete museo**: non sappiamo se la connessione internet del museo è stabile. Se ci sono disconnessioni frequenti, vale la pena aggiungere buffer offline lato TD (Step 0.4 può esserne estensione). Da capire dopo i primi giorni.
- **Watchdog su crash TD**: se TD muore, chi lo fa ripartire? Nello scope di questo progetto possiamo solo *vedere* il crash, non risolverlo. Vale la pena pensare a un servizio Windows / script di restart automatico, ma è fuori scope dashboard.
- **Versioning patch TD**: il campo `version` è solo informativo. Se vogliamo davvero gestire deploy/rollback delle patch da remoto, è un progetto separato (potenzialmente molto utile, ma grosso).
