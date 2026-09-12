# STILE-GUIDA DECOBRANDS — Standard grafico e comportamentale

> **Scopo:** documento unico di riferimento trasversale per tutti i progetti Decobrands che utilizzano lo stesso linguaggio visivo: **prototipo CRM** (`prototipo-crm/`) e **webapp Gestione Ordini** (`webapp/`). Raccoglie, senza tralasciare nulla, colori, tipografia, tabelle, badge, card, modali, toast, form, import, formattazione, icone, comportamento JavaScript, stati di caricamento e differenze tra i due progetti.
>
> **Fonti:** `prototipo-crm/assets/css/crm.css`, `prototipo-crm/assets/js/app.js`, `prototipo-crm/assets/js/data.js`, `prototipo-crm/*.html`, `webapp/views/_head.ejs`, `webapp/views/_foot.ejs`, `webapp/views/*.ejs` (tutte le view), `webapp/static/*.js` (tutti i moduli), `specifica-app-gestione-ordini.md` (v1.4.17), `specifica-crm-provvigioni.md` (v1.2).

---

## 1. Stack e CDN

| Componente | Risorsa | Note |
|---|---|---|
| CSS | `https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css` | Bootstrap 5.3.2 |
| JS | `https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/js/bootstrap.bundle.min.js` | bundle (modali, offcanvas, tab, toast, dropdown inclusi) |
| Icone | `https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.3/font/bootstrap-icons.css` | classe `bi bi-…` |
| JSON viewer (solo audit) | `https://cdn.jsdelivr.net/npm/json-formatter-js@2.3.4/dist/json-formatter.min.css` + `…/json-formatter.umd.min.js` | libreria `JSONFormatter(data, indent, opts)` |
| Backend | Express 4 + EJS (SSR), PostgreSQL | porta webapp 8000 |

Regole d'uso CDN: **mai spezzare** — in ogni pagina (anche login) i 3 link vengono inclusi identici.

---

## 2. Palette colori

### 2.1 Variabili di riferimento

```css
:root {
  --bs-primary: #0d6efd;   /* blu Bootstrap primary */
  --db-bg:      #f5f6f8;   /* sfondo pagina */
  --db-border:  #dee2e6;   /* bordo card/tabelle */
}
```

`body { background: var(--db-bg) }` ovunque. Nessun gradiente, nessuna texture di sfondo.

### 2.2 Tabella colore — semantiche Bootstrap usate

| Nome | Hex | Uso tipico |
|---|---|---|
| primary | `#0d6efd` | azioni principali, bordo-sx KPI, icone titolo pagina, link, FAB aiuto, divider attivo, hover tabella |
| success | `#198754` | conferme, badge "Integra/Esportato", qta positive, pallino ok, riga nuova (bordo input), toast ok |
| danger | `#dc3545` | eliminazione, errori, qta negative, pallino ko, mismatch totale, badge stato Disabilitato/Errore |
| warning | `#ffc107` | stati da verificare/attenzione, righe dirty, highlight nuovo ordine, badge "Non visto" |
| info | `#0dcaf0` | badge confermato, badge DUPLICATO, KPI info, tabella codice Integra nelle liste pickup |
| secondary | `#6c757d` | badge righe/contatori, stati neutri, placeholder, colonne testuali attenuate |
| dark | `#212529` | navbar, header tabella, header modal errore, badge ruolo n8n |
| light | `#f8f9fa` | header tabella sticky alternativa (`table-secondary`), input readonly/calc |

### 2.3 Colori "fissi" esadecimali usati direttamente nei template

| Hex | Dove |
|---|---|
| `#f5f6f8` | sfondo pagina (`body`) |
| `#dee2e6` | bordo default (`--db-border`), divider split-pane, bordo `.mfx-bottom`, bordo risultati picker, bordo `pre.det-code` |
| `#0d6efd` | divider in hover/dragging, border input focus, drop-zone drag-over border |
| `#e8f0fe` | sfondo drop-zone in drag-over |
| `#525659` | sfondo pannello PDF (`#pdf-panel`) |
| `#aaa` | testo "PDF non disponibile" (`#no-pdf`) |
| `#fff3cd` | evidenza riga ordine appena ricevuto via SSE (rimossa dopo 5s) |
| `#e9ecef` | bg header tabella delle righe (`tbl-righe`) e thead sticky `.mfx-scroll`, background step stato |
| `#adb5bd` | sfondo pallino "sconosciuto" (`pall-unknown`), bordo drop-zone upload, frecce stepper |
| `#495057` | colore testo intestazioni tabella (prototipo) |
| `#666` | etichette campo testata ordine (`.lbl`) |
| `#888` | etichetta "Righe" pagina ordine |
| `#f8f9fa` | sfondo input readonly / `pre.det-code` |
| `#ffc107`/`#dc3545`/`#198754`/`#0dcaf0` | varianti bordo-sx `.kpi-card`, varianti `.alert-scadenza` |
| `#fff5f5` / `#f0fdf4` / `#fffdf0` | sfondi alert scadenze (danger/ok/warn) |
| `#cfe2ff` + `#084298` | step "active" |
| `#d1e7dd` + `#0f5132` | step "done" |
| `#e9f0fd` + `#cfe0fb` + `#084298` | `.doc-chip` (chip allegato prototipo) |
| `#fbfcfd` | sfondo `.doc-panel` vuoto |
| `rgba(255,193,7,.15)` | riga "changed" nel diff audit |
| `rgba(220,53,69,.1)` | cella "prima" nel diff audit |
| `rgba(25,135,84,.1)` | cella "dopo" nel diff audit |
| `rgba(220,53,69,.25)` | anello focus sul campo totale mismatch |
| `rgba(0,0,0,.06)` / `rgba(0,0,0,.05)` | ombre KPI / screen-card |
| `rgba(0,0,0,.08)` / `rgba(0,0,0,.25)` | hover screen-card / FAB aiuto |
| `rgba(255,255,255,…)` | `bg-secondary bg-opacity-25` per pista strength bar |

### 2.4 Semaforo quantità (giacenze, picker articoli)

| Stato | Classe | Colore | Font |
|---|---|---|---|
| negativa | `.qta-neg` | `#dc3545` | weight 600 |
| zero / nulla | `.qta-zero` | `#6c757d` | normale |
| positiva | `.qta-pos` | `#198754` | weight 600 |

Nel picker articoli le stesse semantiche sono applicate inline: `text-danger fw-bold` (<0), `text-muted` (=0 o null), `text-success fw-bold` (>0). Formato: massimo 3 decimali (`maximumFractionDigits:3`).

---

## 3. Tipografia

- **Font:** stack di sistema Bootstrap (nessun font esterno).
- **Titoli pagina:** `<h5 class="fw-bold mb-3">` (o `mb-0`) con icona Bootstrap `<i class="bi … me-2 text-primary">` seguita dal titolo (es. `bi-list-ul` Ordini, `bi-boxes` Articoli, `bi-arrow-left-right` Alias).
- **Contatore titolo:** se la pagina mostra un totale, `<span class="badge bg-secondary ms-1" style="font-size:.7rem">N</span>` accanto al titolo (Articoli, Giacenze, Kit).
- **Dimensioni tabella:** header `table-dark`; righe a font ridotto:
  - Lista ordini: `font-size:.85rem` (classe su `<table>`)
  - `tbl-g` giacenze / `tbl-art` articoli / `tbl-kit` kit / `tbl-utenti` utenti / `tbl-audit` audit / `.tbl-errori` errori: `font-size:.83rem`
  - `tbl-alias` codici alias: `font-size:.84rem`
  - Tabelle dentro modali: `font-size:.82rem`
  - Tabella righe ordine: input `font-size:.81rem`, thead `font-size:.74rem`
