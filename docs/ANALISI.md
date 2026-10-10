# PWApp — Documento di analisi

> Catalogo moderno di Progressive Web App inviate dagli utenti, con backend in Docker e app Android nativa capace di "installare" le PWA come app vere. Bilingue (IT/EN), pensato per estendersi in futuro a iOS e desktop.

| | |
|---|---|
| Versione | 0.2 (bozza di analisi: motore ibrido TWA + WebView) |
| Data | 10 ottobre 2026 |
| Stato | Da validare prima dell'avvio dello sviluppo |

---

## Indice

1. [Visione e obiettivi](#1-visione-e-obiettivi)
2. [Attori e casi d'uso](#2-attori-e-casi-duso)
3. [Il nodo tecnico: come si "installa" una PWA da un'app di terzi](#3-il-nodo-tecnico-come-si-installa-una-pwa-da-unapp-di-terzi)
4. [Architettura complessiva](#4-architettura-complessiva)
5. [Backend (progetto Docker)](#5-backend-progetto-docker)
6. [Pipeline di acquisizione di una PWA dall'URL](#6-pipeline-di-acquisizione-di-una-pwa-dallurl)
7. [App Android nativa](#7-app-android-nativa)
8. [Registrazione e autenticazione](#8-registrazione-e-autenticazione)
9. [Design, animazioni ed esperienza d'uso](#9-design-animazioni-ed-esperienza-duso)
10. [Bilinguismo IT/EN](#10-bilinguismo-iten)
11. [Modello dati](#11-modello-dati)
12. [API](#12-api)
13. [Moderazione, sicurezza e qualità del catalogo](#13-moderazione-sicurezza-e-qualità-del-catalogo)
14. [Aspetti legali e policy degli store](#14-aspetti-legali-e-policy-degli-store)
15. [Evoluzione multipiattaforma: iOS e desktop](#15-evoluzione-multipiattaforma-ios-e-desktop)
16. [Struttura del repository](#16-struttura-del-repository)
17. [Roadmap](#17-roadmap)
18. [Rischi e mitigazioni](#18-rischi-e-mitigazioni)
19. [Decisioni aperte](#19-decisioni-aperte)

---

## 1. Visione e obiettivi

Le PWA sono app a tutti gli effetti, ma restano difficili da trovare e da installare: l'installazione dipende dal browser, il prompt è nascosto in un menu e non esiste un posto unico dove sfogliarle. **PWApp** vuole essere quel posto:

- **Catalogo comunitario**: chiunque può proporre una PWA incollando un URL.
- **Arricchimento automatico**: dall'URL il sistema ricava nome, icona, colori, descrizione, screenshot e categorie leggendo il Web App Manifest.
- **Installazione con un tocco**: dall'app Android la PWA diventa un'icona sulla home che si apre a schermo intero, come un'app nativa.
- **Modalità "URL libero"**: l'utente può creare al volo un'app da un URL qualsiasi, anche se non è nel catalogo (stile *Web Apps* di Linux Mint).
- **Registrazione semplice e gratuita**, interfaccia moderna e animata, **bilingue italiano/inglese**.
- **Futuro**: iOS e desktop (Linux, Windows, macOS) con la stessa base dati e, dove possibile, lo stesso codice.

### Obiettivi misurabili per l'MVP

| Obiettivo | Metrica |
|---|---|
| Inserire una PWA è facile | Da URL a scheda pubblicata in meno di 30 secondi, senza compilare campi a mano |
| L'installazione funziona | Oltre il 95% delle installazioni crea un'icona funzionante |
| Registrarsi costa poco | Registrazione in meno di 10 secondi (passkey o Google) |
| Catalogo pulito | Meno dell'1% di schede segnalate come rotte o malevole |

---

## 2. Attori e casi d'uso

| Attore | Cosa fa |
|---|---|
| **Visitatore** (non registrato) | Sfoglia, cerca e installa PWA; crea app da URL libero |
| **Utente registrato** | Tutto quanto sopra, più: propone PWA, recensisce, vota, crea collezioni, sincronizza le app installate tra dispositivi |
| **Sviluppatore / proprietario** | Rivendica la propria PWA verificando il dominio; modifica la scheda; abilita la modalità TWA verificata; scarica un APK firmato |
| **Moderatore** | Gestisce coda di approvazione, segnalazioni e ban |
| **Amministratore** | Configurazione, categorie, statistiche, backup |

### Casi d'uso principali

1. **Proponi una PWA**: l'utente incolla un URL → il backend analizza il sito → mostra un'anteprima già compilata → l'utente sceglie categoria e lingua e conferma → la scheda va in coda o viene pubblicata subito se i controlli automatici passano.
2. **Installa dal catalogo**: tocca "Installa" → l'app crea un'icona sulla home con l'icona della PWA → aprendola parte la PWA a schermo intero.
3. **App da URL libero**: l'utente incolla un URL qualsiasi → l'app recupera icona e nome (o li fa scegliere) → crea il collegamento. Facoltativamente può proporlo al catalogo.
4. **Rivendica la tua PWA**: lo sviluppatore dimostra di controllare il dominio (record DNS TXT, file `.well-known` o meta tag) → ottiene il badge "Verificata" e strumenti avanzati.
5. **Sincronizza**: un utente cambia telefono, accede e ritrova le sue app con un tocco.

---

## 3. Il nodo tecnico: come si "installa" una PWA da un'app di terzi

È la decisione più importante del progetto, perché Android non permette a un'app di installare *in silenzio* la PWA di un altro sito.

> **Decisione presa:** PWApp apre ogni PWA con un **motore ibrido**.
> - Se il sito ha autorizzato PWApp nel proprio `assetlinks.json`, si usa la **TWA** (Chrome): schermo intero, notifiche push, login condivisi con il browser.
> - Se il sito non l'ha fatto, si usa una **WebView a schermo intero** dentro PWApp, come fanno Brave e Firefox con le loro "app installate".
>
> Agli sviluppatori **chiediamo attivamente** di aggiungere PWApp al loro `assetlinks.json` (programma "PWA verificata", §3.6).

### 3.1 Cosa serve sapere sulla TWA

Una **Trusted Web Activity** (TWA, libreria `androidx.browser`) apre un sito dentro un'app Android usando il motore del browser installato (Chrome, Edge, Samsung Internet…). Si ottengono così service worker, notifiche push, pagamenti e cookie condivisi con il browser, senza le limitazioni di una WebView.

La condizione per avere lo **schermo intero senza barra dell'indirizzo** è che il sito dichiari di fidarsi dell'app tramite **Digital Asset Links**: il file `https://<dominio>/.well-known/assetlinks.json` deve contenere il nome del pacchetto e l'impronta SHA-256 del certificato di firma dell'app:

```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "app.pwapp",
    "sha256_cert_fingerprints": ["AB:CD:12:...:EF"]
  }
}]
```

Il file **non serve per installare**: serve solo a togliere la barra in modalità TWA. È una protezione di sicurezza, perché impedisce a un'app qualsiasi di mostrare il sito di una banca a schermo intero spacciandosi per l'app ufficiale.

Senza il file, la TWA funziona lo stesso ma ripiega in modalità *Custom Tab*, cioè con una barra in alto che mostra dominio e titolo. Per questo, quando il file manca, PWApp **non usa la TWA ma la WebView** (§3.4).

Una sola app TWA può essere verificata su **più origini**: ogni sviluppatore aggiunge la stessa voce (pacchetto `app.pwapp` e impronta di PWApp) e la sua PWA si apre a schermo intero nel launcher di PWApp.

### 3.2 Collegamento, WebAPK e APK: cosa ottiene l'utente

| | Collegamento PWApp (come Brave) | WebAPK di Chrome | APK generato (Bubblewrap) |
|---|---|---|---|
| Icona sulla home | ✅ | ✅ | ✅ |
| Nel cassetto delle app / in Impostazioni → App | ❌ | ✅ | ✅ |
| Badge sull'icona | Piccolo logo PWApp (dipende dal launcher) | Nessuno | Nessuno |
| Schermo intero su qualsiasi sito | ✅ (TWA se verificata, altrimenti WebView) | ✅ | ✅ (TWA se verificata, altrimenti WebView) |
| Notifiche push del sito | Solo con TWA | ✅ | Solo con TWA |
| Installazione con un tocco | ✅ | ✅ (conferma in Chrome) | ❌ (sorgenti sconosciute + avvisi) |
| Ammesso nella versione Play di PWApp | ✅ | ✅ | ❌ |

Solo Chrome e Samsung Internet possono generare un **WebAPK** in automatico (server di firma proprietari e installazione privilegiata). Un'altra app può ottenere una "vera app" solo generando e facendo installare un APK, con gli avvisi di sicurezza di Android.

### 3.3 Le strategie adottate

| Livello | Strategia | Quando |
|---|---|---|
| **A** | **Collegamento + launcher ibrido** (predefinito): `ShortcutManager.requestPinShortcut()` crea l'icona; l'icona apre la `LauncherActivity` di PWApp, che sceglie il motore (TWA se verificata, WebView altrimenti) | MVP |
| **B** | **PWA verificata**: il sito aggiunge PWApp al proprio `assetlinks.json`; il backend lo rileva e la scheda passa alla TWA a schermo intero, con badge "Verificata" | Rilevamento automatico nell'MVP, programma completo in Fase 2 |
| **C** | **"Installa con Chrome"**: PWApp apre l'URL in Chrome e l'utente conferma "Installa app"; Chrome genera il WebAPK | MVP, consigliato per le PWA che usano notifiche push o "Accedi con Google" |
| **D** | **APK generato con Bubblewrap** (§3.5): app indipendente, senza badge | Fase 3, solo fuori da Google Play o come strumento per gli sviluppatori |

**Logica di scelta del motore** (eseguita all'apertura e aggiornata dal backend):

```
se il dispositivo ha un browser che supporta TWA
   e l'origine è verificata (assetlinks contiene app.pwapp + nostra impronta)
      → TWA
altrimenti
      → WebView a schermo intero
```

L'app ricontrolla la verifica anche sul dispositivo (`CustomTabsSession.validateRelationship`), perché il file può cambiare tra un controllo del backend e l'altro. L'utente può forzare il motore per ogni app (TWA con barra, WebView, apri nel browser) dalle impostazioni della singola app.

**Motore sostituibile.** Il motore è isolato dietro un'interfaccia (`WebEngine`): se i test mostrano che la WebView non basta per troppe PWA, la si può sostituire con **GeckoView** (il motore di Firefox, distribuito da Mozilla come libreria) senza toccare il resto dell'app. Un fork di Chromium è stato valutato e scartato: costi di manutenzione e di sicurezza troppo alti per un team piccolo.

### 3.4 Il launcher ibrido (livello A)

**Comune a entrambi i motori**

- Si usa **una sola** `LauncherActivity` con `documentLaunchMode="intoExisting"`, oppure intent con `FLAG_ACTIVITY_NEW_DOCUMENT` e un URI diverso per ogni app. Ogni PWA appare come una scheda separata nelle app recenti, con il suo nome e il suo colore (`ActivityManager.TaskDescription`).
- La splash screen usa `background_color` e l'icona del manifest; `theme_color` colora le barre di sistema.
- Si rispettano `display` (`standalone`, `fullscreen`, `minimal-ui`) e `orientation` del manifest.
- Le icone sono generate localmente in formato **adaptive** (sfondo pieno + primo piano con margine di sicurezza). Si preferisce l'icona `purpose: maskable` quando esiste.
- Per i launcher che non supportano i collegamenti fissati (rari) si ripiega su un widget o sull'apertura dall'interno di PWApp.

**Motore TWA (siti verificati)**

- `TrustedWebActivityIntentBuilder` con `setDisplayMode`, `setScreenOrientation`, colori e splash.
- Notifiche push delegate a Chrome e mostrate con il nome della PWA (*notification delegation*).
- Login e cookie condivisi con il browser.

**Motore WebView (tutti gli altri siti)**

- `WebView` a schermo intero, edge-to-edge, con `androidx.webkit` per le funzioni più recenti.
- Attivi JavaScript, DOM storage, IndexedDB e **service worker** (`ServiceWorkerControllerCompat`), quindi funzionano cache e modalità offline.
- **Un profilo separato per ogni PWA** (Multi-Profile di `androidx.webkit`, dove supportato): cookie e dati di un sito non si mescolano con quelli degli altri, e si possono cancellare app per app.
- Navigazione limitata allo `scope` del manifest; i link esterni si aprono nel browser predefinito (`shouldOverrideUrlLoading`).
- Gestione completa di: caricamento file (`onShowFileChooser`), permessi (fotocamera, microfono, posizione) con le richieste di sistema Android, download tramite `DownloadManager`, video a schermo intero (`onShowCustomView`), tema scuro, tasto indietro (cronologia della WebView, poi chiusura), trascina per aggiornare facoltativo.
- Sicurezza: nessun `addJavascriptInterface`, accesso a `file://` e `content://` disattivato, Safe Browsing della WebView attivo, solo HTTPS.
- **Limiti noti, mostrati all'utente nella scheda dell'app:**
  - niente **notifiche push** web (la WebView non le supporta);
  - **login separato** da quello di Chrome;
  - i flussi **"Accedi con Google"** sono bloccati da Google nelle WebView (errore `disallowed_useragent`).

  Per le PWA dove questi limiti pesano (rilevati dal worker o segnalati dagli utenti) la scheda propone in primo piano **"Installa con Chrome"** (livello C).

### 3.5 Bubblewrap in dettaglio (livello D)

[Bubblewrap](https://github.com/GoogleChromeLabs/bubblewrap) è la CLI di Google Chrome Labs che, partendo dal Web App Manifest, genera un progetto Android TWA completo e lo compila in APK/AAB firmato. È la stessa tecnologia usata da PWABuilder per Android. In PWApp diventa un **servizio di build nel progetto Docker**.

**Flusso**

```
richiesta build ─▶ coda "builds" ─▶ container apk-builder
   1. bubblewrap init --manifest=<url manifest>   (non interattivo, valori da twa-manifest.json generato dal backend)
   2. personalizzazione: packageId, nome, colori, icone maskable, splash, shortcuts,
      notification delegation, fallback (customtabs | webview), display mode
   3. bubblewrap build  → app-release-signed.apk + app-release-bundle.aab
   4. upload artefatti su MinIO + log di build ─▶ notifica all'utente
```

- **Immagine Docker `builder/`**: JDK 17 + Android SDK command-line tools + build-tools + `@bubblewrap/cli`. Cache di Gradle e dell'SDK su un volume per scendere da diversi minuti a circa 1 minuto per build. Concorrenza limitata (1–2 build parallele per core) e timeout rigido.
- **`twa-manifest.json` generato dal backend** a partire dai dati già normalizzati della scheda (§6). L'utente non deve compilare nulla.
- **Package name** deterministico: `app.pwapp.u.<slug>` per le build personali, e quello scelto dallo sviluppatore per le build dei proprietari.
- **Firma**:
  - *Build del proprietario*: lo sviluppatore carica la propria keystore (cifrata a riposo, usata solo nel container e poi rimossa), oppure scarica l'AAB non firmato e lo firma con Play App Signing. Riceve inoltre il frammento `assetlinks.json` già pronto con l'impronta corretta, così l'app è a **schermo intero vero**.
  - *Build personale per un utente*: firmata con una chiave PWApp dedicata (separata da quella dell'app principale). Usa il **modello WebView** (sotto), quindi è a schermo intero anche senza `assetlinks.json`: l'utente ottiene un'**app indipendente** nel cassetto, con nome e icona propri, senza badge e senza dipendere da PWApp.
- **Due modelli di progetto**:
  - **Modello TWA** (quello standard di Bubblewrap): per le build dei proprietari e per i siti che hanno autorizzato il pacchetto specifico nel proprio `assetlinks.json`. Ogni APK ha un nome di pacchetto diverso, quindi serve una voce per ciascuno.
  - **Modello WebView** (nostro, generato dallo stesso servizio con un template Gradle dedicato che riusa il motore WebView del launcher, §3.4): per le build personali di siti non verificati. Ha gli stessi limiti della WebView: niente push, login separato.
  - L'opzione `fallbackType: webview` di Bubblewrap **non** basta: interviene solo quando sul dispositivo manca un browser compatibile con TWA, non quando manca la verifica del sito.
- **Distribuzione**: download dalla web app o dal profilo utente. Su Android l'installazione richiede che l'utente abiliti "Installa app sconosciute" per il browser o il file manager: l'app Play di PWApp **non** chiama l'installer, e il permesso `REQUEST_INSTALL_PACKAGES` resta solo nel flavor distribuito fuori Play (§14).
- **Aggiornamenti**: quando cambia il manifest (ricontrollo periodico §6), il backend può proporre una nuova build con `versionCode` incrementato. Le build personali firmate dalla stessa chiave si aggiornano sopra la versione installata.
- **Rischi specifici**: tempi e costi di CPU (servono quote per utente, ad esempio 3 build al giorno); verifica sviluppatori Android dal 2026, per cui gli APK firmati da PWApp per siti di terzi possono essere bloccati sui dispositivi certificati nei paesi dove la regola è attiva. Per questo le build personali sono classificate come **funzione sperimentale**, mentre le build per i proprietari sono il caso d'uso principale.

**Uso di Bubblewrap anche per l'app PWApp stessa (opzionale).** Il catalogo web è una PWA (§9.4), quindi Bubblewrap può impacchettarlo in pochi minuti come TWA verificata. Può servire come **prototipo rapido** da mettere sul Play Store prima che l'app nativa Compose sia pronta. Va però considerato un ponte: le funzioni chiave (collegamenti fissati, share target nativo, Credential Manager, launcher multi-task) richiedono codice nativo.


### 3.6 Programma "PWA verificata" (assetlinks.json)

Obiettivo: far sì che il maggior numero possibile di PWA del catalogo passi dalla WebView alla TWA a schermo intero.

- **Rilevamento automatico** (già nell'MVP): a ogni analisi e ricontrollo (§6) il worker scarica `https://<origine>/.well-known/assetlinks.json` e controlla se contiene il pacchetto `app.pwapp` con l'impronta della nostra chiave di firma. Se sì, la scheda diventa `twa_verified = true` e l'app passa alla TWA alla prossima apertura, **anche se il proprietario non si è mai registrato**.
- **Pagina "Per sviluppatori"** bilingue sul sito, con:
  - spiegazione semplice dei vantaggi (schermo intero, push, badge "Verificata", posizione migliore nei risultati);
  - il **frammento JSON pronto da copiare**, già compilato con pacchetto e impronta;
  - istruzioni per i casi più comuni: file già esistente da integrare (si aggiunge un elemento all'array), hosting statico (Netlify, Vercel, GitHub Pages, Cloudflare Pages), server Nginx/Apache, framework (Next.js, SvelteKit, Angular, WordPress);
  - pulsante **"Verifica ora"** che lancia il controllo e mostra l'esito, con gli errori più comuni spiegati (file non trovato, `Content-Type` diverso da `application/json`, redirect, impronta sbagliata, JSON non valido).
- **Richiesta attiva**:
  - nella scheda di ogni app non verificata, un pulsante **"Sei lo sviluppatore?"** porta alla procedura;
  - quando un'app raggiunge una certa popolarità, PWApp può contattare lo sviluppatore tramite i canali pubblici indicati nel sito o nel manifest (email di contatto, repository), con un messaggio breve e senza insistere;
  - gli utenti possono "chiedere la verifica" con un tocco; il contatore di richieste rende la proposta più convincente per lo sviluppatore.
- **Vantaggi per chi aderisce**: badge "Verificata", modifica della scheda, statistiche di installazione, accesso alle build Bubblewrap (§3.5) e, in futuro, notifiche agli utenti quando esce una nuova versione.
- **Rischio da gestire**: se cambiamo la chiave di firma dell'app, tutte le verifiche si perdono. La chiave va custodita con cura (Play App Signing + backup) e la pagina sviluppatori può indicare in anticipo anche l'impronta di una chiave di riserva, accettata insieme a quella principale.

---

## 4. Architettura complessiva

```
                   ┌───────────────────────────────────────────────┐
                   │                 Client                        │
                   │  Android (Kotlin/Compose)   Web (PWA catalogo) │
                   │  [futuro] iOS · Desktop (Compose Multiplatform)│
                   └───────────────┬───────────────────────────────┘
                                   │ HTTPS / JSON (OpenAPI v1)
┌──────────────────────────────────▼───────────────────────────────────┐
│ Docker Compose                                                       │
│                                                                      │
│  ┌─────────┐   ┌──────────────┐   ┌──────────────┐  ┌─────────────┐  │
│  │  Caddy  │──▶│   API        │──▶│ PostgreSQL   │  │ Meilisearch │  │
│  │ (TLS,   │   │ (TypeScript, │   │ (dati)       │  │ (ricerca)   │  │
│  │  proxy) │   │  Fastify)    │──▶└──────────────┘  └─────────────┘  │
│  └────┬────┘   └──────┬───────┘                                      │
│       │               │ code (BullMQ)                                │
│       │        ┌──────▼───────┐   ┌──────────────┐  ┌─────────────┐  │
│       │        │    Redis     │◀──│ Worker       │─▶│ MinIO (S3)  │  │
│       │        │ code/cache   │   │ scraper +    │  │ icone,      │  │
│       │        └──────────────┘   │ Playwright   │  │ screenshot, │  │
│       │                           └──────────────┘  │ build APK   │  │
│       │                           ┌──────────────┐  └─────────────┘  │
│       └──────────────────────────▶│ Web frontend │                   │
│                                   │ (SvelteKit)  │  [Fase 3]         │
│                                   └──────────────┘  Worker Bubblewrap│
└──────────────────────────────────────────────────────────────────────┘
```

Principi:

- **Un unico backend per tutti i client**: Android, web, iOS e desktop usano le stesse API.
- **Lavori lenti fuori dalla richiesta**: analisi degli URL, screenshot, controlli periodici e build APK girano su worker asincroni con code.
- **Tutto self-hostable** con un `docker compose up`, senza dipendenze cloud obbligatorie. I servizi esterni (Google Safe Browsing, invio email, traduzione) sono opzionali e configurabili.

---

## 5. Backend (progetto Docker)

### 5.1 Stack consigliato

| Componente | Scelta | Motivazione |
|---|---|---|
| Linguaggio | **TypeScript (Node.js 22 LTS)** | Ecosistema web nativo per analizzare HTML e manifest; Playwright e Bubblewrap sono strumenti Node; stesso linguaggio del frontend web |
| Framework HTTP | **Fastify** (oppure NestJS se il team preferisce una struttura rigida) | Veloce, validazione con JSON Schema, generazione OpenAPI |
| ORM / migrazioni | **Drizzle ORM** | Tipizzato, SQL esplicito, migrazioni versionate |
| Database | **PostgreSQL 17** | Relazionale, JSONB per i manifest, `pg_trgm` per la ricerca approssimata |
| Ricerca | **Meilisearch** | Ricerca istantanea, tollerante ai refusi, multilingua, facile in Docker |
| Code e cache | **Redis 7 + BullMQ** | Job con retry, scheduling, rate limiting |
| Storage oggetti | **MinIO** (S3 compatibile) | Icone, screenshot, artefatti APK; in produzione si può passare a qualsiasi S3 |
| Browser headless | **Playwright + Chromium** | Verifica service worker e installabilità, screenshot |
| Reverse proxy | **Caddy** | HTTPS automatico con Let's Encrypt, HTTP/3 |
| Auth | **Better Auth** (libreria TS self-hosted) | Passkey, magic link, OAuth Google/Apple, sessioni; nessun servizio esterno |
| Email | SMTP configurabile (es. Postmark, SES, o Mailpit in sviluppo) | Magic link e notifiche |
| Osservabilità | OpenTelemetry → Grafana/Loki/Prometheus (profilo opzionale) | Log, metriche, tracing |

**Alternativa valutata**: backend in **Kotlin + Ktor**, per condividere i modelli con l'app via Kotlin Multiplatform. È sensata se il team è solo Kotlin, ma si perde l'integrazione nativa con Playwright e Bubblewrap e con il frontend web. La condivisione dei contratti si ottiene comunque generando il client Kotlin dallo **schema OpenAPI**.

### 5.2 `docker-compose.yml` (bozza)

```yaml
services:
  caddy:
    image: caddy:2
    ports: ["80:80", "443:443"]
    volumes: [./deploy/Caddyfile:/etc/caddy/Caddyfile, caddy_data:/data]
    depends_on: [api, web]

  api:
    build: ./backend
    command: node dist/server.js
    env_file: .env
    depends_on: [postgres, redis, meilisearch, minio]

  worker:
    build: ./backend
    command: node dist/worker.js
    env_file: .env
    shm_size: 1gb            # necessario per Chromium
    depends_on: [redis, postgres, minio]

  web:
    build: ./web
    env_file: .env

  postgres:
    image: postgres:17
    environment: { POSTGRES_DB: pwapp, POSTGRES_USER: pwapp, POSTGRES_PASSWORD: ${DB_PASSWORD} }
    volumes: [pg_data:/var/lib/postgresql/data]

  redis:
    image: redis:7-alpine
    volumes: [redis_data:/data]

  meilisearch:
    image: getmeili/meilisearch:v1.11
    environment: { MEILI_MASTER_KEY: ${MEILI_KEY} }
    volumes: [meili_data:/meili_data]

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    volumes: [minio_data:/data]

  # Fase 3 – build APK via Bubblewrap (JDK + Android SDK)
  # apk-builder:
  #   build: ./builder
  #   profiles: ["builder"]

volumes: { caddy_data: {}, pg_data: {}, redis_data: {}, meili_data: {}, minio_data: {} }
```

Ambienti: `compose.dev.yml` aggiunge Mailpit, hot reload e porte esposte; `compose.prod.yml` aggiunge backup automatici di Postgres (es. `pgbackrest` o dump notturno su S3) e limiti di risorse.

### 5.3 Moduli dell'API

- `auth`: registrazione, login, sessioni, token per l'app mobile
- `apps`: schede PWA, versioni del manifest, screenshot
- `submissions`: invio URL, anteprima, stato della coda
- `ingest`: orchestrazione dei job di analisi
- `catalog`: home, categorie, classifiche, collezioni editoriali
- `search`: indicizzazione e query su Meilisearch
- `social`: recensioni, voti, preferiti, segnalazioni
- `owners`: rivendicazione dominio, verifica `assetlinks.json`, statistiche per sviluppatori
- `sync`: libreria personale dell'utente (app installate su ogni dispositivo)
- `admin`: moderazione, ban, audit log
- `builds` (Fase 3): richieste di build APK, artefatti, firme

---

## 6. Pipeline di acquisizione di una PWA dall'URL

Il job `ingest` trasforma un URL in una scheda completa. Gira nel worker ed è idempotente.

```
URL ─▶ normalizza ─▶ controlli sicurezza ─▶ fetch HTML ─▶ trova manifest ─▶ fetch manifest
   ─▶ valida e normalizza ─▶ scarica icone/screenshot ─▶ verifica SW (headless) ─▶ punteggio
   ─▶ deduplica ─▶ salva bozza ─▶ (auto-pubblica | coda moderazione) ─▶ indicizza
```

### 6.1 Passi

1. **Normalizzazione**: forza `https`, rimuove i parametri di tracciamento (`utm_*`, `fbclid`…), segue i redirect (massimo 5), calcola l'**origine**.
2. **Sicurezza (anti-SSRF)**: risolve il DNS e rifiuta IP privati, loopback, link-local e metadata cloud (169.254.169.254); blocca porte non standard; applica timeout (10 s) e limite di dimensione (HTML 2 MB, immagini 5 MB). Le richieste partono da un egress dedicato con user agent dichiarato (`PWAppBot/1.0 (+https://pwapp.example/bot)`).
3. **HTML**: estrae `<link rel="manifest">`, `<title>`, `<meta name="description">`, `theme-color`, Open Graph (`og:image`, `og:description`), `apple-touch-icon` e favicon come ripiego.
4. **Manifest**: legge `id`, `name`, `short_name`, `description`, `start_url`, `scope`, `display`, `display_override`, `theme_color`, `background_color`, `icons` (con `purpose` e `sizes`), `screenshots` (con `form_factor` e `label`), `categories`, `shortcuts`, `lang`, `dir`, `orientation`. Risolve gli URL relativi rispetto all'URL del manifest, come prevede la specifica.
5. **Asset**: scarica le icone, sceglie la migliore (preferendo maskable ≥ 512 px, poi any ≥ 192 px, poi SVG), genera le varianti (48/96/192/512 in WebP + PNG), calcola il **colore dominante** se manca `theme_color` ed estrae un **BlurHash** per i segnaposti animati.
6. **Verifica di installabilità (headless)**: Playwright apre la pagina e controlla che un service worker sia registrato e attivo, che lo `start_url` risponda 200 anche offline (facoltativo) e che non ci siano errori bloccanti. Se `screenshots` manca, cattura uno screenshot mobile (412×915) e uno desktop.
7. **Punteggio di qualità** (0–100). La categoria "PWA" di Lighthouse è stata rimossa nel 2024, quindi usiamo criteri nostri: HTTPS, manifest valido, icona ≥ 512, icona maskable, service worker, offline, screenshot, descrizione, `id` stabile, display standalone, tempo di risposta.
8. **Deduplica**: chiave primaria logica = `manifest.id` risolto, altrimenti `start_url` normalizzato. Se esiste già, si propone di aggiornare la scheda esistente invece di crearne una nuova.
9. **Pubblicazione**: si pubblica subito se il punteggio è ≥ soglia, il dominio non è in blacklist, Safe Browsing (opzionale) è pulito e l'utente ha una reputazione sufficiente. Altrimenti la scheda va in **coda di moderazione**.
10. **Ricontrollo periodico**: ogni 7 giorni (con backoff per le app stabili) si rilegge il manifest per aggiornare icone e nomi e per segnalare le app rotte (3 fallimenti consecutivi → nascosta).

### 6.2 Anteprima sincrona

Per un'esperienza "magica" durante l'invio, l'endpoint `POST /v1/submissions/preview` esegue in modo sincrono i passi 1–5 in versione leggera (timeout 8 s), così l'utente vede subito icona e nome con un'animazione. I passi pesanti (6–7) proseguono in background, e lo stato arriva in tempo reale via **SSE** o polling.

---

## 7. App Android nativa

### 7.1 Stack

| Area | Scelta |
|---|---|
| Linguaggio | Kotlin 2.x |
| UI | **Jetpack Compose** + **Material 3 Expressive** |
| Architettura | MVVM/MVI con flusso dati unidirezionale; moduli Gradle per feature |
| Codice condiviso | Modulo **Kotlin Multiplatform** `shared` (modelli, client API, repository, logica), già pronto per iOS e desktop |
| Rete | Ktor Client + kotlinx.serialization (client generato da OpenAPI) |
| Persistenza | Room (KMP) per cache offline e libreria locale; DataStore per le preferenze |
| DI | Koin (compatibile KMP) |
| Immagini | Coil 3 (KMP) |
| Navigazione | Navigation Compose con route type-safe |
| Background | WorkManager (sync libreria, aggiornamento icone dei collegamenti) |
| Motori web | `androidx.browser` per la TWA (TrustedWebActivityIntentBuilder, CustomTabsClient); `WebView` + `androidx.webkit` per i siti non verificati; interfaccia `WebEngine` per poter passare a GeckoView |
| Auth | **Credential Manager** (passkey, Sign in with Google, password salvate) |
| Notifiche | FCM opzionale (es. "una tua proposta è stata approvata") |
| Qualità | Detekt, ktlint, test con Turbine, screenshot test con Roborazzi/Paparazzi, Baseline Profiles |

**minSdk 26** (Android 8, richiesto dai collegamenti fissati di `ShortcutManager`), **targetSdk** all'ultima versione richiesta da Play.

### 7.2 Schermate

1. **Onboarding** (3 pagine animate, saltabili): cos'è una PWA, come funziona l'installazione, scelta della lingua.
2. **Home / Scopri**: in evidenza (carosello con parallasse), "Nuove", "Di tendenza", categorie in chip, collezioni editoriali.
3. **Cerca**: ricerca istantanea con suggerimenti, filtri (categoria, offline, verificata, lingua).
4. **Dettaglio app**: header con colore estratto dall'icona, screenshot in galleria, punteggio qualità, recensioni, pulsante "Installa" con morph animato (Installa → progresso → Apri).
5. **Aggiungi**: campo URL con incolla dagli appunti e ricezione da "Condividi" (`ACTION_SEND` da qualsiasi browser). Anteprima live, poi due scelte: "Installa solo per me" (URL libero) e "Proponi al catalogo".
6. **Le mie app**: le PWA installate, con riordino, modifica icona e nome, rimozione del collegamento.
7. **Profilo**: proposte inviate e loro stato, recensioni, collezioni, preferenze (lingua, tema, motore di apertura predefinito: automatico, WebView o browser).
8. **Area sviluppatore** (Fase 2): rivendicazione, istruzioni `assetlinks.json`, statistiche.

### 7.3 Intent e integrazioni

- **Share target**: PWApp compare nel menu "Condividi" di Chrome/Firefox → "Crea app da questa pagina".
- **Deep link** verificati (`https://pwapp.example/app/{slug}`) per aprire le schede dal web.
- **Link interni della PWA**: TWA e WebView restano nello `scope` del manifest; i link esterni si aprono nel browser.
- **Scorciatoie del manifest**: i `shortcuts` della PWA diventano shortcut dinamici del collegamento quando possibile.

---

## 8. Registrazione e autenticazione

Obiettivo: **zero attrito, zero costi per l'utente**, nessuna password obbligatoria.

| Metodo | Note |
|---|---|
| **Passkey** (consigliato) | Un tocco con impronta o volto; su Android tramite Credential Manager, sul web con WebAuthn |
| **Sign in with Google** | Un tocco su Android; su iOS si aggiungerà Sign in with Apple, obbligatorio su App Store se si offrono login social |
| **Magic link / codice OTP via email** | Ripiego universale, senza password |
| **Password** | Facoltativa, per chi la preferisce |

- **Navigazione e installazione senza account**: l'account serve solo per proporre, recensire e sincronizzare. In questo modo la registrazione non blocca l'uso dell'app.
- **Account progressivo**: la libreria locale di chi non è registrato viene unita all'account al primo login.
- Sessioni: token di accesso brevi (15 min) + refresh token ruotati, salvati in `EncryptedSharedPreferences`/Keystore. Sul web si usano cookie `HttpOnly`, `Secure`, `SameSite=Lax`.
- Protezione anti-abuso: rate limit per IP e per account, **Cloudflare Turnstile** o hCaptcha solo sui flussi sospetti, Play Integrity facoltativo.
- **GDPR**: consenso esplicito, esportazione dei dati, cancellazione dell'account self-service.

---

## 9. Design, animazioni ed esperienza d'uso

### 9.1 Linguaggio visivo

- **Material 3 Expressive**: forme morbide e variabili, tipografia espressiva, colori vivaci, componenti come il pulsante "Installa" che cambia forma.
- **Colore dinamico**: Material You sul dispositivo (Android 12+) e, nelle pagine di dettaglio, **tema derivato dall'icona della PWA** (Palette API → schema tonale M3). Ogni app "colora" la propria scheda.
- **Tema chiaro/scuro/automatico**, edge-to-edge, supporto ai tablet e ai pieghevoli (layout adattivi con `NavigationSuiteScaffold` e list-detail).
- **Icone**: set Material Symbols (variabile, animabile).
- **Font**: un sans moderno con buon supporto latino (es. *Inter*, *Plus Jakarta Sans* o *Google Sans Flex* se disponibile), più un display per i titoli.

### 9.2 Animazioni (con misura)

| Momento | Animazione |
|---|---|
| Da lista a dettaglio | **Shared element transition**: l'icona vola dalla card all'header |
| Installazione | Il pulsante si trasforma in anello di progresso, poi in "Apri" con una spunta animata; leggero feedback aptico |
| Anteprima da URL | L'icona compare da un segnaposto BlurHash con dissolvenza + scala a molla |
| Liste | Comparsa sfalsata degli elementi; `animateItem()` per riordino e rimozione |
| Carosello in evidenza | Parallasse e transizione di colore dello sfondo |
| Indietro | **Predictive back** con anteprima animata |
| Stati vuoti / onboarding | Illustrazioni animate leggere (Lottie o animated vector drawable) |

Regole: durate tra 150 e 400 ms, curve a **molla** (spring) invece di easing lineari, nessuna animazione che blocchi l'interazione. Si rispetta l'impostazione di sistema **"Rimuovi animazioni"** (`Settings.Global.ANIMATOR_DURATION_SCALE`).

### 9.3 Accessibilità

Contrasto AA, target touch ≥ 48 dp, `contentDescription` su tutte le immagini (incluse le icone delle PWA, con il loro nome), supporto TalkBack e caratteri grandi, nessuna informazione veicolata solo dal colore.

### 9.4 Web

Il catalogo web (SvelteKit oppure Astro) è a sua volta una **PWA**: installabile, offline per le pagine visitate, con **View Transitions API** per le stesse transizioni dell'app. Serve anche alla SEO, perché ogni scheda ha una pagina indicizzabile con dati strutturati `SoftwareApplication`.

---

## 10. Bilinguismo IT/EN

### 10.1 Interfaccia

- Android: `res/values/strings.xml` (inglese, predefinito) + `res/values-it/strings.xml`. Si dichiara `locales_config.xml` per la **lingua per-app** (Android 13+), sceglibile anche dalle impostazioni di PWApp. Plurali con `plurals`, formati con ICU.
- Web: Paraglide JS (SvelteKit) o equivalente, con URL localizzati (`/it/...`, `/en/...`) e `hreflang`.
- Backend: messaggi di errore con codice stabile (`SUBMISSION_DUPLICATE`), tradotti dal client. Email in due lingue secondo la preferenza dell'utente.
- Le traduzioni stanno in un unico posto (es. file JSON/XLIFF nel repo), con controllo in CI che segnala le chiavi mancanti in una delle due lingue.

### 10.2 Contenuti

- I campi testuali di una scheda (`name`, `description`) sono **per lingua** (`app_translations`).
- Origine dei testi, in ordine di priorità: (1) testo inserito dal proprietario verificato, (2) manifest nella lingua corrispondente (le PWA multilingua possono servire manifest diversi in base a `Accept-Language`: il worker li richiede entrambi), (3) testo proposto dall'utente, (4) **traduzione automatica** opzionale (LibreTranslate self-hosted in Docker, oppure DeepL via API), marcata come "tradotta automaticamente".
- Ricerca: Meilisearch indicizza entrambe le lingue; si cerca nella lingua dell'utente con fallback sull'altra.
- Il modello dati è pronto per altre lingue in futuro, senza vincoli su due.

---

## 11. Modello dati

Tabelle principali (PostgreSQL):

```
users(id, email, display_name, avatar_url, locale, role[user|moderator|admin],
      reputation, created_at, deleted_at)
passkeys / oauth_accounts / sessions            -- gestite da Better Auth

apps(id, slug, origin, manifest_id, start_url, scope, manifest_url,
     display, orientation, theme_color, background_color,
     icon_key, icon_maskable_key, blurhash,
     quality_score, has_service_worker, works_offline,
     status[draft|pending|published|hidden|rejected|broken],
     verified_owner_id NULL, twa_verified bool, assetlinks_checked_at,
     webview_issues jsonb,        -- push, login Google, ecc. → suggerisce "Installa con Chrome"
     submitted_by, created_at, updated_at, last_checked_at, check_failures)

app_translations(app_id, locale, name, short_name, description, source[owner|manifest|user|machine])
app_screenshots(id, app_id, storage_key, form_factor[narrow|wide], label, sort)
app_manifests(id, app_id, fetched_at, raw jsonb, hash)       -- storico versioni
categories(id, slug, icon) + category_translations(category_id, locale, name)
app_categories(app_id, category_id)

submissions(id, url, user_id, app_id NULL, status, error_code, preview jsonb, created_at)
reviews(id, app_id, user_id, rating 1..5, body, locale, created_at, hidden)
favorites(user_id, app_id)
collections(id, owner_id NULL, slug, is_editorial) + collection_items + collection_translations
library_entries(user_id, app_id NULL, custom_url NULL, custom_name, custom_icon_key,
                device_id, launch_mode[twa|browser|webview], created_at)
reports(id, app_id, user_id, reason, note, status, created_at)
domain_claims(id, origin, user_id, method[dns|wellknown|meta], token, verified_at)
install_events(app_id, day, platform, count)                 -- aggregati, anonimi
builds(id, app_id, requested_by, package_name, status, artifact_key, log_key, created_at)  -- Fase 3
audit_log(id, actor_id, action, target_type, target_id, data jsonb, created_at)
```

Note:
- `origin` + `manifest_id` hanno un indice univoco: è la regola di deduplica.
- Le statistiche d'installazione si raccolgono **aggregate per giorno**, senza ID dispositivo, per ridurre gli obblighi GDPR.
- Le immagini vivono su S3/MinIO e vengono servite tramite Caddy/CDN con cache lunga e nomi basati sul contenuto (hash).

---

## 12. API

REST JSON versionata (`/v1`), documentata con **OpenAPI 3.1**, da cui si generano i client Kotlin e TypeScript. Paginazione a cursore e header `Accept-Language` per i contenuti.

| Metodo | Endpoint | Descrizione |
|---|---|---|
| GET | `/v1/home` | Sezioni della home (in evidenza, nuove, trend, collezioni) |
| GET | `/v1/apps?category=&sort=&cursor=` | Elenco |
| GET | `/v1/apps/{slug}` | Dettaglio |
| GET | `/v1/apps/{slug}/launch` | Parametri di lancio (URL, scope, display, colori, icone, `twa_verified`, motore consigliato, limiti noti in WebView) |
| POST | `/v1/apps/{slug}/verify-assetlinks` | Ricontrolla subito l'`assetlinks.json` del sito (pulsante "Verifica ora") |
| POST | `/v1/apps/{slug}/verification-requests` | Un utente chiede allo sviluppatore di verificare la PWA |
| GET | `/v1/search?q=` | Ricerca |
| GET | `/v1/categories` | Categorie |
| POST | `/v1/submissions/preview` | Anteprima sincrona da URL (rate-limited, anche anonima) |
| POST | `/v1/submissions` | Proposta al catalogo (auth) |
| GET | `/v1/submissions/{id}` · `/events` (SSE) | Stato della proposta |
| POST | `/v1/apps/{slug}/reviews` · `/favorite` · `/report` | Interazioni (auth) |
| GET/PUT/DELETE | `/v1/me/library` | Libreria sincronizzata |
| POST | `/v1/apps/{slug}/install-event` | Contatore anonimo |
| POST | `/v1/claims` · `/v1/claims/{id}/verify` | Rivendicazione dominio |
| POST | `/v1/apps/{slug}/builds` | Build APK (Fase 3, solo proprietari) |
| * | `/v1/admin/...` | Moderazione |
| GET | `/.well-known/assetlinks.json` | Asset links del dominio PWApp (per deep link e per la TWA del catalogo stesso) |

---

## 13. Moderazione, sicurezza e qualità del catalogo

**Controlli automatici sulle proposte**
- Validità tecnica (manifest, HTTPS, raggiungibilità).
- Google **Safe Browsing / Web Risk** (opzionale), blacklist di domini, euristiche anti-phishing (nomi che imitano brand noti su domini diversi).
- Limite di proposte al giorno per account; reputazione crescente con le proposte approvate.

**Moderazione umana**
- Coda con anteprima, diff tra le versioni del manifest, azioni rapide (approva, rifiuta con motivo, unisci duplicato).
- Segnalazioni degli utenti con motivazioni predefinite (rotta, malevola, contenuto illecito, violazione di marchio, duplicato).
- Audit log di ogni azione.

**Sicurezza applicativa**
- Anti-SSRF nel worker (§6.1); worker in rete Docker isolata, senza accesso a Postgres diretto se non tramite le API interne (o con utente DB con privilegi minimi).
- Chromium headless in sandbox, contesto nuovo per ogni job, nessun cookie persistente, limiti di CPU/RAM/tempo.
- Le immagini scaricate vengono **ricodificate** (sharp/libvips) prima di essere servite: niente SVG "attivi" serviti così come sono (si sanitizzano o si rasterizzano).
- Rate limiting su Caddy e sull'API; header di sicurezza (CSP, HSTS).
- Dipendenze controllate (Renovate + `npm audit`/OSV), immagini Docker scansionate (Trivy) in CI.
- Lato Android: la TWA è sicura per costruzione (gira nel browser); per la WebView valgono le regole del §3.4 (nessun JS bridge, niente `file://`, Safe Browsing attivo, profilo separato per ogni PWA).

---

## 14. Aspetti legali e policy degli store

| Tema | Analisi / azione |
|---|---|
| **GDPR** (utenti UE, titolare in Italia) | Informativa privacy, base giuridica, registro dei trattamenti, minimizzazione (statistiche aggregate), DPA con i fornitori (email, hosting), diritto all'oblio |
| **Digital Services Act** | PWApp ospita contenuti generati dagli utenti: servono un meccanismo di segnalazione ("notice and action"), motivazione delle rimozioni, punto di contatto e termini chiari |
| **Cookie / ePrivacy** | Sul web solo cookie tecnici di sessione; nessun tracciamento di terze parti, quindi niente banner invasivo |
| **Marchi e icone** | Nomi e icone appartengono ai titolari delle PWA: vanno mostrati a scopo descrittivo, con procedura di rimozione su richiesta (stile DMCA/DSA) e possibilità di rivendicazione |
| **Google Play – app WebView/wrapper** | PWApp è un catalogo/launcher, non il "wrapper" di un singolo sito altrui: va descritto con chiarezza la funzione di directory. Poiché i siti non verificati si aprono nella nostra WebView, la scheda Play deve chiarire che sono siti di terzi aperti su richiesta dell'utente, mostrando sempre l'origine (es. nella schermata di caricamento e nel menu dell'app) |
| **Google Play – `REQUEST_INSTALL_PACKAGES`** | Permesso consentito solo per casi d'uso specifici: **non** lo usiamo nella build Play. Il livello D (APK) si consegna ai proprietari, oppure si offre in un flavor separato distribuito fuori Play |
| **Verifica sviluppatori Android (dal 2026)** | Google richiede che anche le app installate fuori dal Play Store provengano da sviluppatori verificati, con introduzione graduale per paese da settembre 2026. Per il livello D questo significa che l'APK va firmato e registrato dallo **sviluppatore della PWA**, non da PWApp |
| **App Store (iOS)** | La regola 4.2 (funzionalità minima) e le linee guida sulle directory di app vanno valutate: l'app iOS deve offrire valore proprio (catalogo, collezioni, sincronizzazione), non solo link |
| **Termini d'uso** | Età minima, contenuti vietati, licenza sui contenuti inviati (recensioni, screenshot), limitazione di responsabilità per le app di terzi |

---

## 15. Evoluzione multipiattaforma: iOS e desktop

### 15.1 Strategia di codice

- Fin dal giorno 1 il modulo `shared` (KMP) contiene modelli, client API, repository, cache e logica di presentazione (ViewModel KMP).
- La UI in **Compose Multiplatform** può essere riusata su iOS (stabile dal 2025) e desktop (JVM). Per ogni piattaforma restano nativi solo il modulo **installer** (come si crea l'app) e le integrazioni di sistema.
- Interfaccia comune:

```kotlin
interface PwaInstaller {
    suspend fun capabilities(): InstallCapabilities          // cosa supporta questa piattaforma
    suspend fun install(app: LaunchSpec, mode: InstallMode): InstallResult
    suspend fun uninstall(id: String): Boolean
    suspend fun installed(): List<InstalledApp>
}
```

### 15.2 iOS

Su iOS **nessuna app può installare una web app sulla home in modo programmatico**, e tutti i browser usano WebKit. Le opzioni:

| Opzione | Note |
|---|---|
| **Guida "Aggiungi a Home"** (predefinita) | L'app apre la PWA in Safari (`SFSafariViewController` non basta: serve Safari vero) e mostra un tutorial animato. Da iOS 26 i siti aggiunti alla Home si aprono come web app per impostazione predefinita, quindi il risultato è buono |
| **Profilo di configurazione Web Clip** (`.mobileconfig`) | Il backend genera un profilo con payload *Web Clip* (URL, icona, nome, `FullScreen`). L'utente lo scarica e lo installa da Impostazioni. Crea un'icona a schermo intero in pochi passaggi; va firmato per evitare l'avviso "non verificato"; è adatto a utenti esperti |
| **Apertura interna** | Libreria "Le mie app" dentro PWApp con apertura in `WKWebView` a schermo intero. Funziona ma non crea icone sulla home ed è limitato dalla regola 4.2 |
| **UE / DMA** | Il supporto alle web app nella UE resta quello standard via Safari; si monitorano le evoluzioni sui motori browser alternativi |

### 15.3 Desktop (Linux, Windows, macOS)

L'ispirazione è **Web Apps di Linux Mint** (`webapp-manager`), che crea file `.desktop` per lanciare un sito in un browser con profilo dedicato e senza interfaccia.

| Piattaforma | Meccanismo |
|---|---|
| **Linux** | Si genera un `~/.local/share/applications/pwapp-<slug>.desktop` con icona in `~/.local/share/icons`, che lancia `chromium --app=<url> --class=pwapp-<slug> --user-data-dir=<profilo>` (oppure Firefox con profilo dedicato, Epiphany/GNOME Web in modalità web app). `StartupWMClass` corretto per raggruppare le finestre. Distribuzione come Flatpak (con portali per i permessi) o AppImage/.deb |
| **Windows** | Collegamento `.lnk` nel menu Start che lancia `msedge --app=<url>` o Chrome con `--app`, oppure apertura della pagina con il pulsante "Installa" del browser. Distribuzione con MSIX/winget |
| **macOS** | Safari "Aggiungi al Dock" (macOS 14+) guidato, oppure generazione di un piccolo bundle `.app` che lancia Chrome/Edge in modalità `--app`. Distribuzione con .dmg notarizzato o Homebrew cask |
| **Web (tutte)** | Il catalogo web può sfruttare la **Web Install API** (`navigator.install()`), in sperimentazione su Chromium/Edge, che permette a un sito di avviare l'installazione di una PWA di un'altra origine. Da adottare come miglioramento progressivo quando sarà disponibile in modo stabile |

App desktop: **Compose Multiplatform Desktop** (riusa `shared` e la UI), con un modulo installer per ogni sistema operativo. In alternativa più leggera: **Tauri 2** con frontend web condiviso con il catalogo.

---

## 16. Struttura del repository

Monorepo:

```
PWApp/
├── backend/            # API + worker (TypeScript, Fastify, Drizzle, BullMQ)
│   ├── src/modules/…
│   ├── src/worker/ingest/…
│   └── Dockerfile
├── web/                # Catalogo web PWA (SvelteKit)
├── mobile/             # Progetto Gradle KMP
│   ├── shared/         # modelli, API client, repository, ViewModel
│   ├── androidApp/     # UI Compose, launcher ibrido (TWA/WebView), installer Android
│   ├── iosApp/         # (futuro)
│   └── desktopApp/     # (futuro)
├── builder/            # (Fase 3) immagine Bubblewrap + Android SDK
├── api/openapi.yaml    # contratto unico → client generati
├── i18n/               # stringhe condivise / glossario IT-EN
├── deploy/             # Caddyfile, compose.prod.yml, script di backup
├── docs/               # questo documento, ADR, guide
├── docker-compose.yml
└── .github/workflows/  # CI: lint, test, build immagini, build APK, Trivy
```

CI/CD (GitHub Actions): test e lint per ogni modulo; build delle immagini Docker pubblicate su GHCR; build dell'AAB firmato e caricamento sul canale *internal testing* di Play; controllo che lo schema OpenAPI e i client generati siano allineati.

---

## 17. Roadmap

| Fase | Durata indicativa | Contenuto |
|---|---|---|
| **0 – Fondamenta** | 2 settimane | Monorepo, Docker Compose, CI, schema DB, OpenAPI iniziale, design system (token colori, tipografia, componenti base) in Figma e Compose |
| **1 – MVP** | 8–10 settimane | Pipeline di acquisizione (§6), catalogo, ricerca, dettaglio, launcher ibrido TWA/WebView (livello **A**), rilevamento automatico di `assetlinks.json` (livello **B**), "Installa con Chrome" (livello **C**), URL libero e share target, registrazione (passkey, Google, magic link), proposte con moderazione base, IT/EN completo, animazioni principali. Rilascio in beta chiusa su Play |
| **2 – Comunità e sviluppatori** | 6 settimane | Recensioni, preferiti, collezioni, sincronizzazione della libreria, rivendicazione dominio, **programma "PWA verificata" completo** (§3.6: pagina sviluppatori, "Verifica ora", richieste degli utenti), statistiche per sviluppatori, catalogo web pubblico SEO, ricontrolli periodici |
| **3 – Build e qualità** | 4–6 settimane | Servizio di build APK/AAB con Bubblewrap (**livello D**): modello TWA per i proprietari, modello WebView per le build personali fuori Play; valutazione di GeckoView se la WebView si è rivelata limitante; traduzione automatica, osservabilità completa, hardening |
| **4 – Desktop** | 6 settimane | App Compose Desktop con installer Linux (.desktop / Flatpak), Windows e macOS |
| **5 – iOS** | 6–8 settimane | App iOS con UI condivisa, guida "Aggiungi a Home" e profili Web Clip, Sign in with Apple |

---

## 18. Rischi e mitigazioni

| Rischio | Impatto | Mitigazione |
|---|---|---|
| Pochi siti aggiungono PWApp al loro `assetlinks.json` | Medio | La WebView garantisce comunque lo schermo intero; programma "PWA verificata" con vantaggi concreti e frammento pronto da copiare (§3.6) |
| Limiti della WebView (niente push, login separato, "Accedi con Google" bloccato) | Medio | Avvisi nella scheda dell'app; "Installa con Chrome" in primo piano per le app colpite; motore sostituibile con GeckoView |
| Cambio della chiave di firma dell'app | Alto | Invalida tutte le verifiche: Play App Signing, backup sicuro, impronta di riserva pubblicata in anticipo |
| Badge dell'app PWApp sulle icone fissate | Basso | Limite di Android; per chi lo vuole evitare, consigliare il WebAPK |
| Rifiuto o sospensione su Google Play (policy wrapper / install packages) | Alto | Niente `REQUEST_INSTALL_PACKAGES` nella build Play; scheda dello store che descrive il catalogo; valore proprio (ricerca, recensioni, sincronizzazione) |
| Spam e phishing nel catalogo | Alto | Controlli automatici, reputazione, moderazione, Safe Browsing, segnalazioni |
| SSRF e abuso del worker come proxy | Alto | Validazione IP post-DNS, rete isolata, limiti di dimensione e tempo, rate limit |
| Siti che bloccano il bot o richiedono login | Medio | Ripiego su favicon/OG; modifica manuale da parte dell'utente; proprietario verificato |
| Cambi di politica di browser e OS (iOS, verifica sviluppatori Android) | Medio | Architettura a installer intercambiabili; monitoraggio; strategie multiple per piattaforma |
| Costi di Chromium headless su larga scala | Medio | Coda con concorrenza limitata, cache dei risultati, analisi completa solo all'invio e poi ricontrolli leggeri (solo HTTP) |
| Qualità delle traduzioni | Basso | Priorità ai testi del proprietario; etichetta "tradotto automaticamente"; correzioni dalla comunità |

---

## 19. Decisioni aperte

Da chiudere prima della Fase 1:

1. **Nome e dominio definitivi** (PWApp? verificare disponibilità del marchio e del dominio).
2. **Backend TypeScript vs Kotlin/Ktor** (raccomandato: TypeScript, §5.1).
3. **Frontend web: SvelteKit vs Next.js vs Astro** (raccomandato: SvelteKit, per prestazioni e View Transitions semplici).
4. **Pubblicazione automatica vs sempre moderata** all'avvio (raccomandato: sempre moderata finché la comunità non è avviata, poi automatica per gli utenti con reputazione).
5. **Hosting di produzione**: singola VPS con Docker Compose (sufficiente per MVP) oppure Kubernetes più avanti.
6. **Distribuzione del flavor con build APK fuori Play** (sì/no) alla luce della verifica sviluppatori Android.
7. **Monetizzazione** futura compatibile con la gratuità (es. app in evidenza segnalate come sponsorizzate, piano "pro" per sviluppatori con statistiche avanzate e build), oppure progetto open source senza monetizzazione.

---

*Questo documento è un'analisi preliminare: le versioni delle librerie e le policy degli store vanno ricontrollate all'avvio di ciascuna fase.*
