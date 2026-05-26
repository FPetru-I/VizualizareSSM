## CUPRINS

1. [Prezentare generală](#1-prezentare-generală)
2. [Arhitectură și flux de date](#2-arhitectură-și-flux-de-date)
3. [Structura HTML](#3-structura-html)
4. [CSS — Clase și variabile](#4-css--clase-și-variabile)
5. [Configurare și constante](#5-configurare-și-constante)
6. [Variabile globale](#6-variabile-globale)
7. [Autentificare](#7-autentificare)
8. [Navigare între tab-uri](#8-navigare-între-tab-uri)
9. [Tab Rapoarte PDF](#9-tab-rapoarte-pdf)
10. [Cache localStorage](#10-cache-localstorage)
11. [Prefetch la hover](#11-prefetch-la-hover)
12. [Indicator progres PDF](#12-indicator-progres-pdf)
13. [Tab Centralizator CSV](#13-tab-centralizator-csv)
14. [Referință elemente DOM](#14-referință-elemente-dom)
15. [Referință completă funcții](#15-referință-completă-funcții)
16. [Deployment și configurare](#16-deployment-și-configurare)

\---

## 1\. Prezentare generală

`rapoarte.html` este o aplicație single-page (SPA) pentru vizualizarea și descărcarea rapoartelor SSM stocate într-un repository GitHub privat. Pagina poate fi hostată pe orice serviciu static (GitHub Pages, Netlify etc.) și comunică exclusiv prin Google Apps Script (GAS) care deține tokenul GitHub.

**Funcționalități principale:**

* Autentificare cu parolă validată server-side prin GAS
* Sesiune persistentă în `sessionStorage` cu expirare automată la miezul nopții
* Listare, căutare și sortare rapoarte PDF
* Previzualizare PDF inline cu indicator de progres animat
* Descărcare PDF
* Prefetch automat la hover (300ms delay) — PDF-ul începe să se descarce înainte de click
* Cache în `localStorage` (max 4MB) — rapoartele vizualizate apar instant la re-accesare
* Indicator `●` verde lângă rapoartele deja în cache
* Tab Centralizator CSV cu tabel sortabil, căutare globală și export `.csv`

\---

## 2\. Arhitectură și flux de date

```
┌──────────────────────┐         ┌───────────────────────┐        ┌─────────────────┐
│   rapoarte.html      │         │  Google Apps Script   │        │  GitHub API     │
│  (GitHub Pages /     │◄───────►│  (GAS Proxy)          │◄──────►│  repo privat    │
│   Netlify)           │  HTTPS  │  - login/token        │  HTTPS │  RapoarteSSM    │
└──────────────────────┘         │  - listare fișiere    │        └─────────────────┘
                                  │  - servire PDF binar  │
                                  │  - servire CSV        │
                                  └───────────────────────┘
```

**De ce GAS ca proxy:**
Repository-ul GitHub este privat — browserul nu poate accesa direct fișierele fără token. GAS deține tokenul GitHub și îl folosește server-side, expunând doar un API protejat prin parolă proprie. Tokenul GitHub nu apare niciodată în codul frontend.

**Fluxul complet al unui request PDF:**

```
1. Browser → GAS: ?action=pdf\&filename=R1\_...pdf\&token=XXX
2. GAS → GitHub API: GET /contents/rapoarte/R1\_...pdf
   (Authorization: Bearer GITHUB\_TOKEN)
3. GitHub → GAS: metadata cu download\_url
4. GAS → GitHub: GET download\_url (fișier binar)
5. GAS → Browser: JSON { status:'success', base64:'...' }
6. Browser: atob(base64) → Uint8Array → Blob → iframe src (blob URL)
```

**Fluxul autentificării:**

```
1. Browser → GAS: ?action=login\&password=parola
2. GAS: verifică parola cu VIEWER\_PASSWORD
3. GAS: dacă corect → token = base64(SESSION\_SECRET + ':' + today)
4. GAS → Browser: { status:'success', token:'...' }
5. Browser: salvează token în sessionStorage\['ssm\_token']
6. Token expiră automat la miezul nopții (data zilei se schimbă)
```

\---

## 3\. Structura HTML

```
<head>
  ├── Variabile CSS (:root)
  ├── Stiluri toate componentele
  └── Link Google Fonts (IBM Plex Sans + IBM Plex Mono)

<body>
  ├── #loginScreen              — ecran login (vizibil la start dacă nu e token)
  │     └── .login-card
  │           ├── .login-logo   — icon scut + titlu "Rapoarte SSM"
  │           ├── input#passwordInput  — câmp parolă (Enter = submit)
  │           ├── button#loginBtn      — buton "Intră"
  │           └── #loginError          — mesaj eroare (ascuns implicit)
  │
  └── #appScreen                — aplicația principală (ascunsă până la login)
        ├── .topbar             — bara de sus albastră
        │     ├── .topbar-left  — icon + "Rapoarte SSM" / "Bază de date centralizată"
        │     └── .topbar-right
        │           ├── .status-pill    — indicator status server (●)
        │           └── button.logout-btn — buton "Ieși"
        │
        ├── .tabs               — tab-uri navigare
        │     ├── #tab-rapoarte — "Rapoarte PDF" (activ implicit)
        │     └── #tab-csv      — "Centralizator CSV"
        │
        └── .main
              ├── #content-rapoarte (.tab-content.active)
              │     ├── .sidebar
              │     │     ├── .sidebar-toolbar
              │     │     │     ├── .search-wrap > #searchInput
              │     │     │     └── .toolbar-row
              │     │     │           ├── #sortSelect
              │     │     │           └── #countBadge
              │     │     └── #fileList   — lista rapoartelor
              │     │
              │     └── .preview
              │           ├── .preview-toolbar
              │           │     ├── #previewFilename
              │           │     └── #btnDownload
              │           ├── .preview-body
              │           │     ├── #previewEmpty    — stare goală
              │           │     ├── #previewLoading  — stare încărcare
              │           │     │     ├── #previewProgressPct  — "42%"
              │           │     │     ├── #progressFill        — bara animată
              │           │     │     └── #previewProgress     — numele fișierului
              │           │     ├── #previewError    — stare eroare
              │           │     └── #pdfFrame        — iframe PDF
              │           └── #statsBar   — Nr | Data | Localitate
              │
              └── #content-csv (.tab-content)
                    └── .csv-panel
                          ├── .csv-toolbar
                          │     ├── #csvSearch
                          │     ├── #btnRefreshCsv
                          │     ├── #btnExportCsv   — dezactivat până la încărcare
                          │     └── #csvCountBadge
                          ├── #csvTableWrap   — tabelul CSV
                          └── #csvStatus      — "N înregistrări · M coloane"
```

\---

## 4\. CSS — Clase și variabile

### Variabile CSS (:root)

```css
--bg            #f4f3ef   /\* fundal pagină \*/
--surface       #fff      /\* fundal carduri, toolbar-uri \*/
--surface2      #f9f8f5   /\* fundal câmpuri input, rânduri pare tabel \*/
--border        #dddbd4   /\* borduri normale \*/
--border-strong #b8b5ac   /\* borduri accentuate \*/
--text          #1a1917   /\* text principal \*/
--text-muted    #6b6860   /\* text secundar \*/
--text-faint    #9b9890   /\* text dezactivat, placeholder \*/
--accent        #1a4d8f   /\* albastru principal (topbar, butoane, focus) \*/
--accent-light  #e8eef7   /\* albastru deschis (hover file-item) \*/
--accent-dark   #0f3060   /\* albastru închis (hover butoane primare) \*/
--yes           #1a6b3c   /\* verde (badge DA, indicator cache ●) \*/
--yes-bg        #eaf5ee   /\* fundal verde deschis \*/
--danger        #c0392b   /\* roșu (erori, badge NU) \*/
--radius        8px       /\* raza colțuri rotunjite \*/
```

### Clase principale — Login

|Clasă|Descriere|
|-|-|
|`.login-card`|Cardul de login centrat, max 360px, shadow|
|`.login-logo`|Container icon + titlu|
|`.login-logo-icon`|Pătrat albastru cu icon scut|
|`.login-input`|Input parolă cu focus border albastru|
|`.login-btn`|Buton "Intră" full-width, albastru|
|`.login-error`|Mesaj eroare roșu (ascuns implicit, `display:none`)|

### Clase principale — App

|Clasă|Descriere|
|-|-|
|`.topbar`|Bara de navigare sus, înălțime 52px, albastru|
|`.status-pill`|Pastilă cu indicator ● și text status|
|`.status-dot`|Punct colorat: verde (ok), galben animat (loading), roșu (error)|
|`.status-dot.loading`|Pulsație CSS (`@keyframes pulse`)|
|`.logout-btn`|Buton "Ieși" transparent cu bordură albă|
|`.tabs`|Container tab-uri, fundal alb, bordură jos|
|`.tab`|Tab individual cu `border-bottom: 2px solid transparent`|
|`.tab.active`|Tab activ: text albastru + `border-bottom: var(--accent)`|
|`.tab-content`|Container conținut tab (`display:none` implicit)|
|`.tab-content.active`|Tab-ul activ curent (`display:flex`)|

### Clase principale — Sidebar și File List

|Clasă|Descriere|
|-|-|
|`.sidebar`|Panou stânga, lățime 340px, border dreapta|
|`.sidebar-toolbar`|Zona search + sort, padding 12px|
|`.search-wrap`|Container input cu icon lupă poziționat absolut|
|`.file-item`|Rând raport în listă, border transparent|
|`.file-item:hover`|Fundal accent-light + bordură albastră deschis|
|`.file-item.active`|Fundal accent-light + bordură albastră + box-shadow inset|
|`.file-item.prefetching::after`|Bară albastră 2px jos (în timp ce se prefetch-ează)|
|`.file-icon`|Pătrat 32px, fundal albastru deschis, icon PDF|
|`.file-item.active .file-icon`|Fundal albastru solid, icon alb|
|`.file-name`|Numele fișierului, font monospaced, truncat cu ellipsis|
|`.file-meta`|Data + localitate, font 11px, muted|
|`.state-msg`|Mesaj centrat (loading/empty/error) cu spinner sau icon|
|`.spinner`|Cerc animat de încărcare (`@keyframes spin`)|

### Clase principale — Preview

|Clasă|Descriere|
|-|-|
|`.preview`|Panoul drept, flex:1, ocupă spațiul rămas|
|`.preview-toolbar`|Bara de sus 48px: filename + buton Descarcă|
|`.preview-filename`|Numele fișierului curent, monospaced, truncat|
|`.preview-filename.empty`|Starea goală: text muted, font normal|
|`.preview-body`|Container central, poziție relativă (pentru overlay-uri)|
|`.preview-empty`|Stare: niciun raport selectat|
|`.preview-loading`|Overlay loading cu spinner + progress bar|
|`.preview-loading.show`|Afișat (`display:flex`)|
|`.preview-error`|Overlay eroare cu buton "Încearcă din nou"|
|`.preview-error.show`|Afișat (`display:flex`)|
|`.pdf-frame`|iframe PDF, `display:none` implicit|
|`.pdf-frame.show`|iframe vizibil (`display:block`)|
|`.stats-bar`|Bara de jos 32px: Nr|
|`.pdf-progress-bar`|Container bara de progres, 200px, 4px înălțime|
|`.pdf-progress-fill`|Umplerea barei, `transition: width 0.2s ease`|
|`.pdf-progress-pct`|Text procentaj "42%", font monospaced albastru|

### Clase principale — CSV

|Clasă|Descriere|
|-|-|
|`.csv-panel`|Container tab CSV, flex coloană|
|`.csv-toolbar`|Bara unelte: search + Reîncarcă + Export CSV|
|`.csv-search`|Input căutare CSV|
|`.csv-table-wrap`|Container scrollabil pentru tabel (overflow:auto)|
|`.csv-table`|Tabelul de date, `border-collapse:collapse`|
|`.csv-table th`|Header sticky (`position:sticky; top:0`), sortabil la click|
|`.csv-table th.sort-asc`|Adaugă ` ↑` după text prin `::after`|
|`.csv-table th.sort-desc`|Adaugă ` ↓` după text|
|`.csv-table tr:nth-child(even) td`|Rânduri pare cu fundal surface2|
|`.badge-da`|Pastilă verde "DA"|
|`.badge-nu`|Pastilă roșie "NU"|
|`.badge-na`|Pastilă albastru-gri "N/A"|
|`.csv-status`|Bara de jos CSV: "N înregistrări · M coloane"|

### Media queries

```css
@media (max-width: 680px) {
  .rapoarte-layout { flex-direction: column; }  /\* sidebar deasupra preview \*/
  .sidebar { width: 100%; border-bottom: 1px solid var(--border); }
  .file-list { max-height: 260px; }             /\* scroll în loc de full-height \*/
  .preview { height: 50vh; }
  .topbar { padding: 0 14px; }
}
```

\---

## 5\. Configurare și constante

```javascript
// ── Endpoint GAS — singura configurare necesară ──
const GAS\_URL = 'https://script.google.com/macros/s/DEPLOYMENT\_ID/exec';
// Înlocuiește DEPLOYMENT\_ID cu cel din GAS → Deploy → Manage deployments

// ── Cache localStorage ──
const LS\_PREFIX = 'ssm\_pdf\_';
// Prefixul cheilor din localStorage pentru PDF-uri
// Ex: localStorage\['ssm\_pdf\_R1\_Test\_2026-05-19.pdf'] = 'base64string...'

const LS\_MAX\_MB = 4;
// Limita totală a cache-ului în megabytes
// Când se depășește, fișierele cele mai vechi sunt șterse (FIFO)
// Valoare conservativă față de limita browser \~5-10MB
```

\---

## 6\. Variabile globale

```javascript
// ── Declarate la nivel de modul (scope script) ──

let allFiles = \[]
// Array cu toate rapoartele returnate de GAS la loadRapoarte()
// Structura unui element:
// { name: "R4\_Bucuresti\_2026-05-20.pdf", nr: "R4",
//   location: "Bucuresti", date: "2026-05-20", sha: "abc123..." }

let currentFile = null
// Raportul selectat curent (obiect din allFiles\[])
// null = niciun raport selectat

let currentBlob = null
// PDF-ul curent ca obiect Blob în memorie
// null = PDF-ul nu a fost încă descărcat în această sesiune
// Resetat la null la fiecare selectFile() pentru a forța re-verificarea cache

let sessionToken = null
// Tokenul de sesiune obținut de la GAS după login reușit
// Salvat și în sessionStorage\['ssm\_token'] pentru persistența între refresh-uri
// Trimis ca parametru \&token= la toate requesturile autentificate

let csvLoaded = false
// Flag: CSV-ul a fost încărcat cel puțin o dată în sesiunea curentă
// Previne re-încărcarea automată la fiecare switch pe tab-ul CSV

let prefetchTimer = null
// ID-ul timerului setTimeout pentru prefetch la hover
// Stocat pentru a fi anulat cu clearTimeout la ieșirea rapidă din hover

let prefetching = new Set()
// Set cu numele fișierelor în curs de prefetch în background
// Previne requesturi duplicate dacă utilizatorul hover-uiește repetat

let csvData = \[]
// Array de array-uri cu datele CSV (fără rândul header)
// Structura: \[ \["R1", "Ion", "ion@ppc.com", ...], \["R2", ...], ... ]

let csvHeaders = \[]
// Array cu numele coloanelor (primul rând din CSV)
// Ex: \["Nr Raport", "Nume", "Email", "Data", "Localitate", ...]

let csvSortCol = -1
// Indexul coloanei după care e sortat tabelul (-1 = nesortrat)

let csvSortDir = 1
// Direcția de sortare: 1 = ascendent (A→Z), -1 = descendent (Z→A)
```

\---

## 7\. Autentificare

### `init()` — IIFE executată automat

```
La încărcarea paginii:
1. Citește sessionStorage\['ssm\_token']
2. Dacă tokenul există → showApp() + loadRapoarte() (sare peste login)
3. Dacă nu există → showLogin()
```

### `showLogin()`

```
- Afișează #loginScreen (display:flex)
- Ascunde #appScreen (display:none)
- Focus automat pe #passwordInput după 100ms
```

### `showApp()`

```
- Ascunde #loginScreen
- Afișează #appScreen (display:flex)
```

### `doLogin()`

```
Apelat la click pe "Intră" sau Enter în câmpul parolă.

1. Citește parola din #passwordInput
2. Dacă câmpul e gol → return (fără acțiune)
3. Dezactivează butonul ("Se verifică..."), ascunde eroarea anterioară
4. GET: GAS\_URL?action=login\&password=PAROLA\_ENCODATA
5. Dacă răspuns { status: 'success', token: '...' }:
   → salvează token în sessionToken și sessionStorage
   → showApp() + loadRapoarte()
6. Dacă eroare:
   → afișează mesajul în #loginError
   → șterge câmpul parolă și pune focus pe el
7. Reactivează butonul
```

### `doLogout()`

```
Apelat la click pe "Ieși" sau la expirarea tokenului (răspuns 'Neautorizat.').

1. Șterge sessionStorage\['ssm\_token']
2. Resetează: sessionToken=null, allFiles=\[], currentFile=null, currentBlob=null
3. Șterge câmpul parolă
4. showLogin()

Notă: cache-ul localStorage NU este șters la logout.
PDF-urile cached rămân disponibile pentru sesiunile viitoare.
```

### Expirarea automată a tokenului

Tokenul GAS este generat ca:

```javascript
Utilities.base64Encode(SESSION\_SECRET + ':' + today)
// today = 'YYYY-MM-DD'
```

La miezul nopții, `today` se schimbă → tokenul salvat în sessionStorage nu mai coincide cu cel calculat de GAS → orice request returnează `{ status:'error', message:'Neautorizat.' }` → `doLogout()` este apelat automat.

\---

## 8\. Navigare între tab-uri

### `switchTab(tab)`

```
Parametri: tab = 'rapoarte' | 'csv'

1. Elimină clasa 'active' de pe toate .tab și .tab-content
2. Adaugă 'active' pe #tab-{tab} și #content-{tab}
3. Dacă tab === 'csv' \&\& !csvLoaded → loadCsv()
   (CSV-ul se încarcă o singură dată, la prima accesare a tab-ului)
```

**Starea vizuală a tab-urilor:**

* Tab activ: `color: var(--accent)` + `border-bottom: 2px solid var(--accent)`
* Tab inactiv: `color: var(--text-muted)` + `border-bottom: transparent`

\---

## 9\. Tab Rapoarte PDF

### `loadRapoarte()`

```
Apelat după login reușit sau la refresh manual.

1. setStatus('loading', 'Se conectează...')
2. GET: GAS\_URL?action=rapoarte\&token=TOKEN
3. GAS returnează: { status:'success', files: \[ {name, nr, location, date, sha}, ... ] }
4. allFiles = data.files (sortate de GAS după dată, descrescător)
5. setStatus('ok', 'N rapoarte')
6. filterFiles() — randează lista

Erori gestionate:
- 'Neautorizat.' → doLogout()
- Orice altă eroare → setStatus('error', ...) + mesaj în #fileList
```

**Structura unui obiect fișier:**

```javascript
{
  name:     "R4\_Bucuresti\_2026-05-20.pdf",  // numele complet al fișierului
  nr:       "R4",                            // numărul raportului
  location: "Bucuresti",                     // localitate (spații în loc de -)
  date:     "2026-05-20",                   // format ISO YYYY-MM-DD
  sha:      "abc123..."                      // SHA-ul fișierului în GitHub
}
```

### `setStatus(type, label)`

```
Actualizează indicatorul din topbar:
type 'loading' → .status-dot.loading (galben pulsator)
type 'error'   → .status-dot.error   (roșu)
altceva        → .status-dot          (verde)
label          → textul din #statusLabel
```

### `filterFiles()`

```
Filtrează allFiles\[] după query-ul din #searchInput și sortează.

Căutare în câmpurile: name, location, nr, date (toate case-insensitive)
Sortare:
  'newest'  → date descrescător (b.date.localeCompare(a.date))
  'oldest'  → date crescător
  'name'    → alfabetic după name

Actualizează:
  #countBadge → "X / Y rapoarte"
  #fileList   → renderList(list)
```

### `renderList(files)`

```
Generează HTML pentru lista din sidebar.

Dacă lista e goală → mesaj "Niciun raport găsit."
Altfel, per fișier:
  - Verifică dacă e cached: isCached(f.name) → indicator ● verde
  - onclick="selectFile(...)" cu datele fișierului serializate JSON
  - onmouseenter="prefetchOnHover(filename)" — prefetch la hover
  - Afișează: icon PDF + nume fișier + data formatată + localitate
  - Clasa 'active' dacă currentFile.name === f.name

Notă de securitate: JSON.stringify(f).replace(/"/g, '\&quot;') previne
XSS la serializarea numelui fișierului în atributul onclick.
```

### `selectFile(f)`

```
Apelat la click pe un fișier din listă.

1. currentFile = f; currentBlob = null
   (resetarea blob-ului forțează re-verificarea cache-ului)
2. Actualizează UI:
   - #previewFilename → f.name
   - #btnDownload → enabled
   - #statsBar → vizibil: f.nr | formatDate(f.date) | f.location
3. filterFiles() — refresh lista pentru marcarea 'active'
4. previewCurrent() — declanșează automat previzualizarea
```

### `showPreviewState(state)`

```
Comută între stările panoului de previzualizare:
state 'empty'   → #previewEmpty vizibil
state 'loading' → #previewLoading.show vizibil (overlay peste iframe)
state 'error'   → #previewError.show vizibil
state null      → toate ascunse (PDF-ul iframe e vizibil)
```

### `previewCurrent()`

```
Logica principală de afișare a unui PDF. 3 niveluri de cache:

NIVEL 1 — In-memory:
  Dacă currentBlob există → showPdfBlob(currentBlob) imediat (0ms)

NIVEL 2 — localStorage:
  loadFromCache(currentFile.name) → dacă există
  → base64ToBlob(cached) → showPdfBlob() (câteva ms)

NIVEL 3 — Download de la GAS:
  showPreviewState('loading')
  Pornește animație progres simulat (setInterval 200ms):
    simPct += (85 - simPct) × 0.08  // exponențial, se oprește la \~85%
  fetchPDF(currentFile.name, null)
  La succes:
    clearInterval → 100% → saveToCache → showPdfBlob → filterFiles (refresh ●)
  La eroare:
    clearInterval → showPreviewState('error')
```

### `fetchPDF(filename, onProgress)`

```
Parametri:
  filename   — numele exact al fișierului
  onProgress — callback(pct) sau null (neutilizat curent, progress e simulat)

1. Construiește URL: GAS\_URL?action=pdf\&filename=...\&token=...
2. fetch(url)
3. Citește răspunsul via ReadableStream (res.body.getReader()):
   - Acumulează chunks\[] în timp real
   - Dacă onProgress \&\& Content-Length → progres real (%)
   - Dacă onProgress \&\& fără Content-Length → estimare din bytes primiți
4. Asamblează toate chunk-urile într-un singur Uint8Array
5. Decodează ca text JSON (răspunsul GAS e întotdeauna JSON)
6. Verifică data.status:
   'Neautorizat.' → doLogout(); return null
   altă eroare   → return null
7. Returnează data.base64 (string base64 al PDF-ului binar)

Notă: Content-Length nu e prezent în răspunsurile GAS → progresul real
din ReadableStream va fi întotdeauna 0→100 fără pași intermediari.
De aceea previewCurrent() folosește o animație simulată separată.
```

### `showPdfBlob(blob)`

```
1. URL.createObjectURL(blob) → URL temporar de tip blob:...
2. pdfFrame.src = objUrl
3. pdfFrame.className = 'pdf-frame show' (face iframe vizibil)
4. showPreviewState(null) — ascunde toate overlay-urile

Notă: blob URL-urile sunt valide doar pe durata sesiunii.
La reload, iframe-ul trebuie re-populat.
```

### `downloadCurrent()`

```
Apelat la click pe "Descarcă".

Dacă currentBlob există (PDF deja în memorie sau cache):
  → Creează <a> temporar cu href=blob URL + download=filename → click()

Dacă nu:
  → previewCurrent() (descarcă și previzualizează)
  → La finalizare: dacă currentBlob → descarcă
```

### `formatDate(d)`

```
Convertește 'YYYY-MM-DD' → 'DD/MM/YYYY' pentru afișare
Ex: '2026-05-20' → '20/05/2026'
Dacă d este undefined/null → '—'
```

### `isNew(d)`

```
Returnează true dacă data d este în ultimele 14 zile față de acum.
(Funcție prezentă în cod dar badge-ul "Nou" a fost eliminat din renderList)
```

\---

## 10\. Cache localStorage

### Scop

Rapoartele PDF descărcate sunt salvate în `localStorage` pentru a fi disponibile instant la vizitele ulterioare, fără re-descărcare de la GAS.

### Structura cheilor

```
localStorage\['ssm\_pdf\_R1\_Test\_2026-05-19.pdf']     = "JVBERi0xLjQ..."
localStorage\['ssm\_pdf\_R4\_Bucuresti\_2026-05-20.pdf'] = "JVBERi0xLjQ..."
```

Prefix definit în `LS\_PREFIX = 'ssm\_pdf\_'`.

### `getCacheKey(filename)`

```
Returnează: LS\_PREFIX + filename
Ex: 'ssm\_pdf\_R1\_Test\_2026-05-19.pdf'
```

### `isCached(filename)`

```
Returnează: boolean
true dacă localStorage conține cheia pentru fișierul respectiv
Folosit în: renderList() (indicator ●), prefetchOnHover(), previewCurrent()
```

### `saveToCache(filename, base64)`

```
Salvează PDF-ul în localStorage cu management automat de spațiu:

1. Calculează dimensiunea noului fișier în MB:
   sizeMB = (base64.length × 0.75) / (1024 × 1024)
   (base64 e cu 33% mai mare decât binar → × 0.75 = dimensiunea reală)

2. Calculează spațiul total ocupat de toate cheile cu prefix LS\_PREFIX
   și construiește lista {cheie, dimensiune} sortată în ordinea adăugării

3. Cât timp (totalMB + sizeMB > LS\_MAX\_MB) \&\& mai sunt chei:
   → Șterge cea mai veche cheie (FIFO)
   → Scade din totalMB

4. localStorage.setItem(key, base64)

Erori (localStorage plin): prinse silențios cu try/catch
```

### `loadFromCache(filename)`

```
Returnează: string base64 sau null
Citește din localStorage; returnează null dacă cheia nu există sau la eroare
```

### `base64ToBlob(base64)`

```
Convertește string base64 în Blob PDF:
1. atob(base64) → string binar
2. new Uint8Array(binary.length) → umple cu charCodeAt()
3. new Blob(\[bytes], { type: 'application/pdf' })
```

### Indicator vizual în listă

Rapoartele cache-uite afișează `●` verde lângă nume:

```javascript
`${f.name}${cached ? ' <span style="color:var(--yes)">●</span>' : ''}`
```

Lista se re-randează după fiecare descărcare/prefetch pentru a actualiza indicatorii.

\---

## 11\. Prefetch la hover

### Mecanism

Când cursorul intră pe un `file-item` (`onmouseenter`), se apelează `prefetchOnHover(filename)`.

### `prefetchOnHover(filename)`

```
1. Dacă fișierul e deja în cache SAU în curs de prefetch → return imediat
2. clearTimeout(prefetchTimer) — anulează prefetch-ul anterior (scroll rapid)
3. setTimeout(() => { ... }, 300):
   — Dacă după 300ms cursorul e încă pe element:
   — Adaugă filename în Set-ul 'prefetching'
   — fetchPDF(filename, null) în background (fără UI loading)
   — La succes: saveToCache → prefetching.delete → filterFiles (refresh ●)
   — La eroare: prefetching.delete (permite re-încercare)
```

### De ce 300ms

Evită descărcări inutile la scroll rapid prin listă. La 300ms de hover, intenția utilizatorului de a accesa fișierul respectiv este clară.

### Indicatorul vizual de prefetch

Clasa `.prefetching` pe `.file-item` adaugă o bară albastră subțire (2px) la baza elementului prin `::after`:

```css
.file-item.prefetching::after {
  content: '';
  height: 2px;
  background: var(--accent);
  opacity: 0.4;
  /\* ... \*/
}
```

*Notă: clasa `.prefetching` e definită în CSS dar nu e aplicată dinamic în codul curent. Bara de progres a prefetch-ului nu e vizibilă în prezent.*

\---

## 12\. Indicator progres PDF

### Problema tehnică

GAS returnează răspunsul ca JSON (nu stream binar direct), fără header `Content-Length`. Prin urmare, `ReadableStream` nu poate calcula un procent real din bytes primiți vs. total.

### Soluția: animație simulată exponențială

```javascript
let simPct = 0;
const simInterval = setInterval(() => {
  const remaining = 85 - simPct;
  simPct += remaining \* 0.08;  // 8% din restul rămas
  // ...actualizează UI...
}, 200);  // la fiecare 200ms
```

**Comportamentul matematic:**

* Pornește rapid: 0% → 30% în primele 2 secunde
* Încetinește progresiv spre 85% (limita superioară)
* Nu ajunge niciodată la 85% singur — se oprește la \~83-84%
* La finalizarea requestului: salt instant la 100%

**Efectul vizual:** bara avansează natural, sugerând activitate continuă fără a promite o durată.

### Elementele UI implicate

|Element ID|Tip|Conținut|
|-|-|-|
|`previewProgressPct`|`<span>`|Text "42%"|
|`progressFill`|`<div>`|Lățime animată CSS (`transition: width 0.2s ease`)|
|`previewProgress`|`<span>`|Numele fișierului în curs de încărcare|

\---

## 13\. Tab Centralizator CSV

### `loadCsv()`

```
Apelat automat la prima accesare a tab-ului CSV sau manual (buton Reîncarcă).

1. Afișează spinner de încărcare în #csvTableWrap
2. GET: GAS\_URL?action=csv\&token=TOKEN
3. GAS returnează: { status:'success', csv: 'text CSV complet' }
4. parseCSV(data.csv) → rows\[]\[]
5. csvHeaders = rows\[0]  (primul rând = header)
6. csvData    = rows.slice(1)  (restul = date)
7. csvLoaded  = true
8. renderCsv(csvData) — randează tabelul
9. #btnExportCsv.disabled = false — activează butonul de export
10. Actualizează statusul: "N înregistrări · M coloane"
```

### `parseCSV(text)`

```
Parser CSV complet, gestionează:
- Câmpuri normale: valoare,valoare
- Câmpuri cu virgule: "valoare, cu virgulă"
- Ghilimele escaped: "valoare cu ""ghilimele"""
- Câmpuri multilinie (newline în interiorul ghilimelelor)
- Linii goale (ignorate)

Returnează: Array<Array<string>> — matrice bidimensională
```

### `filterCsv()`

```
Filtrează csvData\[] după query-ul din #csvSearch.
Căutare case-insensitive în TOATE celulele fiecărui rând.
Actualizează #csvCountBadge: "X / Y rânduri"
Apelează renderCsv(filtered)
```

### `renderCsv(rows)`

```
Generează HTML pentru tabelul CSV.

Header (<thead>):
  - Per coloană: <th onclick="sortCsv(i)"> cu cls sort-asc/sort-desc
  - Header sticky (position:sticky; top:0) — rămâne vizibil la scroll

Corpul (<tbody>):
  - Per rând, per celulă: verifică valoarea
    'DA'  → <span class="badge-da">DA</span>   (verde)
    'NU'  → <span class="badge-nu">NU</span>   (roșu)
    'N/A' → <span class="badge-na">N/A</span>  (albastru-gri)
    altceva → text simplu
  - title="{valoare}" pe <td> pentru tooltip la celule trunchiate
```

### `sortCsv(colIdx)`

```
Toggle pe aceeași coloană: csvSortDir \*= -1 (inversează direcția)
Coloană nouă: csvSortCol = colIdx; csvSortDir = 1 (ascendent)

Sortare cu localeCompare(bv, undefined, {numeric: true}):
  numeric:true → R2 < R10 (numeric), nu R10 < R2 (lexicografic)

Re-aplică filtrul curent din #csvSearch înainte de sortare.
renderCsv(rows) cu datele sortate.
```

### `exportCsv()`

```
Descarcă toate datele (nu cele filtrate) ca fișier CSV.

1. Verifică că există date (csvHeaders + csvData non-goale)
2. escVal(v) per celulă: escapează " ca "", înconjoară cu " dacă conține , sau "
3. Construiește text CSV: header\\nrând1\\nrând2\\n...
4. new Blob(\['\\uFEFF' + csvText], { type: 'text/csv;charset=utf-8;' })
   '\\uFEFF' = BOM UTF-8 → Excel deschide corect caracterele românești (ș, ț, ă etc.)
5. <a download="centralizator\_SSM\_YYYY-MM-DD.csv"> → click programatic
```

\---

## 14\. Referință elemente DOM

### Ecran Login

|ID|Tip|Rol|
|-|-|-|
|`loginScreen`|`<div>`|Container principal login|
|`passwordInput`|`<input type="password">`|Câmp parolă|
|`loginBtn`|`<button>`|Buton "Intră"|
|`loginError`|`<div>`|Mesaj eroare (display:none implicit)|

### Aplicație principală

|ID|Tip|Rol|
|-|-|-|
|`appScreen`|`<div>`|Container principal aplicație|
|`statusDot`|`<div>`|Punctul colorat indicator status|
|`statusLabel`|`<span>`|Textul indicator status|
|`tab-rapoarte`|`<div>`|Tab "Rapoarte PDF"|
|`tab-csv`|`<div>`|Tab "Centralizator CSV"|
|`content-rapoarte`|`<div>`|Conținut tab rapoarte|
|`content-csv`|`<div>`|Conținut tab CSV|

### Tab Rapoarte

|ID|Tip|Rol|
|-|-|-|
|`searchInput`|`<input>`|Căutare rapoarte|
|`sortSelect`|`<select>`|Selector sortare|
|`countBadge`|`<span>`|"X / Y rapoarte"|
|`fileList`|`<div>`|Lista fișierelor PDF|
|`previewFilename`|`<span>`|Numele fișierului selectat|
|`btnDownload`|`<button>`|Buton descărcare|
|`previewEmpty`|`<div>`|Stare: niciun fișier selectat|
|`previewLoading`|`<div>`|Overlay încărcare PDF|
|`previewProgressPct`|`<span>`|Procentaj progres "42%"|
|`progressFill`|`<div>`|Bara de progres (width animat)|
|`previewProgress`|`<span>`|Numele fișierului în încărcare|
|`previewError`|`<div>`|Overlay eroare|
|`previewErrorMsg`|`<span>`|Textul erorii|
|`pdfFrame`|`<iframe>`|Vizualizator PDF inline|
|`statsBar`|`<div>`|Bara statistici (Nr, Data, Localitate)|
|`statName`|`<span>`|Numărul raportului|
|`statDate`|`<span>`|Data formatată DD/MM/YYYY|
|`statLocation`|`<span>`|Localitate|

### Tab CSV

|ID|Tip|Rol|
|-|-|-|
|`csvSearch`|`<input>`|Căutare în CSV|
|`btnRefreshCsv`|`<button>`|Reîncarcă CSV de la GAS|
|`btnExportCsv`|`<button>`|Export fișier .csv (disabled până la încărcare)|
|`csvCountBadge`|`<span>`|"X / Y rânduri"|
|`csvTableWrap`|`<div>`|Container scrollabil tabel|
|`csvStatus`|`<div>`|"N înregistrări · M coloane"|

\---

## 15\. Referință completă funcții

|Funcție|Parametri|Return|Descriere|
|-|-|-|-|
|`init()`|—|`void`|IIFE: verifică token și routează la login sau app|
|`showLogin()`|—|`void`|Afișează ecranul de login, focus pe câmpul parolă|
|`showApp()`|—|`void`|Afișează aplicația principală|
|`doLogin()`|—|`Promise<void>`|Trimite parola la GAS, salvează token, inițializează app|
|`doLogout()`|—|`void`|Curăță sesiunea și redirecționează la login|
|`switchTab(tab)`|`string`|`void`|Schimbă tab-ul activ; încarcă CSV la prima accesare|
|`loadRapoarte()`|—|`Promise<void>`|Obține lista de rapoarte de la GAS|
|`setStatus(type, label)`|`string, string`|`void`|Actualizează indicatorul de status din topbar|
|`isNew(d)`|`string`|`boolean`|True dacă data e în ultimele 14 zile|
|`formatDate(d)`|`string`|`string`|Convertește 'YYYY-MM-DD' → 'DD/MM/YYYY'|
|`filterFiles()`|—|`void`|Filtrează și sortează lista de rapoarte|
|`renderList(files)`|`Array`|`void`|Generează HTML lista sidebar cu cache indicator|
|`selectFile(f)`|`Object`|`void`|Selectează un raport și declanșează previzualizarea|
|`showPreviewState(state)`|`string\|null`|`void`|Comută starea panoului preview|
|`getCacheKey(filename)`|`string`|`string`|Returnează cheia localStorage pentru un fișier|
|`isCached(filename)`|`string`|`boolean`|Verifică dacă fișierul e în cache|
|`saveToCache(filename, base64)`|`string, string`|`void`|Salvează în cache cu management spațiu FIFO|
|`loadFromCache(filename)`|`string`|`string\|null`|Citește din cache|
|`base64ToBlob(base64)`|`string`|`Blob`|Convertește base64 în Blob PDF|
|`prefetchOnHover(filename)`|`string`|`void`|Prefetch cu delay 300ms la hover|
|`fetchPDF(filename, onProgress)`|`string, Function\|null`|`Promise<string\|null>`|Descarcă PDF de la GAS via ReadableStream|
|`previewCurrent()`|—|`Promise<void>`|Afișează PDF-ul curent (cache → GAS) cu progress|
|`showPdfBlob(blob)`|`Blob`|`void`|Afișează un Blob PDF în iframe|
|`downloadCurrent()`|—|`void`|Descarcă PDF-ul curent ca fișier|
|`loadCsv()`|—|`Promise<void>`|Încarcă și parsează CSV-ul de la GAS|
|`parseCSV(text)`|`string`|`Array<Array<string>>`|Parser CSV cu suport ghilimele și virgule|
|`filterCsv()`|—|`void`|Filtrează rândurile CSV după query|
|`renderCsv(rows)`|`Array`|`void`|Generează HTML tabel cu badge-uri DA/NU/N/A|
|`sortCsv(colIdx)`|`number`|`void`|Sortează tabelul după coloana specificată|
|`exportCsv()`|—|`void`|Descarcă toate datele CSV cu BOM UTF-8|

\---

## 