- **Intestazioni tabella (prototipo globale):** `.table thead th { font-weight:600; color:#495057 }`; in webapp l'header è `table-dark` quindi testo bianco.
- **Padding celle tabella (prototipo):** `.table > :not(caption) > * > * { padding: .55rem .75rem }`.
- **Etichette campo:** `.lbl` → `font-size:.76rem; color:#666; margin-bottom:1px` (usata per i campi testata ordine). In audit/errori `.det-label` → `font-size:.75rem; color:#6c757d; margin-bottom:2px; text-transform:uppercase; letter-spacing:.03em`.
- **Etichetta sezione:** tag-like → `font-size:.7rem; font-weight:700; text-transform:uppercase; color:#888` (es. "Righe" + badge conteggio `.68rem`).
- **Testo piccolo:** `text-muted small`; la utility prototipo `.table td .text-muted small { font-size:.8rem }`.
- **Versione app:** `<span class="text-white-50 small opacity-50" style="font-size:.72rem">v1.4.17</span>` in navbar e login.
- **Corsivo placeholder "—":** ogni campo vuoto viene reso con il carattere em-dash `—` (non "n/d").
- **Codice monospace:** numeri P.IVA, codici articolo/Integra, filename → `<code>` (Bootstrap `.code` = monospace).

---

## 4. Navbar (app autenticate)

### 4.1 Struttura (webapp — `_head.ejs`)

```html
<nav class="navbar navbar-dark bg-dark px-3 py-2">
```

- **Brand:** `<a class="navbar-brand d-flex align-items-center gap-2">` con:
  - logo `<img>` quando `clienteConfig.clienteLogo` presente: `max-height:<altezzaNavbar>px; max-width:120px; object-fit:contain; filter:brightness(0) invert(1)`
  - `onerror` → nasconde l'img e mostra in linea la fallback `<i class="bi bi-box-seam">`
  - testo = `clienteConfig.clienteNome`; `.navbar-brand { font-weight: 700 }`
- **Link diretto:** `<a class="nav-link text-white-50">` → "Ordini" con `bi-list-ul me-1`.
- **Menu "Strumenti":** dropdown `nav-link text-white-50 dropdown-toggle` con `bi-tools`; accanto dal menu un'icona segnaletica errori: `<i class="bi bi-exclamation-triangle-fill text-warning" id="nav-err-icon" style="font-size:.85rem">`, nascosta se `erroriAperti===0`, `title="N errore/i non visto/i"`.
- **Dropdown:** `dropdown-menu-end dropdown-menu-dark`, voci con icona `me-2`:
  1. **Errori** — `bi-exclamation-triangle` + `<span class="badge bg-warning text-dark ms-2" id="nav-err-count">`
  2. divider
  3. **Carica PDF** — `bi-file-earmark-pdf`
  4. **Guida** — `bi-book`
  5. **Pulsante aiuto** — `bi-question-circle` (voce interattiva, vedi cap. Help)
  6. *(solo se role admin)* divider, **Clienti** `bi-people`, **Utenti** `bi-person-gear`, **Audit** `bi-journal-text`, divider, **Articoli** `bi-boxes`, **Codici Alias** `bi-arrow-left-right`, **Giacenze** `bi-box-seam`, **Kit** `bi-diagram-3`
- **Chiusura barra:** versione (`v…`), utente corrente `<i class="bi bi-person-fill me-1">username`, pulsante logout in un `<form method="post" action="/logout">` → `btn btn-sm btn-outline-secondary text-white-50 border-secondary py-0 px-2` font `0.78rem`, icona `bi-box-arrow-right`, testo "Esci".
- **"Apri in nuova scheda" automatico** (`_foot.ejs`): ad ogni `dropdown-item[href]` del menu viene aggiunto, via JS, un link esterno `<a class="newtab-ico text-white px-2 py-1">` con `bi-box-arrow-up-right`; la voce diventa flessibile (`d-flex`, `.flex-grow-1`). Stile: `.newtab-ico { opacity:.45; transition:opacity .15s }` → hover `opacity:1; color:#fff!important`. Esclusi `href="#"` o già marcati `data-newtab`.

### 4.2 Prototipo CRM (navbar iniettata da `app.js navbar()`)

Stesso look: `navbar navbar-dark bg-dark px-3 py-2 navbar-expand-lg` con:
- brand logo `assets/img/logo-decobrands.png` (stessi filtri + fallback `bi-box-seam`) + testo "CRM"
- link: **Dashboard** `bi-speedometer2`, **Ordini Fornitore** `bi-truck`, **Bollettini** `bi-receipt`, **Liquidazione** `bi-cash-stack`
- dropdown **Anagrafiche** `bi-gear` → Fornitori `bi-building`, Agenti `bi-people`, Interscambio `bi-arrow-left-right` (menu `dropdown-menu-dark`)
- voce attiva → classe `active` sul `nav-link`
- utente fisso `admin`, versione `VERSIONE_PROTOTIPO`, bottone **Launcher** `bi-grid-fill` (torna a `index.html`)
- toggler mobile `navbar-toggler` + collapse `#mainNav`

---

## 5. Layout pagina

### 5.1 Pagine a tabella piena (webapp)

Blocchi CSS ripetuti in *tutte* le view con tabella:

```css
html,body{height:100%;overflow:hidden}
#page-wrap{display:flex;flex-direction:column;height:calc(100vh - 52px);padding:1rem 1.25rem}
#table-card{flex:1;display:flex;flex-direction:column;overflow:hidden;min-height:0}
#table-card .table-responsive{flex:1;overflow-y:auto}
#table-card .table-responsive thead th{position:sticky;top:0;z-index:1}
```

- L'altezza `calc(100vh - 52px)` compensa la navbar (52px).
- La tabella scorre **solo il corpo**, header sticky. La paginazione è nel `.card-footer`.
- Struttura tipica di una pagina: header con titolo + azioni → (eventuale) `card shadow-sm` barra strumenti → `card shadow-sm#table-card` con `.table-responsive` → `_pagination`.

### 5.2 Pagina dettaglio ordine (split pane)

```css
#layout{display:flex;height:calc(100vh - 52px)}
#pdf-panel{width:50%;min-width:150px;background:#525659;display:flex;flex-direction:column}
#pdf-panel iframe{flex:1;border:none;width:100%}
#divider{width:5px;flex-shrink:0;cursor:col-resize;background:#dee2e6;transition:background .15s}
#divider:hover,#divider.dragging{background:#0d6efd}
#data-panel{flex:1;min-width:200px;overflow:hidden;padding:1rem 1.25rem;display:flex;flex-direction:column}
```

- **Divider risize** (drag): durante il drag → `body{cursor:col-resize;user-select:none}`, tutti gli `iframe` ricevono `pointer-events:none`; ampi `minW=150`, `maxW=layout.offsetWidth-200-5`. Alla fine del drag tutto viene ripristinato.
- **PDF non disponibile** (`#no-pdf`, flex colonna centrata, colore `#aaa`): icona `bi-file-earmark-pdf-fill` 3.5rem, `<strong>PDF non disponibile</strong>`, `small` con istruzioni. Nell'iframe il param URL è `#pagemode=none&navpanes=0&toolbar=1&view=FitH`.
- **Header pannello dati:** bottone indietro `btn btn-sm btn-outline-secondary` con `bi-arrow-left`; titolo `fw-bold "Ordine #id"`; sottotitolo filename troncato `.74rem` grigio con `title` full (troncamento "intelligente": se > 28 caratteri → `base.substring(0,24)+'... .pdf'`); a destra azioni: *Salva Testata* (`btn-sm btn-primary`, `bi-floppy`), *Riga* (`btn-sm btn-success`, `bi-plus-lg`), *Esporta Integra* (`btn-sm btn-primary`, `bi-cloud-download`) **oppure** stato esportato: `btn-sm btn-outline-success disabled` con `bi-check-lg` "Integra esportato" + `btn-sm btn-outline-secondary` `bi-arrow-counterclockwise` (reset); *Elimina* (`btn-sm btn-outline-danger`, `bi-trash`) → apre `#modalEliminaTestata`.

### 5.3 Padding/centricità

