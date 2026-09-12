# Palette colori Decobrands - Documento di riferimento

> **Fonte canonica**: STILE-GUIDA-DECOBRANDS.md - sez. 2 "Palette colori" (copia nel progetto: `docs/STILE-GUIDA-DECOBRANDS.md`).
> Questo documento **centralizza** la palette, la descrive contesto per contesto e ne dichiara l'**uso previsto nella futura applicazione** (design-first, accessibile, dark-mode-ready).
> Versione: 1.0 - allineata a specifica-sistema-marketing-ai.md (v1.2) e vista-imprenditore.md (v1.0)

---

## 1. Principi di base

| Regola | Dettaglio |
|---|---|
| **Sfondo sempre neutro** | `body { background: var(--db-bg) }`. Niente gradiente, niente texture di sfondo. |
| **Semantica > decorazione** | Ogni colore ha un **ruolo di stato/azione**, mai un vezzo estetico. |
| **Contrasto accessibile** | Le coppie colore/sfondo rispettano la leggibilità WCAG (testi sui colori di stato non in versione "chiara"). |
| **Un solo punto di verità** | I token sono le variabili CSS `:root`; non si usa mai un hex "nudo" nel markup se esiste la variabile. |
| **Dark-mode ready** | I ruoli sono semantici: in futuro basterà ridichiarare `--db-*` nel tema scuro senza toccare i componenti. |

---

## 2. Variabili CSS di riferimento (token)

```css
:root {
  --bs-primary: #0d6efd;   /* blu Bootstrap primary */
  --db-bg:      #f5f6f8;   /* sfondo pagina */
  --db-border:  #dee2e6;   /* bordo card/tabelle */
}
```

**Contratto**: nessuna pagina, scheda o documento va oltre questo set di token per il "tono di fondo". Gli elementi più decorativi/fissi sono elencati nella sez. 4 e vanno **promossi a variabile** prima di essere riusati (regola di igiene per la futura applicazione).

---

## 3. Semantiche Bootstrap - ruoli e uso

| Nome | Hex | Uso tipico (oggi) | Uso previsto nella futura applicazione |
|---|---|---|---|
| **primary** | `#0d6efd` | azioni principali, bordo-sx KPI, icone titolo pagina, link, FAB aiuto, divider attivo, hover tabella | azioni primarie, CTA, accento focus, link, indicatore selezione attiva, bordo-sx card KPI |
| **success** | `#198754` | conferme, badge "Integra/Esportato", qta positive, pallino ok, riga nuova (bordo input), toast ok | conferme operative, stati "ok/attivo", KPI positivi, tost di successo, scheda "pronta" |
| **danger** | `#dc3545` | eliminazione, errori, qta negative, pallino ko, mismatch totale, badge Disabilitato/Errore | azioni distruttive, errori di validazione, KPI sotto soglia, stato "anomalia" |
| **warning** | `#ffc107` | stati da verificare, righe dirty, highlight nuovo ordine, badge "Non visto" | stati "attenzione/da revisionare", elementi da validare, evidenza novità |
| **info** | `#0dcaf0` | badge confermato, badge DUPLICATO, KPI info, codice Integra nelle liste | note informative, duplicati/merge, suggerimenti IA, metadati |
| **secondary** | `#6c757d` | badge righe/contatori, stati neutri, placeholder, colonne attenuate | non-azioni, contatori, testo secondario, stato "in bozza" |
| **dark** | `#212529` | navbar, header tabella, header modal errore, badge ruolo n8n | testi primari scuri, header, contenitori ad alto contrasto |
| **light** | `#f8f9fa` | header tabella sticky alternativa, input readonly/calc | superfici alternate, campi bloccati, sfondi attivi leggeri |

---

## 4. Colori fissi (hex usati direttamente oggi)

> Regola per la futura applicazione: ciascuno di questi viene **promosso a token** (`--db-*`) alla prima occorrenza di riuso.

