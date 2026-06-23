Documentație tehnică — Platformă Vizualizare Rapoarte SSM
Tip fișier: HTML monolitic (HTML + CSS + JavaScript într-un singur fișier)
Backend: Google Apps Script (GAS), folosit ca proxy de autentificare și acces la GitHub
Repository sursă date: `RapoarteSSM` pe GitHub, foldere `rapoarte\_raw` (RO) și `rapoarte\_raw\_EN` (EN)
---
1. Arhitectura generală
```
Login (email + parolă)
        │
        ▼
GAS verifică email + parolă → emite token de sesiune
        │
        ▼
Pagina cere lista de rapoarte + CSV-ul centralizator (în paralel)
        │
        ▼
Utilizatorul selectează un raport → GAS citește fișierul HTML din GitHub → afișat în <iframe>
        │
        ▼
Tab separat "Centralizator" → CSV brut parsat și afișat ca tabel interactiv
```
Nu există bază de date proprie — toate rapoartele sunt fișiere HTML statice stocate în GitHub, iar centralizatorul e un singur fișier `.csv`. Pagina doar le citește prin GAS și le randează.
---
2. Autentificare
2.1 Flux de login (`doLogin()`)
Necesită email și parolă. Trimite ambele către GAS:
```
GAS\_URL?action=login\&password=...\&email=...
```
Verificarea parolei e făcută integral pe partea de GAS (hash, nu se vede în acest fișier).
Verificarea email-ului e făcută server-side, în GAS — pagina trimite email-ul ca parametru simplu, fără validare locală suplimentară înainte de trimitere (în afară de `if (!email || !pw) return;`).
La succes, GAS returnează un `token`, salvat în `sessionStorage.getItem('ssm\_token')`.
2.2 Persistarea sesiunii
Token-ul e citit din `sessionStorage` la încărcarea paginii (`init()`); dacă există, utilizatorul e logat automat fără să reintroducă parola — sesiunea ține cât ține tab-ul/fereastra browserului (nu supraviețuiește restart de browser, pentru că `sessionStorage` se șterge la închiderea tab-ului).
2.3 Logout (`doLogout()`)
Șterge token-ul din `sessionStorage` și resetează toată starea în memorie (`allFiles`, `currentFile`, `currentBlob`, etc.) înainte de a reveni la ecranul de login.
---
3. Structura paginii — cele 2 tab-uri
3.1 Tab „Rapoarte" (`#content-rapoarte`)
Layout pe 2 coloane:
Sidebar stânga (`.sidebar`) — listă de rapoarte, cu căutare și sortare
Panou dreapta (`.preview`) — previzualizare HTML a raportului selectat, într-un `<iframe>`
3.2 Tab „Centralizator" (`#content-csv`)
Un singur panou (`.csv-panel`) cu:
bară de căutare globală (toate coloanele)
toggle „Arată comentariile" (vezi secțiunea 8)
tabel HTML generat din parsarea CSV-ului, cu sortare și redimensionare de coloane
Comutarea între tab-uri se face prin `switchTab(tab)`, care ascunde/arată `.tab-content` corespunzător și încarcă datele CSV la prima accesare a tab-ului (`if (tab === 'csv' \&\& !csvLoaded) loadCsv();`).
---
4. Încărcarea listei de rapoarte — `loadRapoarte()`
La autentificare (sau la reluarea sesiunii), se fac 2 cereri în paralel:
```javascript
Promise.all(\[
  fetch(...\&action=rapoarte...),
  fetch(...\&action=csv...)
])
```
Lista de fișiere (`action=rapoarte`) — returnează metadate per raport (`name`, `nr`, `date`, `location`), extrase de GAS din numele fișierelor din GitHub.
CSV-ul centralizator (`action=csv`) — folosit aici doar pentru a extrage coloana `Locatie` și a o asocia fiecărui raport prin numărul de raport (`locatieMap`), pentru a afișa locația exactă (diferită de localitate) în lista de rapoarte.
Golirea automată a cache-ului local — la fiecare apel de `loadRapoarte()` (deci la fiecare refresh de pagină), toate cheile din `localStorage` cu prefixul `ssm\_pdf\_` sunt șterse. Scopul: garantarea că la fiecare reîncărcare a paginii, rapoartele afișate sunt mereu varianta curentă din GitHub, nu o versiune veche cache-uită local.
---
5. Lista de rapoarte — căutare, sortare, randare
5.1 `filterFiles()`
Filtrează `allFiles` (toată lista încărcată) pe baza textului din `#searchInput`, căutând în `name`, `location`, `nr`, `date` simultan. Sortează crescător sau descrescător după numărul de raport (extras din `nr` prin `parseInt(...replace(/\[^0-9]/g,''))`), nu alfabetic.
5.2 `renderList(files, isTest)`
Generează HTML-ul listei. Fiecare element (`.file-item`) include:
iconiță document
nume fișier + indicator verde (●) dacă fișierul e deja în cache local
metadate: dată formatată (`formatDate`), localitate, locație
Parametrul `isTest` e un rest arhitectural pentru un sistem de „rapoarte de test" care nu mai pare a fi folosit activ în acest fișier (funcțiile `loadTeste`, `selectTestFile` etc. există dar nu sunt conectate la niciun tab vizibil în HTML-ul curent) — de verificat dacă acest cod mort poate fi eliminat într-o curățare viitoare.
---
6. Previzualizarea unui raport
6.1 Selectarea unui raport — `selectFile(f)`
Detectează dispozitivul prin `navigator.userAgent`:
Desktop → `previewCurrent()` încarcă direct HTML-ul în iframe
Mobil (iPhone/iPad/Android) → afișează imediat modalul cu 2 opțiuni (`#mobModal`), în timp ce HTML-ul se descarcă în fundal (`window.\_mobHtml`)
6.2 `previewCurrent()` — încărcare cu cache
Dacă raportul e deja în `currentBlob` (memorie) → randează direct.
Dacă e în `localStorage` (cache) → randează din cache.
Altfel → `fetchHTMLFor()` descarcă din GitHub prin GAS, salvează în cache (`saveToCache`), apoi randează.
O bară de progres simulată (`simPct`) crește progresiv către 85% în timp ce se așteaptă răspunsul rețelei, apoi sare la 100% când conținutul ajunge — pur cosmetic, nu reflectă progresul real al descărcării.
6.3 `showHtmlPreview()` — randarea efectivă
Injectează `<title>` cu numele fișierului (fără extensie) în HTML-ul primit.
Injectează forțat un stil `@page { size: A4 portrait; }` pentru consistență la print, indiferent de stilurile proprii ale raportului.
Setează rezultatul ca `iframe.srcdoc`.
Aplică nivelul de zoom curent (`currentZoom`) prin `iframe.contentDocument.body.style.zoom`.
---
7. Cache local (`localStorage`)
7.1 Chei și prefixe
`LS\_PREFIX = 'ssm\_pdf\_'` — rapoarte normale
`LS\_TEST\_PREFIX = 'ssm\_test\_'` — rapoarte test (cod rezidual, vezi secțiunea 5.2)
`LS\_MAX\_MB = 4` — limita totală de spațiu alocat cache-ului
7.2 `getCacheKey(filename, isTest)`
Cheia include un prefix de limbă: dacă `currentLang === 'en'`, cheia devine `ssm\_pdf\_en\_{filename}`. Astfel versiunile RO și EN ale aceluiași raport sunt cache-uite separat, fără să se suprascrie una pe alta.
7.3 `saveToCache()` — eviction LRU simplificat
Înainte de a salva un fișier nou, calculează spațiul total ocupat de cache (aproximat din lungimea string-ului × 0.75, simulând raportul base64→bytes). Dacă adăugarea noului fișier ar depăși `LS\_MAX\_MB`, șterge cele mai vechi chei (`keys.shift()`, ordine de inserare, nu de utilizare) până se face loc.
---
8. Comutarea limbă RO/EN
8.1 Mecanism
`currentLang` controlează:
traducerea textelor din interfață (`I18N`, `applyLang()`, `t(key)`)
folderul din care se descarcă rapoartele (`fetchHTMLFor`: `\&folder=rapoarte\_raw\_EN` dacă `currentLang === 'en'`)
prefixul cheii de cache (secțiunea 7.2)
Preferința de limbă e salvată persistent în `localStorage.getItem('ssm\_lang')` și restaurată la încărcarea paginii (independent de sesiunea de login).
8.2 `setLang(lang)`
La comutare:
Actualizează `currentLang`, salvează preferința.
Re-traduce toată interfața (`applyLang()`).
Resetează `currentBlob` la `null` și șterge din cache varianta RO a raportului curent — pentru a forța o redescărcare din folderul corect de limbă (`previewCurrent()` e reapelat automat).
Important: sistemul de traducere RO/EN se aplică doar interfeței și raportului afișat momentan. CSV-ul centralizator (tab-ul „Centralizator") nu are o variantă EN — `loadCsv()` citește mereu același `centralizator.csv`, independent de `currentLang`.
---
9. Descărcare / Printare raport
9.1 Desktop — `downloadCurrent()`
Nu deschide un tab nou și nu generează un fișier separat — printează direct conținutul iframe-ului deja încărcat:
```javascript
frame.contentDocument.title = fileName;  // numele apare corect în dialogul de print
frame.contentWindow.print();
```
Dacă iframe-ul nu are încă raportul încărcat, apelează mai întâi `previewCurrent()` și abia după print, cu un delay de 600ms pentru randare.
9.2 Mobil — `\_mobOpenUrl(html, print)`
Spre diferență de desktop, pe mobil se deschide un tab nou (`window.open('', '\_blank')`), în care se scrie HTML-ul cu `document.write()`. Motivul pentru abordarea diferită: pe mobil nu există un iframe vizibil de printat direct, deci raportul trebuie expus ca document de prim-nivel al unui tab nou pentru ca dialogul de print al sistemului să-l recunoască corect.
Același stil A4 forțat (`@page`) e injectat și aici, separat de injectarea din `showHtmlPreview()`.
---
10. Modal mobil cu 2 opțiuni (`#mobModal`)
Pe dispozitive mobile, selectarea unui raport nu randează direct — afișează un modal cu:
„Vizualizare" (`mobActionView()`) → deschide raportul într-un tab nou, fără print
„Descarcă / Printează" (`mobActionDownload()`) → deschide raportul și declanșează dialogul de print
Ambele funcții gestionează cazul în care HTML-ul nu s-a descărcat încă (polling la 400ms cu animație de puncte „Se încarcă...", `window.\_mobHtml` setat asincron din `selectFile`/`selectTestFile`).
---
11. Tab Centralizator — parsare și randare CSV
11.1 `parseCSV(text)`
Parser CSV manual, caracter cu caracter, care gestionează corect câmpurile încadrate în ghilimele (inclusiv ghilimele escapate `""`) și virgulele din interiorul câmpurilor citate. Nu folosește nicio librărie externă.
11.2 `renderCsvFor(rows, headers, isTest)`
Construiește un `<table>` HTML complet de la zero la fiecare randare (nu actualizează incremental DOM-ul existent). Pentru fiecare celulă, valorile `DA`/`NU`/`N/A` sunt înlocuite cu badge-uri colorate (`.badge-da`, `.badge-nu`, `.badge-na`).
11.3 Filtrarea coloanelor de comentarii — `showComments`, `isCommentColumn()`
```javascript
function isCommentColumn(headerName) {
  return (headerName || '').toUpperCase().includes('COMENTARIU');
}
```
Implicit, coloanele care conțin „COMENTARIU" în nume sunt ascunse din tabel (pentru lizibilitate — centralizatorul are sute de coloane Răspuns/Comentariu alternante). Checkbox-ul „Arată comentariile" (`#toggleComments`) le readuce. Lista de indici vizibili e recalculată la fiecare randare:
```javascript
const visibleIdx = headers.map((h, i) => i).filter(i => showComments || !isCommentColumn(headers\[i]));
```
> \*\*Notă pentru mentenanță:\*\* funcția `onToggleComments()` e definită de două ori identic în fișierul curent (duplicat literal) — a doua declarație suprascrie prima fără efect practic, dar e cod redundant de eliminat la o curățare.
11.4 Sortare — `sortCsvFor(colIdx, isTest)`
Click pe header sortează crescător; un al doilea click pe aceeași coloană inversează direcția (`csvSortDir \*= -1`). Sortarea e alfanumerică (`localeCompare` cu opțiunea `numeric:true`), deci numerele din text sunt comparate corect (ex. „10" după „9", nu lexicografic).
11.5 Export CSV — `exportCsv()`
Reconstruiește CSV-ul din `csvHeaders`/`csvData` păstrate în memorie (nu din ce e randat pe ecran — deci exportul include toate coloanele, indiferent de starea toggle-ului „Arată comentariile"), cu BOM UTF-8 (`\\uFEFF`) pentru compatibilitate Excel, și declanșează descărcarea ca fișier `.csv` cu data curentă în nume.
---
12. Redimensionarea coloanelor (drag + auto-fit)
12.1 Handle-uri de redimensionare
Fiecare `<th>` are 2 zone de drag (`.col-resize-handle`), poziționate la marginea stânga și dreapta a coloanei, care se extind vizual și dincolo de marginea coloanei (`right:-10px; width:20px`) pentru o zonă de prindere mai ușoară cu mouse-ul. Handle-ul din stânga al coloanei N redimensionează de fapt coloana N-1 (`handle.parentElement.previousElementSibling`), astfel încât drag-ul pe orice graniță vizuală dintre 2 coloane mereu redimensionează coloana din stânga graniței.
12.2 `initColResize(e, handle, dir)`
Pattern clasic de drag: la `mousedown` se înregistrează listeneri globali `mousemove`/`mouseup` pe `document`, care actualizează `th.style.width/minWidth/maxWidth` în timp real și se dezînregistrează la `mouseup`. Lățimea minimă e limitată la 40px.
12.3 `autoFitColumn(handle, dir)` — dublu-click
La dublu-click pe un handle, lățimea coloanei e recalculată automat pentru a încăpea conținutul cel mai larg, folosind un `<canvas>` offscreen și `ctx.measureText()` pentru a măsura precis lățimea textului (header + toate valorile din coloana respectivă, inclusiv `title`-ul complet al celulei dacă textul e altfel trunchiat vizual), plus 28px padding de siguranță.
---
13. Responsive / comportament mobil
13.1 Breakpoint
`@media(max-width:680px)` schimbă layout-ul sidebar+preview din orizontal (side-by-side) în vertical, cu doar unul vizibil la un moment dat (`mobile-hide`/`mobile-show`).
13.2 `mobileShowPreview()` / `mobileBackToList()`
Comută clasele `.mobile-hide`/`.mobile-show` pe `.sidebar`/`.preview` din interiorul containerului tab-ului curent (`#content-rapoarte` sau `#content-teste`). Aceste funcții sunt apelate automat din `selectFile`/`selectTestFile`, dar — de notat — nu există în acest fișier niciun buton vizibil „Înapoi la listă" pe mobil; revenirea la listă pe ecrane mici se face implicit doar prin selectarea unui alt raport sau prin navigarea browserului (back), nu printr-un control dedicat în interfață.
---
14. Detecție mobil prin User-Agent
Folosit consecvent în 3 locuri (`selectFile`, `selectTestFile`, `showHtmlPreview`):
```javascript
/iPhone|iPad|iPod|Android/i.test(navigator.userAgent)
```
Nu există o singură funcție/constantă centralizată pentru această verificare — e duplicată identic de 3 ori. O eventuală refactorizare ar putea extrage-o într-o funcție `isMobileDevice()`.
---
15. Internaționalizare (I18N)
Structură simplă, plată: `I18N.ro` și `I18N.en` sunt obiecte cu aceleași chei, valorile fiind textele traduse. `t(key)` face fallback la română dacă cheia nu există în limba curentă sau dacă limba e necunoscută.
`applyLang()` aplică traducerile pe baza atributelor `data-i18n` (text), `data-i18n-placeholder` (placeholder input), `data-i18n-title` (atribut `title`), parcurgând tot DOM-ul cu `querySelectorAll` la fiecare comutare de limbă — nu există binding reactiv, traducerea e o trecere completă peste DOM de fiecare dată.
---
16. Note pentru mentenanță viitoare
Cod rezidual „teste": funcțiile `loadTeste`, `filterTeste`, `selectTestFile`, `previewCurrentTest`, `zoomPreviewTest` și variabilele `allTestFiles`/`currentTestFile`/`testeLoaded` există complet funcțional, dar nu sunt conectate la niciun tab vizibil în HTML-ul curent (nu există `#tab-teste` sau `#content-teste` în markup). Sigur de eliminat dacă funcționalitatea „rapoarte de test" a fost definitiv abandonată, sau de reconectat dacă a fost doar temporar dezactivată.
`onToggleComments()` duplicată — a doua definiție e identică cu prima, fără efect funcțional, dar adaugă confuzie la citire.
Detecția de mobil repetată de 3 ori — candidat pentru extragere într-o singură funcție.
CSV-ul nu are variantă EN — la comutarea pe engleză, doar rapoartele individuale se traduc (din folderul `rapoarte\_raw\_EN`); tabul Centralizator rămâne mereu în română, indiferent de `currentLang`. De clarificat dacă acesta e comportamentul intenționat pe termen lung.
Export CSV exportă mereu toate coloanele, inclusiv comentariile, independent de starea toggle-ului de afișare — comportament intenționat (export complet), dar trebuie comunicat clar utilizatorilor care se bazează vizual pe toggle.
Stilul A4 forțat la print (`@page{size:A4}`) e injectat în 2 locuri diferite (`showHtmlPreview` pentru desktop, `\_mobOpenUrl` pentru mobil) cu cod aproape identic — candidat pentru extragere într-o singură constantă/funcție reutilizabilă.
