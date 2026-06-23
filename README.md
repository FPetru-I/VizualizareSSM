Platforma Vizualizare Rapoarte SSM
Tip fisier: HTML monolitic (HTML + CSS + JavaScript intr-un singur fisier)
Backend: Google Apps Script (GAS), folosit ca proxy de autentificare si acces la GitHub
Repository sursa date: RapoarteSSM pe GitHub, foldere `rapoarte_raw` (RO) si `rapoarte_raw_EN` (EN)
Aceasta versiune a documentatiei reflecta fisierul curent, dupa eliminarea completa a codului rezidual legat de "rapoarte de test" (tab, functii, variabile de stare, chei de cache si elemente CSS care nu mai aveau nicio legatura cu interfata vizibila).
---
1. Arhitectura generala
```
Login (email + parola)
        |
        v
GAS verifica email + parola, emite token de sesiune
        |
        v
Pagina cere lista de rapoarte si CSV-ul centralizator (in paralel)
        |
        v
Utilizatorul selecteaza un raport, GAS citeste fisierul HTML din GitHub, afisat in iframe
        |
        v
Tab separat "Centralizator": CSV brut parsat si afisat ca tabel interactiv
```
Nu exista baza de date proprie. Toate rapoartele sunt fisiere HTML statice stocate in GitHub, iar centralizatorul e un singur fisier `.csv`. Pagina doar le citeste prin GAS si le randeaza.
---
2. Autentificare
2.1 Flux de login — functia `doLogin()`
Necesita email si parola. Trimite ambele catre GAS:
```
GAS_URL?action=login&password=...&email=...
```
Verificarea parolei e facuta integral pe partea de GAS (hash, nu se vede in acest fisier). Verificarea email-ului e facuta server-side, in GAS. Pagina trimite email-ul ca parametru simplu, fara validare locala suplimentara inainte de trimitere, in afara de verificarea ca ambele campuri sunt completate.
La succes, GAS returneaza un token, salvat in `sessionStorage` sub cheia `ssm_token`.
2.2 Persistarea sesiunii
Token-ul e citit din `sessionStorage` la incarcarea paginii, in functia `init()`. Daca exista, utilizatorul e logat automat fara sa reintroduca parola. Sesiunea tine cat tine tab-ul sau fereastra browserului, pentru ca `sessionStorage` se sterge la inchiderea tab-ului.
2.3 Logout — functia `doLogout()`
Sterge token-ul din `sessionStorage` si reseteaza starea in memorie (`allFiles`, `currentFile`, `currentBlob`) inainte de a reveni la ecranul de login.
---
3. Structura paginii — cele doua tab-uri
3.1 Tab Rapoarte — elementul `content-rapoarte`
Layout pe doua coloane:
Sidebar stanga, clasa `sidebar`: lista de rapoarte, cu cautare si sortare.
Panou dreapta, clasa `preview`: previzualizare HTML a raportului selectat, intr-un `iframe`.
3.2 Tab Centralizator — elementul `content-csv`
Un singur panou, clasa `csv-panel`, cu:
bara de cautare globala pe toate coloanele,
toggle "Arata comentariile" (vezi sectiunea 11.3),
tabel HTML generat din parsarea CSV-ului, cu sortare si redimensionare de coloane.
Comutarea intre tab-uri se face prin functia `switchTab(tab)`, care ascunde sau arata continutul corespunzator si incarca datele CSV la prima accesare a tabului respectiv.
---
4. Incarcarea listei de rapoarte — functia `loadRapoarte()`
La autentificare, sau la reluarea sesiunii, se fac doua cereri in paralel:
fetch catre actiunea `rapoarte`
fetch catre actiunea `csv`
Lista de fisiere (actiunea `rapoarte`) returneaza metadate per raport: nume, numar, data, localitate, extrase de GAS din numele fisierelor din GitHub.
CSV-ul centralizator (actiunea `csv`) e folosit aici doar pentru a extrage coloana `Locatie` si a o asocia fiecarui raport prin numarul de raport, pentru a afisa locatia exacta in lista de rapoarte.
Golirea automata a cache-ului local: la fiecare apel de `loadRapoarte()`, deci la fiecare refresh de pagina, toate cheile din `localStorage` cu prefixul `ssm_pdf_` sunt sterse. Scopul este garantarea ca, la fiecare reincarcare a paginii, rapoartele afisate sunt mereu varianta curenta din GitHub, nu o versiune veche cache-uita local.
---
5. Lista de rapoarte — cautare, sortare, randare
5.1 Functia `filterFiles()`
Filtreaza `allFiles` pe baza textului din campul de cautare, cautand simultan in nume, localitate, numar de raport si data. Sorteaza crescator sau descrescator dupa numarul de raport, extras prin `parseInt`, nu alfabetic.
5.2 Functia `renderList(files)`
Genereaza HTML-ul listei. Fiecare element include:
iconita document,
nume fisier, plus un indicator verde daca fisierul e deja in cache local,
metadate: data formatata, localitate, locatie.
---
6. Previzualizarea unui raport
6.1 Selectarea unui raport — functia `selectFile(f)`
Detecteaza dispozitivul prin functia centralizata `isMobileDevice()` (vezi sectiunea 14):
Pe desktop, `previewCurrent()` incarca direct HTML-ul in iframe.
Pe mobil, se afiseaza imediat modalul cu doua optiuni, in timp ce HTML-ul se descarca in fundal, stocat temporar in `window._mobHtml`.
6.2 Functia `previewCurrent()` — incarcare cu cache
Daca raportul e deja in `currentBlob`, in memorie, se randeaza direct.
Daca e in `localStorage`, se randeaza din cache.
Altfel, `fetchHTMLFor()` descarca din GitHub prin GAS, salveaza in cache prin `saveToCache`, apoi randeaza.
O bara de progres simulata creste progresiv catre 85% in timp ce se asteapta raspunsul retelei, apoi sare la 100% cand continutul ajunge. Aceasta bara e pur cosmetica, nu reflecta progresul real al descarcarii.
6.3 Functia `showHtmlPreview(html)` — randarea efectiva
Injecteaza titlul, fara extensie, in HTML-ul primit.
Injecteaza fortat un stil de pagina A4, prin constanta `A4_PRINT_STYLE`, pentru consistenta la print, indiferent de stilurile proprii ale raportului.
Seteaza rezultatul ca `iframe.srcdoc`.
Aplica nivelul de zoom curent prin `iframe.contentDocument.body.style.zoom`.
---
7. Cache local — `localStorage`
7.1 Chei si prefixe
`LS_PREFIX`, cu valoarea `ssm_pdf_`, e singurul prefix de cache folosit in versiunea curenta.
`LS_MAX_MB`, cu valoarea 4, e limita totala de spatiu alocat cache-ului.
7.2 Functia `getCacheKey(filename)`
Cheia include un prefix de limba: daca `currentLang` este `en`, cheia devine `ssm_pdf_en_` urmat de numele fisierului. Astfel versiunile RO si EN ale aceluiasi raport sunt cache-uite separat, fara sa se suprascrie una pe alta.
7.3 Functia `saveToCache(filename, content)` — eviction simplificat
Inainte de a salva un fisier nou, se calculeaza spatiul total ocupat de cache, aproximat din lungimea string-ului inmultita cu 0.75, simuland raportul intre lungimea base64 si numarul real de bytes. Daca adaugarea noului fisier ar depasi `LS_MAX_MB`, se sterg cele mai vechi chei, in ordinea de inserare, nu de utilizare, pana se face loc.
---
8. Comutarea limba RO/EN
8.1 Mecanism
Variabila `currentLang` controleaza:
traducerea textelor din interfata, prin obiectul `I18N` si functiile `applyLang` si `t`,
folderul din care se descarca rapoartele, in `fetchHTMLFor`, care adauga parametrul de folder `rapoarte_raw_EN` daca `currentLang` este `en`,
prefixul cheii de cache, descris in sectiunea 7.2.
Preferinta de limba e salvata persistent in `localStorage`, sub cheia `ssm_lang`, si restaurata la incarcarea paginii, independent de sesiunea de login.
8.2 Functia `setLang(lang)`
La comutare:
se actualizeaza `currentLang` si se salveaza preferinta,
se re-traduce toata interfata prin `applyLang`,
se reseteaza `currentBlob` la `null` si se sterge din cache varianta curenta a raportului afisat, pentru a forta o redescarcare din folderul corect de limba. `previewCurrent()` e reapelat automat dupa aceasta resetare.
De retinut: sistemul de traducere RO/EN se aplica doar interfetei si raportului afisat momentan. CSV-ul centralizator, din tabul Centralizator, nu are o varianta EN. `loadCsv()` citeste mereu acelasi `centralizator.csv`, independent de `currentLang`.
---
9. Descarcare si printare raport
9.1 Desktop — functia `downloadCurrent()`
Nu deschide un tab nou si nu genereaza un fisier separat. Printeaza direct continutul iframe-ului deja incarcat, setand mai intai titlul documentului din iframe la numele fisierului, astfel incat acesta sa apara corect in dialogul de salvare ca PDF al browserului. Daca iframe-ul nu are inca raportul incarcat, se apeleaza mai intai `previewCurrent()` si abia apoi se printeaza, cu o intarziere de 600 ms pentru randare.
9.2 Mobil — functia `_mobOpenUrl(html, print)`
Spre diferenta de desktop, pe mobil se deschide un tab nou, in care se scrie HTML-ul direct cu `document.write`. Motivul pentru abordarea diferita: pe mobil nu exista un iframe vizibil de printat direct, deci raportul trebuie expus ca document de prim nivel al unui tab nou pentru ca dialogul de print al sistemului sa il recunoasca corect.
Acelasi stil A4 fortat, prin constanta `A4_PRINT_STYLE`, e injectat si aici, reutilizand exact aceeasi definitie ca cea folosita la previzualizarea pe desktop.
---
10. Modalul mobil cu doua optiuni
Pe dispozitive mobile, selectarea unui raport nu randeaza direct, ci afiseaza un modal cu doua butoane:
Vizualizare — functia `mobActionView()`, care deschide raportul intr-un tab nou, fara print.
Descarca / Printeaza — functia `mobActionDownload()`, care deschide raportul si declanseaza dialogul de print.
Ambele functii gestioneaza cazul in care HTML-ul nu s-a descarcat inca, printr-un polling la 400 ms, cu o animatie simpla de puncte care indica incarcarea. Continutul descarcat e stocat temporar in `window._mobHtml`, setat asincron din `selectFile`.
---
11. Tab Centralizator — parsare si randare CSV
11.1 Functia `parseCSV(text)`
Parser CSV manual, caracter cu caracter, care gestioneaza corect campurile incadrate in ghilimele, inclusiv ghilimelele escapate, si virgulele din interiorul campurilor citate. Nu foloseste nicio librarie externa.
11.2 Functia `renderCsvFor(rows, headers)`
Construieste un tabel HTML complet de la zero la fiecare randare, nu actualizeaza incremental DOM-ul existent. Pentru fiecare celula, valorile DA, NU si N/A sunt inlocuite cu insigne colorate.
11.3 Filtrarea coloanelor de comentarii — variabila `showComments` si functia `isCommentColumn`
Implicit, coloanele care contin cuvantul COMENTARIU in nume sunt ascunse din tabel, pentru lizibilitate, intrucat centralizatorul are sute de coloane de tip Raspuns si Comentariu alternante. Checkbox-ul "Arata comentariile" le readuce. Lista de indici vizibili e recalculata la fiecare randare, pe baza starii curente a variabilei `showComments`.
11.4 Sortare — functia `sortCsvFor(colIdx)`
Click pe header sorteaza crescator. Un al doilea click pe aceeasi coloana inverseaza directia. Sortarea e alfanumerica, folosind `localeCompare` cu optiunea `numeric` activata, deci numerele din text sunt comparate corect (de exemplu 10 dupa 9, nu in ordine lexicografica).
11.5 Export CSV — functia `exportCsv()`
Reconstruieste CSV-ul din `csvHeaders` si `csvData`, pastrate integral in memorie, nu din ce e randat pe ecran. Exportul include deci toate coloanele, indiferent de starea toggle-ului "Arata comentariile". Fisierul exportat include un marcaj BOM UTF-8 pentru compatibilitate cu Excel, si declanseaza descarcarea ca fisier CSV, cu data curenta in nume.
---
12. Redimensionarea coloanelor — drag si auto-fit
12.1 Manere de redimensionare
Fiecare antet de coloana are doua zone de tip drag, pozitionate la marginea stanga si dreapta a coloanei, care se extind vizual si dincolo de marginea coloanei pentru o zona de prindere mai usoara cu mouse-ul. Manerul din stanga al unei coloane redimensioneaza de fapt coloana precedenta, astfel incat tragerea pe orice granita vizuala dintre doua coloane mereu redimensioneaza coloana din stanga granitei.
12.2 Functia `initColResize(e, handle, dir)`
Pattern clasic de drag: la apasarea butonului mouse-ului se inregistreaza listeneri globali de miscare si de eliberare pe `document`, care actualizeaza latimea coloanei in timp real si se dezinregistreaza la eliberarea butonului. Latimea minima e limitata la 40px.
12.3 Functia `autoFitColumn(handle, dir)` — dublu-click
La dublu-click pe un maner, latimea coloanei e recalculata automat pentru a incapea continutul cel mai larg, folosind un canvas offscreen si functia `measureText` pentru a masura precis latimea textului, atat headerul cat si toate valorile din coloana respectiva, plus un spatiu de siguranta de 28px.
---
13. Responsive — comportament pe mobil
13.1 Breakpoint
Media query-ul pentru latimi sub 680px schimba layout-ul sidebar si preview din orizontal (side-by-side) in vertical, cu doar unul vizibil la un moment dat.
13.2 Functiile `mobileShowPreview()` si `mobileBackToList()`
Comuta clasele de afisare pe sidebar si preview din interiorul containerului tabului Rapoarte. Aceste functii sunt apelate automat din `selectFile`, dar nu exista in acest fisier niciun buton vizibil "Inapoi la lista" pe mobil. Revenirea la lista pe ecrane mici se face implicit doar prin selectarea unui alt raport sau prin navigarea browserului inapoi, nu printr-un control dedicat in interfata.
---
14. Detectia dispozitivelor mobile — functia `isMobileDevice()`
Centralizata intr-o singura functie, reutilizata in toate locurile care au nevoie de aceasta verificare:
```javascript
function isMobileDevice() {
  return /iPhone|iPad|iPod|Android/i.test(navigator.userAgent);
}
```
Inainte de curatarea codului, aceasta verificare era duplicata identic in trei locuri diferite. Centralizarea elimina riscul ca o eventuala modificare viitoare a logicii de detectie sa fie aplicata doar partial.
---
15. Internationalizare — I18N
Structura simpla, plata: `I18N.ro` si `I18N.en` sunt obiecte cu aceleasi chei, valorile fiind textele traduse. Functia `t(key)` face fallback la romana daca cheia nu exista in limba curenta sau daca limba e necunoscuta.
Functia `applyLang()` aplica traducerile pe baza atributelor `data-i18n` (pentru text), `data-i18n-placeholder` (pentru placeholder-ul campurilor de input) si `data-i18n-title` (pentru atributul `title`), parcurgand tot DOM-ul la fiecare comutare de limba. Nu exista binding reactiv — traducerea e o trecere completa peste DOM de fiecare data cand se schimba limba.
---
16. Stilul A4 fortat la print — constanta `A4_PRINT_STYLE`
Definita o singura data, la nivel global:
```javascript
const A4_PRINT_STYLE = /* stilul CSS care forteaza marimea paginii A4 portrait si latimea de 210mm la print */;
```
Aceasta constanta e reutilizata identic in doua locuri: la previzualizarea pe desktop, in `showHtmlPreview`, si la deschiderea raportului pe mobil, in `_mobOpenUrl`. Inainte de curatarea codului, aceasta definitie era duplicata, cu o mica diferenta: versiunea folosita pe mobil avea o paranteza inchisa lipsa in regula media print, ceea ce putea produce CSS invalid. Centralizarea intr-o singura constanta elimina si aceasta inconsistenta.
---
17. Functionalitate eliminata fata de versiunile anterioare
Aceasta sectiune documenteaza ce a fost scos din pagina, pentru context istoric si pentru a evita reintroducerea accidentala a unor referinte la cod care nu mai exista.
A fost eliminat complet sistemul de "rapoarte de test", care includea:
un tab separat in interfata, cu propriul continut si stilizare distincta,
o lista proprie de fisiere de test, incarcata separat de lista principala,
un CSV centralizator de test, separat de cel principal,
chei de cache cu prefix propriu,
toate functiile asociate: incarcarea listei de test, filtrarea, selectarea unui fisier de test, zoom separat, previzualizare separata, incarcarea si filtrarea CSV-ului de test,
toate elementele CSS asociate, inclusiv variabilele de culoare dedicate si stilurile pentru elementele de lista marcate ca fiind de test,
un modal mobil care, in versiunile anterioare, retinea separat daca fisierul deschis era unul de test.
Toate functiile ramase in pagina care anterior acceptau un parametru boolean pentru a alege intre fluxul normal si fluxul de test au fost simplificate, eliminand acel parametru. Aceasta afecteaza, printre altele, functiile de gestionare a cache-ului, functia de descarcare a continutului HTML al unui raport, functiile de randare si sortare a tabelului CSV, si functiile de afisare sau ascundere a panourilor pe ecrane mici.
A fost eliminata si o variabila de stare nefolosita, gandita initial pentru o functionalitate de preincarcare a rapoartelor la trecerea cursorului peste un element din lista, functionalitate care nu a fost niciodata implementata efectiv in pagina.
---
18. Note pentru mentenanta viitoare
CSV-ul nu are varianta EN: la comutarea pe engleza, doar rapoartele individuale se traduc, din folderul `rapoarte_raw_EN`. Tabul Centralizator ramane mereu in romana, indiferent de limba selectata. Daca acest comportament nu e cel dorit pe termen lung, ar fi nevoie de un fisier CSV separat in engleza, generat la trimiterea raportului din formular, plus o modificare pe partea de GAS pentru a accepta un parametru de limba la citirea centralizatorului.
Export CSV exporta mereu toate coloanele, inclusiv comentariile, independent de starea toggle-ului de afisare. Acesta e un comportament intentionat, de export complet, dar trebuie comunicat clar utilizatorilor care se bazeaza vizual pe toggle si ar putea presupune ca exportul reflecta ce vad pe ecran.
Bara de progres din previzualizare e cosmetica, nu reflecta progresul real al descarcarii. Daca in viitor se doreste o bara de progres reala, ar fi nevoie de expunerea progresului de descarcare din raspunsul fetch, ceea ce necesita o abordare diferita fata de simpla citire a corpului raspunsului ca JSON.