| Hex | Dove è usato oggi | Token proposto (futuro) |
|---|---|---|
| `#f5f6f8` | sfondo pagina (`body`) | `--db-bg` ✅ già token |
| `#dee2e6` | bordo default, divider split-pane, bordo risultati picker, bordo `pre.det-code`, bordo `.mfx-bottom` | `--db-border` ✅ già token |
| `#0d6efd` | divider in hover/dragging, border input focus, drop-zone drag-over border | `--db-focus` |
| `#e8f0fe` | sfondo drop-zone in drag-over | `--db-dropzone-bg` |
| `#525659` | sfondo pannello PDF (`#pdf-panel`) | `--db-pdf-bg` |
| `#aaa` | testo "PDF non disponibile" (`#no-pdf`) | `--db-pdf-empty` |
| `#fff3cd` | evidenza riga ordine appena ricevuto via SSE (rimossa dopo 5s) | `--db-sse-highlight` |
| `#e9ecef` | bg header tabella righe (`tbl-righe`), thead sticky `.mfx-scroll`, bg step stato | `--db-th-bg` |
| `#adb5bd` | sfondo pallino "sconosciuto" (`pall-unknown`), bordo drop-zone upload, frecce stepper | `--db-muted-dim` |
| `#495057` | colore testo intestazioni tabella (prototipo) | `--db-th-color` |
| `#666` | etichette campo testata ordine (`.lbl`) | `--db-field-label` |
| `#888` | etichetta "Righe" pagina ordine | `--db-section-label` |
| `#f8f9fa` | sfondo input readonly / `pre.det-code` | `--db-readonly-bg` |
| `#ffc107` / `#dc3545` / `#198754` / `#0dcaf0` | varianti bordo-sx `.kpi-card`, varianti `.alert-scadenza` | derivano da semantiche (sez. 3) |
| `#fff5f5` / `#f0fdf4` / `#fffdf0` | sfondi alert scadenze (danger/ok/warn) | `--db-alert-bg-danger` / `-ok` / `-warn` |
| `#cfe2ff` + `#084298` | step "active" | `--db-step-active-bg` / `--db-step-active-fg` |
| `#d1e7dd` + `#0f5132` | step "done" | `--db-step-done-bg` / `--db-step-done-fg` |
| `#e9f0fd` + `#cfe0fb` + `#084298` | `.doc-chip` (chip allegato prototipo) | `--db-chip-bg` / `--db-chip-border` / `--db-chip-fg` |
| `#fbfcfd` | sfondo `.doc-panel` vuoto | `--db-panel-empty` |
| `rgba(255,193,7,.15)` | riga "changed" nel diff audit | `--db-diff-changed` |
| `rgba(220,53,69,.1)` | cella "prima" nel diff audit | `--db-diff-before` |
| `rgba(25,135,84,.1)` | cella "dopo" nel diff audit | `--db-diff-after` |
| `rgba(220,53,69,.25)` | anello focus sul campo totale mismatch | `--db-mismatch-ring` |
| `rgba(0,0,0,.06)` / `rgba(0,0,0,.05)` | ombre KPI / screen-card | `--db-shadow-kpi` / `--db-shadow-card` |
| `rgba(0,0,0,.08)` / `rgba(0,0,0,.25)` | hover screen-card / FAB aiuto | `--db-shadow-hover` / `--db-shadow-fab` |
| `rgba(255,255,255,.)` | `bg-secondary bg-opacity-25` per pista strength bar | `--db-strength-track` |

---

## 5. Semaforo quantità (giacenze, picker articoli)

| Stato | Classe | Colore | Font |
|---|---|---|---|
| negativa | `.qta-neg` | `#dc3545` | weight 600 |
| zero / nulla | `.qta-zero` | `#6c757d` | normale |
| positiva | `.qta-pos` | `#198754` | weight 600 |

**Stessa semantica inline nel picker articoli**: `text-danger fw-bold` (<0), `text-muted` (=0 o null), `text-success fw-bold` (>0). Formato: massimo 3 decimali (`maximumFractionDigits:3`).

**Regola di estensione (futuro)**: il semaforo è il modello per ogni metrica quantitativa della futura app (KPI marketing, punteggi lead, budget). Convenzione fissa:
- **rosso** = sotto soglia / negativo / errore
- **grigio** = neutro / assente / zero
- **verde** = sopra soglia / positivo / ok
- (opzionale) **giallo** = zona di attenzione definita dal contesto

---

## 6. Regole applicative per la futura applicazione

1. **Token-first**: ogni colore usato più di una volta diventa variabile `:root` (Colonna "Token proposto" in sez. 4). Vietato l'hex nudo inline.
2. **Semantico, non cromatico**: si nomina il *ruolo* (es. `--db-success`), mai lo *stato visivo* ("verde"). Così un dark theme o un tema brand rimappano i token senza toccare i componenti.
3. **Contrasto**: testo su colore di stato usato come *sfondo* → sempre versione scura del token su fondo chiaro (es. `#084298` su `#cfe2ff`, `#0f5132` su `#d1e7dd`). Colori "full" (#dc3545, #198754, #0d6efd) solo per testo/icone/bordi, mai come sfondo con testo bianco.
4. **Sfondo pagina**: sempre neutro `--db-bg`. I colori accentuati entrano come bordi-sx, pillole, icone, non come superfici piene (fa eco allo stile KPI card esistente).
5. **Stati temporanei** (es. riga "appena arrivata" `#fff3cd`, evidenza SSE): usare colore + restituzione automatica al neutro (timer), mai persistenza CSS.
6. **Dark-mode**: i token della sez. 4 sono già ruoli → in futuro basta un `[data-theme="dark"]` che ridichiara `--db-*`; i componenti restano invariati.
7. **Semantica per stato IA** (futura app marketing):
   - proposta **da revisionare** → `warning`
   - proposta **approvata** → `success`
   - proposta **scartata/anomalia** → `danger`
   - consiglio/informativa IA → `info`
   - bozza/non iniziata → `secondary`

---

## 7. Albero dei file in cui la palette vive

| File | Ruolo |
|---|---|
| `docs/STILE-GUIDA-DECOBRANDS.md` | Fonte canonica completa (68 sezioni, incl. componenti) |
| `docs/palette-colori-decobrands.md` | **Questo documento**: riferimento rapido palette + regole d'uso |
| `docs/vista-imprenditore.html` | Applicazione concreta (4 diagrammi + KPI) basata su queste semantiche |
| Futuro: `assets/css/tokens.css` | Layering token secondo sez. 4 (non ancora creato: si crea in fase di implementazione webapp) |

---

*Documento da tenere allineato alla STILE-GUIDA ogni volta che la palette cambia (impatto: webapp, prototipo, documenti di vendita).*
