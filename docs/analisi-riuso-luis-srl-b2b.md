# Analisi riuso — "Luis Srl - B2B" come base per il sistema Decobrands Marketing AI

> Scopo: valutare se l'attuale applicazione **Luis Srl B2B** (`C:\Progetti\Luis Srl - B2B` — portale B2B ordinazioni in produzione per Luis S.r.l.) può essere **base di partenza** (o riferimento) per il sistema descritto in `specifica-sistema-marketing-ai.md` (Decobrands).
> Versione: 1.0 — aggiornata alle specifiche marketing v1.2 e roadmap marketing v1.1.

---

## 1. Sintesi esecutiva

**Luis B2B non è una base funzionale, ma è un'eccellente base architetturale.** Lo stack è lo stesso (NestJS/Next.js/Prisma/Postgres+pgvector/Gemini) e l'integrazione Igor/Gemd (Integra) read-only è già fatta, matura e collaudata. Ma i casi d'uso marketing di Decobrands (schede "parlanti", estrazione da immagini e cataloghi PDF, template grafici popolati per cliente, canali WhatsApp/email, presidio umano dello stagista) **non esistono in Luis e vanno costruiti da zero** — da riusare sono i **pattern infrastrutturali**, non le feature.

**Verdetto (in sintesi):** "riuso come riferimento" — clonare/portare in Decobrands i moduli di **integrazione Integra**, **embedding/ricerca**, **AI (Gemini)**, **RBAC**, **coupon/campagne**, **approvazione umana**; costruire ex novo tutto il layer marketing (estrazione cataloghi, cutout, template vincolati, canali, workflow stagista).

---

## 2. Cosa è Luis Srl B2B (fotografia)