- Pagine contenuto (tipo Guida): contenitore centrato `max-width:860px; margin:0 auto; width:100%` in header e scroll.
- `#guida-scroll{flex:1;overflow-y:auto;max-width:860px;margin:0 auto;width:100%}`.

---

## 6. Card

- Standard webapp: `<div class="card shadow-sm">` — bordo override prototipo `.card { border-color: var(--db-border) }`.
- **Barra strumenti**: `card shadow-sm mb-3` con `card-body p-2`; dentro una riga flex `d-flex align-items-center gap-2 flex-wrap` con ricerca, `vr mx-1`, drop-zone e pulsanti. Zona risultati import: `<div id="import-result" class="mt-2" style="display:none">`.
- **Filtri audit/errori**: `card shadow-sm mb-3` + `card-body p-2`, form `row g-2 align-items-end`, label `form-label small mb-0/mb-1`, select/input `form-select-sm`/`form-control-sm`.
- **Card header custom** (coda upload): `card-header d-flex align-items-center justify-content-between py-2 px-3 bg-dark text-white` con titolo `small fw-semibold`.
- KPI e screen-card vanno a cap. specifici.

---

## 7. Tabelle — regole generali

Pattern condivisi:

- `<table class="table table-hover align-middle mb-0">` (lista ordini, utenti, audit, errori) oppure `table-sm` (clienti, articoli, giacenze, kit, codici alias, coda upload).
- Header sempre `thead class="table-dark"`.
- Header dei **sortable** (lista ordini, clienti): `th.sortable` con `<a href="…">` senza underline e icona freccia via CSS:

```css
th.sortable{cursor:pointer;user-select:none;white-space:nowrap}
th.sortable:hover{background-color:#373b3e !important}
th.sortable .sort-icon{margin-left:4px;font-size:.75em;opacity:.5}
th.sortable.asc  .sort-icon::after{content:'▲';opacity:1}                          /* lista ordini: ' ▲' (con spazio), clienti: '▲' */
th.sortable.desc .sort-icon::after{content:'▼';opacity:1}
th.sortable:not(.asc):not(.desc) .sort-icon::after{content:'⇅'}
th.sortable a{color:inherit;text-decoration:none;display:block}
```

> Nota: la lista ordini usa le frecce con spazio iniziale (`' ▲'`, `' ▼'`, `' ⇅'`); clienti le usa identiche ma senza spazio. L'elemento `.sort-icon` è `<span class="sort-icon"></span>` dentro il link.

- **Righe cliccabili con navigazione:** `<tr onclick="location='/…'" style="cursor:pointer">`; le celle con bottoni racchiuse da `onclick="event.stopPropagation()"`.
- **Righe "esportate/lette" attenuate:** `class="table-light text-muted"` (righe con `csv_creato` in lista ordini; righe errore "visto"; righe giacenza…). In `errori.js` la condizione `isVisto` aggiunge `tr.classList.add('table-light','text-muted')`. Lo stesso per righe esportate in lista (`show all`).
- **Riga in errore coda upload:** `table-danger` (riga intera sfondo rosso chiaro).
- **Hover:** `table-hover`; nell'upload le righe picker diventano `table-active` al mouseover via JS.
- **Celle di sola visualizzazione dentro tabella edit:** `code` per codici.
- **Valori nulli in cella:** em-dash `—`.
- **Collegamento cella→ordinamento** in lista ordini: attributo `data-val` con valore normalizzato (minuscolo) per ordinamento client (impostato dal server; non usato altrove).

### 7.1 Tavole speciali: tabella righe ordine (`tbl-righe`)

```css
.tbl-righe th{font-size:.74rem;white-space:nowrap;position:sticky;top:0;z-index:2;background:#e9ecef}
.tbl-righe td{padding:.2rem .3rem;vertical-align:middle}
.tbl-righe input{border:1px solid transparent;border-radius:3px;padding:2px 4px;width:100%;background:transparent;font-size:.81rem}
.tbl-righe input[type="number"]{text-align:right}
.tbl-righe input:focus{outline:none;border-color:#0d6efd;background:#fff}
.tbl-righe tr.dirty input{border-color:#ffc107}
.tbl-righe tr.new-row input{border-color:#198754}
```

- Header tabella `table-secondary`; caption nel primo `th` (width 22px) `title="Presenza in anagrafica articoli"`.
- Colonne: pallino | # | Cod.Int | Cod.Forn | Descrizione (min-width 120px) | UM | Qtà | Prezzo | IVA% | Totale | azioni.
- Azioni per riga: `btn-icon btn-outline-primary` `bi-floppy` (salva) e `btn-icon btn-outline-danger` `bi-trash` (elimina).
- Comportamento:
  - ogni `input` → aggiunge `tr.dirty` + ricalcola totale; se campo `codice_fornitore` → `checkAnagrafica(tr)` con debounce 350ms.
  - `calcolaTotale()` somma `[data-field="totale_riga"]`; aggiorna `.text-end #totale-righe` (`X.XX €` o `—`) e `#f_totale_calcolato`; se `|estratto − calcolato|>0.01` aggiunge classe `.mismatch` al campo calcolato.
  - `#f_totale_calcolato` è `readonly tabindex="-1"`, `background:#f8f9fa;cursor:default`; `.mismatch` → `border-color:#dc3545!important; box-shadow:0 0 0 .2rem rgba(220,53,69,.25)`.
  - `checkPiva()` (debounce 350ms) sul campo P.IVA: aggiorna il pallino `#piva-dot`: vuoto → nessuna classe; durante la chiamata `pall-unknown`; ok → `pall-ok` (title "Cliente in anagrafica — Integra: X · clicca per cambiare"); ko → `pall-ko` (title "P.IVA non presente in anagrafica clienti · clicca per cercare"). Click → apre cliente picker.
  - Tooltip su input troncati: al `mouseenter`, `title = value` se `scrollWidth > clientWidth`.
  - Nuova riga via modal: campi già salvati, numero riga = n righe +1.
  - `aggiungiRiga()` produce riga `data-id="new"` + `class="new-row"`, focus sul primo input.

### 7.2 Tavole picker (articolo/cliente)

- `<table class="table table-sm table-hover mb-0" style="font-size:.83rem">`, header `thead table-secondary` sticky `top:0;z-index:1`.
- Righe cliccabili `cursor:pointer`, hover `table-active`.
- Articolo: colonne Codice (`code`), Descrizione (`text-muted`), UM (`text-muted small`), Giacenza (`text-end`, colori semaforo). Colonna qta formattata max 3 decimali.
- Cliente: Ragione Sociale, P.IVA (`code`), Cod. Integra (`code text-primary`).

---

## 8. Badge — vocabolario completo

### 8.1 Badge "stato" del CRM (prototipo e `.badge-stato`)

Classe base: `.badge-stato { font-size:.72rem; padding:.3em .6em; font-weight:600; text-transform:uppercase; letter-spacing:.3px }`.

| Stato | Classe | Sfondo | Testo |
|---|---|---|---|
| Inviato | `.badge-inviato` | `#ffc107` | `#000` |
| Confermato | `.badge-confermato` | `#0dcaf0` | `#000` |
| Fatturato | `.badge-fatturato` | `#198754` | `#fff` |
| Da verificare | `.badge-da_verificare` | `#ffc107` | `#000` |
| Verificato | `.badge-verificato` | `#198754` | `#fff` |
| Contestato | `.badge-contestato` | `#dc3545` | `#fff` |
| Attivo | `.badge-attivo` | `#198754` | `#fff` |
| Inattivo | `.badge-inattivo` | `#6c757d` | `#fff` |

### 8.2 Badge webapp (utility Bootstrap)

| Contesto | Classi | Testo | Dove |
|---|---|---|---|
| Import riuscito | `bg-success` | "N inviati" | upload ordini risultato |
| Import fallito | `bg-danger` | "N falliti" | upload ordini risultato |
| Righe ordine (conteggio) | `bg-secondary` | numero righe | lista/dettaglio ordine |
| Contatore titolo | `bg-secondary` `font-size:.7rem` | totale | articoli/giacenze/kit |
| Ruolo admin utente | `bg-primary` (testo bianco in Bootstrap 5) | "admin" | utenti |
| Ruolo n8n | `bg-dark` | "n8n" | utenti |
| Ruolo operator | `bg-secondary` | "operator" | utenti |
| Utente abilitato | `bg-success` | "Attivo" | utenti |
| Utente disabilitato | `bg-danger` | "Disabilitato" | utenti |
| Deve cambiare pwd | `bg-warning text-dark` | `bi-key` | utenti (title tooltip) |
| Utente di sistema | `bg-secondary` | `bi-shield-lock` + "sistema" | utenti |
| Op INSERT audit | `bg-success` | "INSERT" | audit |
| Op UPDATE audit | `bg-warning text-dark` | "UPDATE" | audit |
| Op DELETE audit | `bg-danger` | "DELETE" | audit |
| Errore non visto | `bg-warning text-dark` | `bi-eye-slash` "Non visto" | errori |
| Errore visto | `bg-success` | `bi-eye` "Visto" | errori |
| Tipo errore DUPLICATO | `bg-info` | tipo | dettaglio errore |
| Tipo errore generico | `bg-danger` | tipo | dettaglio errore |
| Esportato Integra | `bg-success` | `bi-check-lg` "Integra" | lista ordini |
| Errore navbar count | `bg-warning text-dark` | numero | dropdown Strumenti |
| Contatore tab alias/kit | `bg-secondary` oppure `bg-warning text-dark` (se >0) | numero | modal conferma sostituzioni |
| Stato coda upload | cfr. `STATI` sotto | — | pagina upload |
| Guida — aiuto ctx | `badge rounded-pill bg-primary` `bi-question-lg` | — | testo guida |

**Stati coda upload (`ordini-upload.js STATI`):**

| stato | badge | icona+badge | label |
|---|---|---|---|
| `pending` | `bg-secondary` | `bi-hourglass` | In attesa |
| `processing` | `bg-primary` | `bi-gear spin` | Elaborazione |
| `done` | `bg-success` | `bi-check-circle-fill` | Completato |
| `error` | `bg-danger` | `bi-exclamation-circle` | Errore |

`.stato-badge{font-size:.72rem;letter-spacing:.03em}`. La colonna icona separata: done/failed/processing/pending con `bi-…` + colori `text-success/text-danger/text-primary/text-secondary`, `spin` anima l'icona (vedi §Caricamenti).

---

## 9. Pallini / semafori

```css
.pallino{display:inline-block;width:10px;height:10px;border-radius:50%;vertical-align:middle}
.pall-ok{background:#198754}
.pall-ko{background:#dc3545}
.pall-unknown{background:#adb5bd}
.pall-cell{cursor:pointer}
.pall-cell:hover{background:#eef2f7}
```

- Uso: presenza articolo in anagrafica (colonna pallino righe ordine, cliccabile → apre articolo picker), presenza P.IVA cliente (pallino assoluto dentro il campo, `right:9px;top:50%;transform:translateY(-50%)`, `cursor:pointer` → cliente picker).
- Stato intermedio di caricamento: `pall-unknown` (grigio) con title "Verifica in corso…".

---

## 10. KPI card (dashboard/CRM)

```css
.kpi-card{ border-left:4px solid var(--bs-primary); background:#fff; border-radius:.375rem; padding:1rem 1.25rem; box-shadow:0 1px 3px rgba(0,0,0,.06); }
.kpi-card .kpi-value{ font-size:1.8rem; font-weight:700; line-height:1.1; }
.kpi-card .kpi-label{ font-size:.78rem; text-transform:uppercase; letter-spacing:.5px; color:#6c757d; margin-top:.25rem; }
.kpi-card.kpi-warning{ border-left-color:#ffc107; }
.kpi-card.kpi-danger{ border-left-color:#dc3545; }
.kpi-card.kpi-success{ border-left-color:#198754; }
.kpi-card.kpi-info{ border-left-color:#0dcaf0; }
```

- Struttura: `<div class="kpi-card"><div class="kpi-label">…</div><div class="kpi-value">…</div></div>`.
- Nell'app ordini i "box" informativi della testata sono `col-6/col-3/col-4` in `row g-2` dentro card-body `p-2`. (Nessuna sostituzione col prototipo: il prototipo usa kpi-card.)

---

## 11. Stepper stati pratica (prototipo)

```css
.step-states{display:flex;align-items:center;gap:.35rem;flex-wrap:wrap}
.step-states .step{display:inline-flex;align-items:center;gap:.35rem;font-size:.8rem;padding:.3rem .65rem;border-radius:2rem;background:#e9ecef;color:#6c757d;font-weight:600}
.step-states .step.active{background:#cfe2ff;color:#084298}
.step-states .step.done{background:#d1e7dd;color:#0f5132}
.step-arrow{color:#adb5bd;font-size:.7rem}
```

---

## 12. Alert / Note

- **Alert scadenze** (CRM): `.alert-scadenza { border-left:4px solid #dc3545; background:#fff5f5; padding:.6rem 1rem; border-radius:.25rem; margin-bottom:.5rem; font-size:.88rem }` con varianti `.alert-scadenza.scadenza-ok` / `.scadenza-warn` (verde/giallo).
- **Alert webapp**: format standard Bootstrap con `py-2 small` (es. errori login, disclaimer upload, box "Il file va in errore?").
  - Disclaimer upload: `alert alert-warning small` con `border-left:4px solid #ffc107`, `<p class="fw-bold mb-2">` + `bi-exclamation-triangle-fill`, elenchi `ul mb-2 ps-3`.
  - Visto errori: `alert alert-success py-2 small` con `bi-eye`.
- **Nota prototipo** (in demo, es. §4.4/§5.7): `alert-light border` + piccola `text-muted`.
- **doc-panel / doc-chip** (prototipo, allegati pratica): pannello `border:1px dashed var(--db-border); border-radius:.5rem; background:#fbfcfd`; con allegato `border-style:solid; background:#fff`; chip allegato `display:inline-flex; gap:.5rem; background:#e9f0fd; border:1px solid #cfe0fb; color:#084298; padding:.35rem .7rem; border-radius:.4rem; font-size:.85rem`.
- **Nuovo errore (push)**: `.nuovo-errore-alert{position:fixed;top:70px;right:1.5rem;z-index:9998;animation:slideIn .3s ease}`; `@keyframes slideIn{from{transform:translateX(120%)}to{transform:translateX(0)}}`. Composizione: `alert alert-warning alert-dismissible shadow py-2 px-3`, `min-width:260px;font-size:.85rem`, icona `bi-exclamation-triangle-fill`, `<strong>Nuovo errore ricevuto</strong> #id`, auto-rimozione dopo 8s + pulsante close.

---

## 13. Form