- **Tipo**: portale e-commerce **B2B ordinazioni** per rivenditori di **Luis S.r.l.** (grossista fioristi/garden, Via F. Bellafino 28/30, Bergamo). Clienti = rivenditori B2B (1 account per azienda). Target: **ordini**, non marketing.
- **Stack** (`C:\Progetti\Luis Srl - B2B`):
  - Backend **NestJS 11**, Node 24, **Prisma 7** su **PostgreSQL 16** (+ pgvector disponibile), socket.io WS.
  - Frontend **Next.js 16 (App Router)**, React 19, i18n it/en (`next-intl`), CSS nativo OKLch, **senza** shadcn/Tailwind.
  - AI: **Google Gemini** (flash-pro testi / flash-image vision), **embedding** Gemini (768d) o LM Studio locale; ricerca semantica/hybrid su pgvector (o coseno in Node se PG<15).
  - Integrazione **Integra read-only**: FDW + viste `b2b_*`, sync schedulata (articoli/listini 15', giacenze 10', ordini 15') con **swap atomico**; export ordini verso Integra come Excel `.xlsx` + riconciliazione `mvt_vsrif`. **Nessuna scrittura** verso il gestionale via API.
  - WhatsApp Business: **non implementata** (solo link/condividi). Email: **nodemailer SMTP generico**, niente Resend/SES.
  - Infra: Docker-compose in dev, **NSSM services Windows** + Caddy (HTTPS Let's Encrypt) in prod. Niente task scheduler: cron in-process `@nestjs/schedule`.
- **Stato**: molto avanzato (847 commit, module ordini/checkout/coupon/AI/CRM presenti), **non in go-live finale** (blocco "Collaudo/formazione/go-live" ancora aperto).
- **Roba "AI-ansible/generata" presente nella root**: mockup HTML + `.artifact.json` (demo grafiche agent/admin), `roadmap-b2b-luis.md`, `specifiche-b2b-luis.md`, PDF/PPTX, ecc.

### Non è "lo stagista"
Gli attori sono: cliente rivenditore, staff/admin Luis, venditori (ruolo AGENTE, non implementato), controparte switch house **AGOMIR** (software house di Integra). **Non esiste un ruolo "operatore dati / stagista".** Il "Luis" nel nome è l'**azienda**, non lo stagista. (Nei doc Decobrands "Luis" appare come stagista/AGENTE in `prompt` — verificare e non sovrapporre.)

---

## 3. Confronto requisito per requisito (marketing spec ↔ Luis)

| Area (spec Decobrands) | In Luis? | Note |
|---|---|---|
| **Integra read-only + sync** (§4, §12) | ✅ **SÌ — maturo** | FDW, viste `b2b_*`, sync 15', swap atomico, export Excel riconciliato. Miglior riferimento esistente. |
| **Prisma + PG 16 + pgvector** (§5, §12.1) | ✅ **SÌ** | stesso pattern; embedding (Gemini/LM Studio) + coseno in Node. |
| **Gemini / LLM: testi, descrizioni, insight** (§9, §10) | ✅ **SÌ** | API Gemini, prompt template, cache, tracciamento costi (`AiUsage`). |
| **Embedding + ricerca semantica / per immagine** (§6.3, §9) | ✅ **SÌ** | ricerca semantica, ricerca per immagine, cutout implicito, backfill, simili (Lookalike §6.4). |
| **Schede prodotto arricchite** (materiali/finiture da immagine) | ⚠️ **Parziale** | c'è trascrizione immagine AI (descrizioni/attributi) ma **non codificata come "candidata + revisione umana + versionata"** (manca worklist, confidenza, sorgente versionata). |
| **Estrazione da cataloghi PDF fornitori** (§6.6, UC-09) | ❌ **NO** | nessuna tabella fornitore/catalogo PDF; solo codice fornitore richiesto a Integra, non ancora ricevuto. |
| **Workflow operatore dati / stagista** (§2.1, §6.7, §11.4, UC-10) | ❌ **NO** | c'è presidio manuale admin (modifica descrizioni, "Genera tutto" con conferma) ma nessuna **dashboard dedicata + code revisione + approvazione IA→umano versionata**. |
| **Template/layout grafici configurabili** (§5.6, §10.8, UC-11/UC-12) | ❌ **NO** | solo `communication_templates` (testo/variabili) + brand voice; **manca il configuratore grafico/blocchi e la generazione IA vincolata alla blocklist** (UC-11/11b/12). |
| **Visual stand e cutout PNG** (§10.10, UC-07/08) | ❌ **NO** | immagini prodotto/stand presenti (cutout implicito), ma nessun **compositing controllato stand+cutout** né varianti allestimento. |
| **Canale WhatsApp Business Cloud API** (§10.5-10.6) | ❌ **NO** | solo link di condivisione; template WhatsApp Meta assenti. |
| **Canale email (Resend/SES) + template** (§10.4, §10.13) | ⚠️ **Parziale** | nodemailer SMTP generico; niente Resend/SES, niente sistema template email popolati + metriche. |
| **Catalogo PDF / scheda prodotto condivisibile** (§11.2, UC-03) | ✅ **SÌ (parziale)** | pagine pubbliche `/p/[token]` e `/progetti/[token]` esistono (reali, senza prezzi). |
| **Coupon / campagne / QR / segmentazione** (UC-04/06) | ✅ **SÌ — riusabile** | `campaigns`, `campaign_usage`, coupon con QR, target, filtri, stati. |
| **Proposta omnicanale per cliente (UC-03)** | ❌ **NO** | c'è "progetto/offerta" condivisa, ma nessun motore di **comunicazione popolata per cliente** con motivazione e template. |
| **KPI marketing / fedeltà / rischio** (§18, §19) | ❌ **NO** | ci sono dashboard/suggerimenti AI, ma non le tabelle KPI marketing (fedeltà, coveragio, costi IA, rete consegna). |

---

## 4. Cosa riusare da Luis (base "infrastrutturale")

Da portare in Decobrands **come pattern/pattern-code** (con licenza/consenso), NON copiare ciecamente:

1. **Modulo syncing Integra** (FDW + viste `b2b_*` + `syncConfig` + swap atomico) → è il cuore invariato del §4-§6 spec.
2. **Modulo AI/LLM** (`integrazione`/`generation`): orchestrazione Gemini, prompt template, cache `<esperimento+prompt>`, tracciamento costi `AiUsage`, API usage con RBAC.
3. **Modulo embedding/RAG**: `embedding.service` (Gemini 768d), similarità coseno in Node, `backfill-embeddings`, ricerca ibrida + per immagine.
4. **RBAC + guardie** (`users`, `PermissionGroup`, `roles/permissions.guard`) → da estendere con i ruoli Decobrands (operatore dati, stagista, ufficio marketing).
5. **Coupon/campagne + QR** → riusare quasi intatto per UC-04/UC-06.
6. **Pattern approvazione umana** esistente (framework admin "Genera tutto" con conferma, modifica manuale) → **estenderlo** a coda revisione §6.7 + dashboard §11.4.
7. **Frontend base** (theme OKLch terracotta, i18n it/en, layout dashboard/admin, pagine pubbliche token) → riusabile per §11 gestione proposta e pagine pubbliche.
8. **Infra Windows**: NSSM + Caddy + cron in-process → stesso approccio su Decobrands (roadmap Fase 𝑅/1).

## 5. Cosa NON c'è e va costruito "da marketing"

1. **Estrazione dati da immagini prodotto** come pipeline **con candidate versionate + confidenza + sorgente + approvazione** (§6.6, UC-09).
2. **Parsing cataloghi PDF fornitori** → `supplier_catalog_items` + `catalog_documents` (UC-09).
3. **Workflow stagista/operatore dati**: dashboard (§11.4), code revisione (§6.7), KPI presidio, onboarding/RBAC.
4. **Generazione IA di template/layout vincolata alla blocklist** + editing umano (UC-11/11b/12, §10.8, configurabuster §5.6).
5. **Cutout PNG trasparente** + **originalizzazione stand** (compositing controllato, §10.10, UC-07/08).
6. **Canale WhatsApp Business Cloud API** con template Meta.
7. **Canale email professionale** (Resend/SES) con template + metriche.
8. **KPI marketing/fedeltà/costi IA/lookalike/segment se | gestione proposta popolata per cliente e multi-canale** — pur essendo già in Luis l'integrazione Integra ed il catalogo, il "motore" di proposta popolata e la campagna marketing intenzionale vanno aggiunti.

---

## 6. Approccio consigliato — "riuso come riferimento, non come base"

Per evitare di trascinare il peso di un progetto B2B ordinazioni in un sistema che è **legate marketing**:

- **Do**: riusare i pattern-code di Rafael (Integra sync, AI/LLM, embedding/RAG, RBAC, coupon), gli stili di frontend/dashboard, la deployment Windows.
- **Do not**: riusare le entità ordini/checkout/catalogo storefront come se fossero le entità marketing; tenere **separati i due DB/domini** (Decobrands legge Integra, Non legge privy Luis).
- Conseguenza pratica: il passo più rapido è **incollare/portare nel nuovo repo Decobrands solo i moduli indicati al §4**, e costruire il layer marketing (§5) come moduli nuovi (allineati alla roadmap Fase 1-4).

---

## 7. Piano di implementazione (porta a porta, allineato a roadmap marketing Fase 1-8)

> Numerazione per fase = roadmap-implementazione.md.

- **Fase 0 (pre)**: portare `sync Integra` (FDW+viste) e `AI/embedding` da Luis al nuovo repo → base pronta. Verificare `SETUP.md`/`setup-fdw-prod.sql` di Luis per adattare le viste `b2b_*` ai nomi Decobrands.
- **Fase 1 (base dati + IA portata)**: RBAC esteso (operatore dati/stagista), store dati candidati–versionati (`*_extractions`, `communication_templates`), con consensi.
- **Fase 2 (enrichment)**: pipeline estrazione immagini prodotto (Toastetto) + cataloghi PDF fornitori (Section-cutout §6.6 sub-step) → candidate in coda.
- **Fase 3 (RAG)**: pagina-immagine/posizionamento vettoriale reuse da Luis; repository copertura/embedding.
- **Fase 4-5 (generazione]**: motore proposta popolata per cliente (UC-03) + template grafici con blocklist e configurabuster (UC-11/12) riusando coupon/alinea.
- **Fase 6 (canali)**: WhatsApp Business Cloud API + email Resend/SES + catalogo PDF (diviso email/WhatsApp/catalogo/visual stand).
- **Fase 7 (foto/visual)**: cutout + compositing stand (UC-07/08), riusando pagina pubbliche `/p/` token per il modulo condivisibile.
- **Fase 8 (AI/qualità)**: KPI marketing/fedeltà, costi llm, risk, RICERCA/doppio pag. (Annex B) con dashboard stagista come gate finale.

---

## 8. Conclusione

**Sì, può essere una base di partenza — ma solo come "base architetturale/pattern", non come "base funzionale".** La cosa più preziosa che Decobrands erede da Luis è l'integrazione Integra già collaudata (read-only, FDW, sync, export Excel) e lo stack AI (Gemini, embedding, RAG) su Prisma/PG/pgvector. Tutto il cuore del valore marketing (schede parlanti, estrazione da immagini/cataloghi, template grafici vincolati a blocklist, canali WhatsApp/email, workflow stagista con code di revisione) va costruito ex novo, ben disaccoppiato, usando i pattern e le code/revisione umana come presidio.

*Riferimenti: specifica-sistema-marketing-ai.md (v1.2), roadmap-implementazione.md (v1.1), repo `Luis Srl - B2B` (NestJS/Next.js/Prisma/PG16/Gemini, 847 commit).*