- Etichette: `form-label` (default), oppure `form-label small fw-semibold` (login/password/utenti).
- Campi obbligatori: `<span class="text-danger">*</span>` accanto all'etichetta.
- Dimensioni: `form-control`/`form-select`/`input-group` **sm** nei tool/form filtro; grandi solo in modali utenti/login.
- `input-group` con bottoni icona dentro (es. password, picker): `<button class="btn btn-outline-secondary" type="button" tabindex="-1">`.
- **Caselle**: `form-check` + `form-check-input`; switch `form-switch` con `role="switch"` centrato nella cella (`d-flex justify-content-center m-0`).
- **Textarea**: `form-control form-control-sm` + `rows`.
- **Input readonly**: `disabled` attr (es. "Email mittente") o `readonly` con `background:#f8f9fa` (campo descrizione "dalla scheda articolo", totale calcolato).
- **Placeholder** sempre in italiano, minuscolo, con `…` (es. "Cerca codice, descrizione, EAN…", "utente@dominio.it", "Codice articolo componente").
- **Numeri**: `type="number"` con `step` mirato (`0.01` per prezzi/totali, `any` per quantità/IVA, `min` quando serve). Allineamento `text-end` per i numeri nei form/modali.
- **Date**: `type="date"` con valore ISO `yyyy-mm-dd`.
- **Password**: input + occhietto `bi-eye`/`bi-eye-slash` in `input-group`; `togglePwd(id,btn)` cambia tipo e icona. Minimo 8 caratteri (`minlength="8"`).
- **Strength password** (`cambia-password`, `reset-password`):
  - pista: `<div class="mt-1 bg-secondary bg-opacity-25 rounded" style="height:4px">` + barra `.strength-bar` (`height:4px;border-radius:2px;transition:width .3s,background .3s`).
  - Score (nx condizioni): `length>=8`, `length>=12`, `[A-Z]`, `[0-9]`, `[^A-Za-z0-9]`.
  - Mapping: 0 `0% bg-danger` (nessun testo), 1 `20% bg-danger` "Molto debole", 2 `40% bg-warning` "Debole", 3 `60% bg-info` "Discreta", 4 `80% bg-primary` "Buona", 5 `100% bg-success` "Ottima". Etichetta sotto in `.form-text`.

### 13.1 Import anagrafiche (drop zone)

- Drop zone: `<div id="drop-zone" class="form-control form-control-sm d-flex align-items-center gap-2" style="width:260px;border-style:dashed" onclick="document.getElementById('…').click()">`; input file `display:none` (`#drop-zone input[type=file]{display:none}`); icona `bi-file-earmark-spreadsheet text-muted`; testo placeholder `text-muted small text-truncate` (es. "Seleziona o trascina CSV / Excel…", "Seleziona Prodotti.xlsx…", "Seleziona export magazzino…", "Seleziona COMPARATIVA CODICI…").
- Drag-over: `.drag-over{border-color:#0d6efd !important;background:#e8f0fe}` (transition `.2s`).
- **Upload PDF ordini**: drop zone grande `border border-2 border-dashed rounded-2 d-flex flex-column align-items-center justify-content-center p-3`, `border-color:#adb5bd !important`, `min-height:72px`, icona `bi-cloud-arrow-up fs-2 text-muted`, "Solo file .pdf" a `.7rem`. Filtra `accept=".pdf" multiple`.
- Pulsante import: `btn btn-sm btn-primary` con `bi-cloud-upload me-1`, `disabled` finché nessun file; durante il lavoro viene sostituito dallo spinner + testo "Importazione…" (varia per pagina).
- Esito: zona `#import-result` (nascosta), poi alert success/danger con riepilogo; su successo `showToast` + `location.reload()` dopo **1200ms** (nei moduli import webapp); testo in coda upload: badge `bg-success` "N inviati" / `bg-danger` "N falliti" + lista errato `text-danger small` separati da ` · `.
- Template scaricabili (clienti): dropdown bottone `btn btn-sm btn-outline-secondary dropdown-toggle` "Template" → Excel `bi-file-earmark-excel text-success`, CSV `bi-file-earmark-text`.

---

## 14. Bottone icona & azioni compatte

```css
.btn-icon{ padding:2px 7px; font-size:.8rem; }
```

- Varianti usate: `btn-outline-primary` (esporta/salva/modifica/componenti), `btn-outline-danger` (elimina), `btn-outline-warning` (reset pwd, urgenze), `btn-outline-secondary` (nuova scheda/altro), `btn-danger` (elimina file coda), `btn-warning` (rielabora), `btn-success` (aggiungi componente).
- Tabella lista ordini ha anche `.btn-del{padding:2px 7px;font-size:.8rem;line-height:1.2}`.
- Sempre `title` esplicativo sul pulsante.
- Durante operazioni asincrone il bottone mostra `<span class="spinner-border spinner-border-sm"></span>` (o il segno `⟳` ruotato) e viene disabilitato.

---

## 15. Modali

### 15.1 Sistema "modal-fixed" (3 taglie)

```css
.modal-fixed .modal-dialog{height:var(--mfx-h);max-width:var(--mfx-w);width:calc(100% - 2rem)}
.modal-fixed.mfx-sm{--mfx-w:480px;--mfx-h:62vh}
.modal-fixed.mfx-md{--mfx-w:840px;--mfx-h:76vh}
.modal-fixed.mfx-lg{--mfx-w:1160px;--mfx-h:86vh}
.modal-fixed .modal-content{height:100%;display:flex;flex-direction:column;overflow:hidden}
.modal-fixed .modal-header,.modal-fixed .modal-footer{flex:0 0 auto}
.modal-fixed .modal-body{flex:1 1 auto;min-height:0;display:flex;flex-direction:column;overflow:hidden}
.mfx-top{flex:0 0 auto;margin-bottom:.5rem}
.mfx-bottom{flex:0 0 auto;margin-top:.6rem;border-top:1px solid #dee2e6;padding-top:.6rem}
.mfx-scroll{flex:1 1 auto;min-height:0;overflow-y:auto}
.mfx-scroll table{border-collapse:separate;border-spacing:0}
.mfx-scroll table thead th{position:sticky;top:0;z-index:3;background:#e9ecef;box-shadow:inset 0 -1px 0 #adb5bd,inset 0 1px 0 #adb5bd}
```

- Tab dentro modali fissi: `.modal-fixed .tab-content{flex:1 1 auto;min-height:0;display:flex}` e `.tab-pane.active{display:flex;flex-direction:column;min-height:0}` (`width:100%`).
- Usato da: picker articolo/cliente (`mfx-md`), componenti kit (`mfx-md`), conferma sostituzioni alias/kit (`mfx-lg`).

### 15.2 Modali standard

- Header: `<div class="modal-header">` con `<h5 class="modal-title">` (icona `me-2` se pertinente) e `btn-close`.
- Footer: `Annulla` (`btn btn-secondary`) a sinistra, azione primaria `btn-primary` con `bi-check-lg me-1` ("Salva", "Crea Kit", "Aggiungi riga", "Imposta password").
- Versioni compatte: `py-2` su header/footer, `modal-sm` per le conferme distruttive.
- **Conferma distruttiva** (elimina ordine/riga): `modal-dialog modal-dialog-centered modal-sm`, header `py-2 border-bottom-0` con `h6` + `bi-exclamation-triangle-fill text-danger me-2`, body `text-center text-muted small` (con `<strong>` sul soggetto, "non è reversibile"), footer centrato `justify-content-center gap-2 border-top-0` con `btn-sm btn-outline-secondary` Annulla + `btn-sm btn-danger` Elimina.
- **Dettaglio errore**: `modal-dialog modal-lg modal-dialog-scrollable`, header `bg-dark text-white py-2` con title `d-flex … gap-2`, badge tipo, data e stato nel titolo, `btn-close-white`. Body con `det-label`/`det-value`, `<pre class="det-code">`, collasso "Contesto JSON". Footer con pulsanti specifici (PDF a sinistra con `me-auto`, Chiudi `btn-secondary`, segna visto/non visto).
- **Modal "Nuova Riga"**: campi in griglia `row g-2` con `col-12/col-3`, etichette `form-label small mb-1`, input `form-control-sm text-end` per numeri, descrizione readonly; ricalcolo automatico totale = qta×prezzo (solo se l'utente non ha editato manualmente il totale).
- **Modal Utente**: username, email (asterisco + `form-text` "L'utente riceverà la password provvisoria su questo indirizzo."), ruolo (`operator`/`admin`), checkbox Abilitato.

### 15.3 Viewer JSON / Diff (audit)

- Header con tabs: `btn-group btn-group-sm`, tab attivo `btn btn-sm btn-dark active`, non attivi `btn-outline-secondary`; icona tab diff `bi-layout-split me-1`.
- Bottone copia: `btn btn-sm btn-outline-secondary` `bi-clipboard` → al click diventa `btn-success` con `bi-clipboard-check` per 2s (poi torna).
- Body: `modal-body p-0` con `max-height:72vh;overflow:auto`; container JSON `#modalJsonContainer{font-size:.82rem;line-height:1.5;padding:1rem}` e regole per `.json-formatter-row{padding:1px 0}`.
- **Diff table** (`.diff-tbl`): `table table-sm mb-0`, `font-size:.82rem`, header `table-dark` sticky `top:0;z-index:1`, colonne Campo 22% / Prima 39% / Dopo 39%.
  - riga modificata → `.diff-changed{background-color:rgba(255,193,7,.15)!important}`; cella prima `.diff-cell-before{background-color:rgba(220,53,69,.1)!important}`; cella dopo `.diff-cell-after{background-color:rgba(25,135,84,.1)!important}`.
  - gruppi (`.testata`, `.riga`, `.righe`, altri) con riga intestazione `table-secondary` `fw-bold py-1 small text-uppercase tracking-wide` (testo uppercase con letter-spacing); ordine: testata → altri → riga → righe.
  - valori: null → `text-muted fst-italic "null"`; oggetto → `<pre>` `.76rem` pre-wrap max-height 120px; boolean → `text-primary fw-bold`; numero → `text-success`.
- `JSONFormatter(data, 2, { hoverPreviewEnabled:true, hoverPreviewArrayCount:5, hoverPreviewFieldCount:5 })`.

---

## 16. Picker articolo/cliente (autocomplete "modale")

- Apertura: `openArticoloPicker(cb, [valoreIniziale])` / `openClientePicker(cb, [valoreIniziale])`.
- Config: header `py-2`, title "Cerca Articolo"/"Cerca Cliente" con `bi-search`; body `p-2`.
- Ricerca: `input-group mfx-top` con `<span class="input-group-text"><i class="bi bi-search"></i></span>`, campo `ap-search`/`cp-search` (`autocomplete="off"`), pulsante ✕ (`btn btn-outline-secondary`, nascosto, `apClear()`).
- Risultati: `#ap-results`/`#cp-results` = `mfx-scroll border:1px solid #dee2e6;border-radius:4px`.
- Stati del contenitore risultati:
  - placeholder: `text-center text-muted py-4 small` + `bi-keyboard` "Inizia a digitare per cercare…"
  - caricamento: `text-center text-muted py-3 small` + `spinner-border spinner-border-sm` "Ricerca in corso…"
  - errore: `text-center text-danger py-3 small` + `bi-exclamation-circle <msg>`
  - nessun risultato: `text-center text-muted py-3 small` "Nessun … trovato per "<strong>q</strong>"."
- Debounce **280ms**; invio su Enter; sostituisce i risultati da `<tbody>` a `<table>` con header sticky.
- Selezione: riga `cursor:pointer`, `table-active` al mouseover; click → nasconde modal e chiama callback.

### 16.1 Autocomplete "tendina" (articolo, solo Kit)

`attachArticoloAutocomplete(input, cb)`:
- dropdown `div.list-group.shadow` `position:fixed;z-index:2000;max-height:260px;overflow-y:auto;font-size:.83rem` posizionato sotto l'input (`top: r.bottom+2px; width:r.width`), riposizionato su scroll/resize.
- ricerca a partire da **2 caratteri**, debounce **250ms**, `limit=20`.
- item: `button.list-group-item list-group-item-action py-1`, attivo `class="active"`, `<code class="text-white">codice</code>` + `<span class="text-muted">descrizione</span>`.
- navigazione frecce su/giù, Enter seleziona, Esc chiude; chiusura al blur (ritardo 150ms) e dopo selezione (mousedown, non click, per precedere il blur).

---

## 17. Toast e notifiche

- Container: `.toast-container{position:fixed;bottom:1.5rem;right:1.5rem;z-index:9999}` (iniettato da `_foot.ejs` / app.js).
- Elemento: `<div id="toast" class="toast text-white border-0" role="alert"><div class="d-flex"><div class="toast-body" id="toast-msg"></div><button class="btn-close btn-close-white me-2 m-auto" data-bs-dismiss="toast"></button></div></div>`.
- Funzione `showToast(msg, ok=true)`: swap di classe `bg-${ok?'success':'danger'}`, `delay:3000`, `bootstrap.Toast.getOrCreateInstance(...).show()`. Messaggio col `textContent` (nessun HTML).
- Fallback prototipo: se manca `#toast` → `alert((ok?'':'ERRORE: ')+msg)` (usato fuori webapp).
- Messaggi tipici: `'… generato ✓'`, `'… salvato ✓'`, `'Riga eliminata'`, `'Errore: ' + e.message` (rosso), `'File Integra generato ✓'`.

---

## 18. Paginazione

- Partial `_pagination.ejs`; parametri `{ page, totalPages, total, baseUrl }` (query senza `&page=`).
- Solo se `totalPages>1 || total>0` → `.card-footer d-flex align-items-center justify-content-between py-2 flex-wrap gap-2`.
- Info: `Pagina **X** di **Y** · N record totali`.
- Nav: `ul.pagination.pagination-sm mb-0`; elementi `«`, `‹`, eventuali `…` (pulsante `disabled` con `span.page-link`), numeri (window ±2), `›`, `»`; attivo `.active`, disabilitato `.disabled`; first/last hanno `title`.
- Pagina **errori** usa una paginazione uguale ma renderizzata via JS in `#pagination-errori` (stessa markup, `onclick="goPage(n)"`).

---

## 19. Empty state (lista/ricerca vuota)

- Tabelle SSR: `<tr><td colspan="N" class="text-center py-4 py-5 text-muted">Messaggio</td></tr>`.
- Pagine senza righe ma con layout flex: `<div class="text-center text-muted py-5">` con icona grande `fs-1 d-block mb-2` (es. `bi-people`, `bi-boxes`, `bi-box-seam`, `bi-arrow-left-right`, `bi-diagram-3`, `bi-inbox`) e messaggio contestuale CTA (importa file, nuovo kit, ecc.).
- Messaggi chiave: "Nessun ordine trovato", "Nessun cliente presente. Importa un file per iniziare.", "Importa l'anagrafica da Integra per iniziare.", "Nessun file in coda", "Nessun risultato per «x»."

---

## 20. Real-time, SSE, polling

- **SSE globale** (`_foot.ejs`): `window._sseGlobal = new EventSource('/api/ordini/stream')`; `onerror` silenzioso (riconnessione automatica); alla `beforeunload` viene chiusa (evita saturazione pool HTTP/1.1 — limite 6/host).
- Evento `new-order` (pagina ordini): inserisce la riga `tr` in testa `<tbody>` con `background:#fff3cd`, `cursor:pointer`, click → dettaglio; contatore `#count-label = N ordini`; rimozione highlight dopo 5s; gestione duplicate per `data-id`; elimina la riga "nessun ordine" se presente.
- Evento `nuovo_errore`: su tutte le pagine aggiorna `aggiornaNavBadge()`; nella pagina errori crea `.nuovo-errore-alert` (8s di vita) + `carica()`.
- `aggiornaNavBadge()`: GET `/api/errori/count`, aggiorna `nav-err-icon` (display + title "N errori non visti") e `nav-err-count` (testo + display).
- **Polling coda upload**: ogni **3000ms** `aggiornaTabella()`; `visibilitychange` mette in pausa (indicatore `poll-indicator` `bi-circle-fill` passa da `text-success` a `text-secondary`); `pagehide` → clear interval + abort controller. Transizioni: riga appena `done` resta con badge verde ~1.8s poi `fadeOutRow` (opacity `.7s`, rimozione a 750ms) + DELETE dall'API.
- **Pagina lista ordini** riusa `window._sseGlobal` (o ne crea una) ed ascolta `new-order`.

---

## 21. Formattazione dei valori (convention comune)

- **Euro**: `€` + `toLocaleString('it-IT',{minimumFractionDigits:2})` — helper webapp/prototipo `fmtEuro`. In lista ordine: `_tg.toFixed(2) + ' €'` (spazio prima di €); nel totale calc `toFixed(2)+' €'`. In messaggi generici: `'123,45 €'`.
- **Numeri**: `toLocaleString('it-IT',{maximumFractionDigits:3})` per quantità/giacenze; `fmtNum(n, dec)`.
- **Percentuali**: `Number(n).toFixed(2)+'%'` (prototipo `fmtPct`).
- **Date**: solo data → `dd/mm/yyyy` (spezzando la stringa ISO `substring(0,10).split('-').reverse().join('/')` oppure `toLocaleDateString('it-IT',{day:'2-digit',month:'2-digit',year:'numeric'})`).
- **Data+ora**: `dd/mm/yyyy hh:mm` (stringa ISO: `substring(0,10).split('-').reverse().join('/') + ' ' + substring(11,16)`); audit aggiunge i secondi `hh:mm:ss`. Locale `.toLocaleString('it-IT')` per campi come "Aggiornato il"; `new Date(...).toLocaleString('it-IT')`.
- **ISO invio**: `toISOString().substring(0,10)` per date, `(0,16)` per datetime.
- **Import file**: CSV sempre con **BOM `\uFEFF`** e separatore `;` (`exportCSV(filename, header, rows)`); quote/èscape per valori con `;`, `"`, `\n` (`escI` nei route csv). File Integra = XLSX `yyyymmdd_<codiceIntegra>_<prog>.xlsx`, 28 colonne `mvt_`/`mvr_`, algoritmo split alias & kit.
- **Valori vuoti**: em-dash `—`.
- **JSON nel messaggio errore**: `tryFmt` → `JSON.stringify(JSON.parse(s), null, 2)` (pretty) o stringa grezza.

---

## 22. Icone Bootstrap — mappa delle azioni

| Azione | Icona | Dove |
|---|---|---|
| Titolo lista ordini | `bi-box-seam` | lista ordini, giacenze (titolo/empty), fallback brand |
| Titolo articoli | `bi-boxes` | articoli |
| Titolo clienti / utenti | `bi-people` | clienti, utenti, menu |
| Gestione utenti | `bi-person-gear` | menu Strumenti |
| Kit / distinte | `bi-diagram-3` | kit (titolo, empty, modal) |
| Codici alias / interscambio | `bi-arrow-left-right` | codici-alias, interscambio prototipo, menu |
| Audit log | `bi-journal-text` | audit |
| Errori | `bi-exclamation-triangle` / `-fill` | menu, icona navbar, warning alert |
| Carica PDF | `bi-file-earmark-pdf` (menu/righe coda) / `bi-file-earmark-pdf-fill` (no-pdf, 3.5rem) | menu, upload |
| Guida | `bi-book` | menu, guida |
| Aiuto context | `bi-question-lg` (FAB), `bi-question-circle` (toggle menu) | help |
| Pagina guida | `bi-book` | guida |
| Ricerca | `bi-search` | bottoni ricerca, picker |
| Esporta Integra | `bi-cloud-download` | lista/dettaglio |
| Import | `bi-cloud-upload` | bottoni import, upload |
| Salva | `bi-floppy` | testata/riga, modal salva |
| Aggiungi | `bi-plus-lg` | nuova riga, nuovo kit, nuovo utente, componente |
| Elimina | `bi-trash` | ordini, righe, utenti; `bi-trash3` in clienti/kit/alias |
| Modifica utente | `bi-pencil` | utenti |
| Reset password utente | `bi-key` | utenti, cambia-password title |
| Evoluzione stato | `bi-check-lg` | "Integra esportato", conferme modali |
| Torna indietro | `bi-arrow-left` | dettaglio ordine, upload |
| Torna agli ordini | `bi-arrow-left` | upload |
| Reset export | `bi-arrow-counterclockwise` | dettaglio ordine (csv-reset) |
| Aggiorna/rielabora | `bi-arrow-clockwise` | coda upload (rielabora) |
| Tag _order | `bi-tag` | coda upload (forza) |
| Apri nuova scheda | `bi-box-arrow-up-right` | voci menu, link ordine esistente |
| Esci | `bi-box-arrow-right` | navbar, login |
| Accedi | `bi-box-arrow-in-right` | login |
| Stato visto | `bi-eye` / `bi-eye-slash` | errori, badge, segna non visto |
| Password occhio | `bi-eye` / `bi-eye-slash` | toggle password |
| File allegato | `bi-file-earmark-spreadsheet` | drop-zone import |
| Template excel | `bi-file-earmark-excel` | dropdown template (verde) |
| Template csv | `bi-file-earmark-text` | dropdown template |
| Clock | `bi-clock-history` | "Aggiornato", coda card header |
| Filtra | `bi-funnel` | audit, errori, filtri |
| Reset filtri | `bi-x-lg` | reset ricerca/filtri, clear picker |
| Utente corrente | `bi-person-fill` | navbar, prototipo |
| Sistema | `bi-shield-lock` | badge utente sistema, cambia-password icon |
| Info | `bi-info-circle` | note modale alias, schede |
| Keyboard (placeholder ricerca) | `bi-keyboard` | placeholder picker |
| Braces JSON | `bi-braces` | bottone json audit (prima/dopo) |
| Split diff | `bi-layout-split` | bottone diff, tab diff |
| Copia clipboard | `bi-clipboard` / `bi-clipboard-check` | audit |
| Codice/contesto | `bi-code-slash` | collasso contesto JSON |
| Chevron down | `bi-chevron-down` | collapsible contesto, header sezioni guida |
| Speedometer (dashboard) | `bi-speedometer2` | prototipo |
| Ordini fornitore | `bi-truck` | prototipo |
| Bollettini | `bi-receipt` | prototipo |
| Liquidazione | `bi-cash-stack` | prototipo |
| Fornitori | `bi-building` | prototipo |
| Strumenti/meccanica | `bi-tools`, `bi-gear` | menu Strumenti, dropdown Anagrafiche |
| Launcher | `bi-grid-fill` | bottone launcher prototipo |
| Upload cloud | `bi-cloud-arrow-up` | drop-zone upload PDF |
| Inbox (empty) | `bi-inbox` | coda upload vuota |
| Inviato/check | `bi-send` | richiedi reset |
| Envelope | `bi-envelope-open` | richiedi reset |
| Spinner | `bi-gear spin` / `⟳` / `spinner-border` | stati elaborazione |

Regola d'uso: l'icona precede il testo con `me-1`/`me-2` a seconda del contesto; i bottoni con solo icona hanno sempre `title`.

---

## 23. Caricamenti/spinner

- `spinner-border spinner-border-sm` usato nei pulsanti (es. esporta), nei picker ("Ricerca in corso…"), nel dettaglio errore ("Caricamento…").
- **Rotazione continua icona Bootstrap**: `.spin{display:inline-block}` + `.spin::before{display:inline-block;animation:spin 1s linear infinite;transform-origin:50% 50%}` con `@keyframes spin{to{transform:rotate(360deg)}}` (usato su `bi-gear` e su `⟳`).
- Durante upload: il bottone diventa `disabled` e il contenuto `<span class="spin me-1">⟳</span>Caricamento…` / `spinner-border spinner-border-sm` (nei vari moduli).

---

## 24. Login & pagine di autenticazione

- Pagine standalone (no `_head/_foot`): `body { background:#f5f6f8; display:flex; align-items:center; justify-content:center; min-height:100vh; }` (in prototipo: `body.login` con stessi stili).
- Card wrapper: `.login-card` (max-width 360px), `.card-wrap` (max-width 400px cambia pwd/reset / 380px richiedi).
- Header centrato `text-center mb-4`: logo (max-height altezzaLogin, `filter` solo navbar; nel login l'img NESSUNA invert) + fallback `bi-box-seam fs-1 text-primary`; `<h5 class="fw-bold mt-2 mb-0">` nome cliente; `<small class="text-muted">Accedi per continuare</small>`; versione `.7rem opacity .6`.
- Errori: `alert alert-danger py-2 small` con `bi-exclamation-circle me-1`.
- Successo reset: `alert alert-success py-2 small` con `bi-check-circle me-1`.
- Bottoni full-width `btn btn-primary w-100` con icona: Accedi `bi-box-arrow-in-right` (login e successo reset), Invia link di reset `bi-send`, Imposta password `bi-check-lg`.
- Link secondari centrati `text-center mt-3` con `text-muted small`: "Password dimenticata?", "Torna al login" (`bi-arrow-left`).

---

## 25. Guida & aiuto contestuale (FAB/offcanvas)

- **Pagina /guida**: lista di sezioni renderizzata da `help-data.js` → `sectionCard`:
  `<div class="help-sec card mb-2" data-title data-id>` con `card-header py-2 d-flex align-items-center gap-2 cursor:pointer` (toggle `this.nextElementSibling.classList.toggle('d-none')`), icona sezione `text-primary`, `<strong class="flex-grow-1">title</strong>`, freccia `bi-chevron-down text-muted small`; body `card-body py-2 small` (aperto se filtro). Hover header: `.help-sec .card-header:hover{background:#f1f3f5}`; un `.help-sec .card-body p:last-child{margin-bottom:0}`.
  Ricerca live sull'input (filtra per titolo o testo html).
- **FAB aiuto** (tutte le pagine tranne `/login`): pulsante fisso `position:fixed;right:18px;bottom:18px;z-index:1050;width:46px;height:46px;border-radius:50%;border:none;background:#0d6efd;color:#fff;box-shadow:0 2px 8px rgba(0,0,0,.25);font-size:1.3rem;cursor:pointer` con `bi-question-lg`.
  - On/off persistente: `localStorage['help_fab_hidden']`; il menu "Strumenti → Pulsante aiuto" mostra lo stato `Pulsante aiuto: OFF` (rosso) / `ON` (verde); con toast di conferma.
  - Click → offcanvas laterale width 420px, header `border-bottom` con `bi-life-preserver text-primary` e title "Aiuto"; body con sezioni contestuali (prima aperta) o messaggio "Per questa pagina non c'è un aiuto specifico. Apri la guida completa qui sotto."; footer link `btn btn-outline-primary btn-sm w-100 mt-2` "Apri la guida completa" (`bi-book`).
  - Matching per percorso: pattern esatti, `:id` (regex `\d+`), suffisso `/*`.
- Guida: pulsante `badge rounded-pill bg-primary` `bi-question-lg` menzionato come indice.

---

## 26. Conventions di progetto (regole di stile e copy)

- Lingua interfaccia: **italiano**, maiuscole/minuscole come da tabella, i messaggi di stato usano `✓`; i messaggi di errore dicono "Errore: …".
- Titolo pagina: icona + nome; titolo finestra `<title>` in italiano.
- Placeholder: minuscolo, con "…", espliciti ("Cerca …", "utente@dominio.it").
- Troncamento: filename in dettaglio ordine (MAX_BASE 28 → `base.substring(0,24)+'... .pdf'`); celle elenco errori con `text-truncate` + `max-width` + `title` col valore pieno.
- Numeri/date/it-IT come da §21; divise sempre `€` dopo i numeri con spazio (`123,45 €`).
- Nessun framework UI aggiuntivo oltre Bootstrap; nessun font esterno; niente animazioni oltre al set indicato (fade-out righe, slideIn alert, spin, transition colori).
- Segnaletica accessibilità di base: `role="alert"` sui toast, `role="tab"` sui tab, `aria-hidden`, `title` su tutti i bottoni icona, `alt` sui logo, `lang="it"`.
- Responsive minimo: flex wrap nelle barre strumenti (`.flex-wrap`), griglie `row g-2`/`col-*`; layout split-pane a pannelli bassi su schermi ridotti non previsto (webapp desktop-first, si scorrono tabelle).
- Ruoli/permessi influenzano i menu (voci admin solo per `role==='admin'`).

---

## 27. Differenze Prototipo CRM ↔ Webapp

| Aspetto | Prototipo CRM (`prototipo-crm/`) | Webapp (`webapp/`) |
|---|---|---|
| Rendering | HTML statico + JS/data iniettati | EJS SSR + API |
| Navbar | generata da `navbar()` (link dashboard/ordini-f/bollettini/liquidazione + dropdown Anagrafiche) | statica in `_head.ejs` (Ordini + dropdown Strumenti) |
| Auth | utente demo fisso `admin` | login + sessioni + remember token + ruoli (`operator`/`admin`/`n8n`), cambia/reset password |
| Stato ordini | `inviato`/`confermato`/`fatturato` con badge `.badge-stato` | stato derivato da priorità: evidenza righe, pallini anagrafica, badge esportazione |
| Motore provvigioni | calcolo client-side (`commissionePerOrdine`, scaglioni, interscambio) | — (ambito elaborazione ordini/Integra) |
| Dati | `DATA` in `data.js` (26 clienti, 5 fornitori, scaglioni Q3 2026, OGGI demo `2026-09-04`) | DB PostgreSQL reale + audit via trigger |
| Real-time | — | SSE `/api/ordini/stream`, poll coda 3s, badge errori live |
| Layout pagine | pagine scrollabili + navbar | desktop-first `calc(100vh-52px)` con tabelle sticky |
| Modali picker | — | modal-fixed mfx-md/lg + offcanvas aiuto |
| Export provvigioni | CSV client-side (BOM `\uFEFF`, `;`) | XLSX Integra server-side (28 colonne) |

Elementi **condivisi identici**: palette, navbar dark, toast, formattazione it-IT, utility `.btn-icon`, badge stato, KPI card, drop-zone, paginazione, `—` per i vuoti, iconografia Bootstrap.

---

## 28. File di riferimento per esteso

| File | Contenuto chiave |
|---|---|
| `webapp/views/_head.ejs` | stili globali, `.modal-fixed`/`.mfx-*`, navbar |
| `webapp/views/_foot.ejs` | toast, `apiFetch`, `_sseGlobal`, `aggiornaNavBadge`, newtab |
| `webapp/views/_pagination.ejs` | paginazione |
| `webapp/views/_modal-articolo.ejs`, `_modal-cliente.ejs` | modali picker riutilizzabili |
| `webapp/views/lista.ejs` + `static/lista.js` | lista ordini, sort, SSE new-order, export CSV Integra, delete |
| `webapp/views/ordine.ejs` + `static/ordine.js` | split pane, testata, righe, alias/kit conferma, CSV Integra |
| `webapp/views/clienti.ejs`, `articoli.ejs`, `giacenze.ejs`, `kit.ejs`, `codici-alias.ejs` | anagrafiche + import |
| `webapp/views/utenti.ejs`, `audit.ejs`, `errori.ejs` | utenti, JSON viewer/diff, errori processing |
| `webapp/views/ordini-upload.ejs` + `static/ordini-upload.js` | drop zone PDF + coda + poll |
| `webapp/views/guida.ejs`, `static/help.js`, `static/help-data.js` | guida, FAB, offcanvas, sezioni |
| `webapp/views/login.ejs`, `cambia-password.ejs`, `richiedi-reset.ejs`, `reset-password.ejs` | auth |
| `webapp/static/articolo-picker.js`, `cliente-picker.js`, `articolo-autocomplete.js` | picker/autocomplete |
| `prototipo-crm/assets/css/crm.css` | palette, badge-stato, KPI, alert-scadenza, stepper, doc-panel, launcher |
| `prototipo-crm/assets/js/app.js` | formattazione, motore provvigioni, navbar, toast, exportCSV |
| `prototipo-crm/assets/js/data.js` | seed demo (cliente/fornitore/scaglioni/ordini/bollettini/interscambio) |
| `specifica-app-gestione-ordini.md` | specifica v1.4.17 (auth, configurazione, API, real-time) |
| `specifica-crm-provvigioni.md` | specifica CRM v1.2 (regole provvigioni, sezioni UI) |