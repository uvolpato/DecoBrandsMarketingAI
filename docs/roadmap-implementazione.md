# Roadmap di Implementazione — Sistema Marketing IA Decobrands

**Riferimento:** `docs/specifica-sistema-marketing-ai.md` (v1.2)
**Scopo:** guida operativa step-by-step per **implementazione, verifica e test** di ogni fase.
**Versione:** 1.1 — 12 settembre 2026

---

> ## VINCOLO NON NEGOZIABILE — SOLA LETTURA SUL GESTIONALE
>
> **Il database di Integra è SOLO LETTURA. MAI, e ripeto MAI, alterare i dati
> sul DB del gestionale.**
>
> Nessun `INSERT`/`UPDATE`/`DELETE`, nessun `CREATE`/`ALTER`/`DROP`, nessun
> trigger, grant o modifica di configurazione sul PostgreSQL di Integra.
> L'utenza usata (es. `decobrands_ro`) ha il **solo permesso `SELECT`** su tabelle
> e viste concordate; in alternativa si legge da una **replica read-only**.
>
> **Obbligo operativo per ogni step di questa roadmap:**
> 1. La connessione deve essere **verificata read-only** prima di ogni uso (test di
>    scrittura che deve fallire) — vedi Step 0.1.2, Step 1.2.2 e Fase 1.
> 2. Nessun test, fixture, seed o strumento di sviluppo deve puntare **scritture**
>    verso l'istanza/l'utenza di Integra (né produzione, né staging, né local).
> 3. Qualunque agente IA o sviluppatore che violi la regola è estromesso dal flusso.
> 4. Ogni gate di fase include il controllo "nessuna scrittura tentata sul gestionale".

---

## Indice

- [Come usare questo documento](#come-usare-questo-documento)
- [Metodologia di test trasversale](#metodologia-di-test-trasversale)
- [Fase 0 — Discovery Integra (PostgreSQL)](#fase-0--discovery-integra-postgresql)
- [Fase 1 — Fondamenta e dati](#fase-1--fondamenta-e-dati)
- [Fase 2 — Arricchimento semantico](#fase-2--arricchimento-semantico)
- [Fase 3 — RAG / Vector Store](#fase-3--rag--vector-store)
- [Fase 4 — Customer Intelligence](#fase-4--customer-intelligence)
- [Fase 5 — Generazione comunicazioni (MVP)](#fase-5--generazione-comunicazioni-mvp)
- [Fase 6 — WhatsApp Business API](#fase-6--whatsapp-business-api)
- [Fase 7 — Prospect & Lookalike](#fase-7--prospect--lookalike)
- [Fase 8 — Ufficio Marketing IA](#fase-8--ufficio-marketing-ia)
- [Fase 9 — Ottimizzazione, scala, governance](#fase-9--ottimizzazione-scala-governance)
- [Matrice delle dipendenze](#matrice-delle-dipendenze)

---

## Come usare questo documento

Ogni fase è divisa in **step**; ogni step in **sub-step** con quattro blocchi:

- **Implementazione** — cosa costruire e come.
- **Verifica (DoD)** — checklist accettazione: si passa allo step successivo solo quando tutto è spuntato.
- **Test** — come provare (unit, integration, E2E, contratti, eval IA).
- **Deliverable** — artefatti prodotti.

Tra una fase e l'altra il **gate di fase** riassume i criteri per procedere. Livelli di autonomia IA (L1/L2/L3) come da specifica §2.3.

---

## Metodologia di test trasversale

Norme applicate a tutti gli step.

### T1 — Piramide dei test

| Livello | Contenuto | Strumento | Criterio |
|---|---|---|---|
| Unit | logica pura: calcoli, trasformazioni, normalizzazioni, validazioni | Vitest/Jest | coverage ≥ 80% su moduli core |
| Integration | DB (ttx), code, Redis, connettore `pg` Integra (contenuto) | istanza PostgreSQL+Redis locale Windows dedicata ai test + seed | verdi su CI |
| E2E | flussi UI: scheda cliente, coda approvazione, invio | Playwright | flussi critici verdi |
| Contract/API | contratti HTTP (esiti, status, payload) | supertest + OpenAPI schema | compatibilità endpoint |

> **Nota ambiente**: nessun uso di Docker. Test, staging e produzione girano su **Windows nativo** (PostgreSQL, Redis e runtime Node come servizi Windows). Gli ambienti di test usano istanze locali dedicate con seed, mai l'istanza di Integra e mai il DB applicativo di produzione.

### T2 — Test deterministici vs probabilistici (IA)

- **Deterministici**: funzioni pure, ETL, RFM, retrieval strutturato → test normali con input/output attesi.
- **Probabilistici (LLM)**: generazione/testuale → **golden set** (coppie input→atteso), **eval harness** con metriche oggettive e **LLM judge** per consistenza; esecuzione su CI settimanale e in pre-release.
- **Snapshot/regression**: hash dei prompt e dei contesti → se cambia il prompt, cambia il golden set consapevolmente.

### T3 — Ambienti

| Ambiente | Uso | Dati |
|---|---|---|
| `local` | sviluppo step-by-step | seed sintetico anonimizzato (fixture) |
| `staging` | integrazione, test pre-pilota | sottoinsieme reale anonimizzato + golden |
| `prod` | pilota reale | dati reali, funzionalità in scudo (feature flag) |

### T4 — Metriche IA di riferimento

| Metrica | Soglia di accettazione |
|---|---|
| Retrieval Recall@k | ≥ 0.85 sul golden set per tipo entità |
| Grounding (0 allucinazioni fattuali) | = 0 sugli invii approvati |
| Valutazione reviewer umano | ≥ 90% bozze "sendable" senza modifiche |
| Insight precision (falsi positivi) | ≥ 95% (falsi positivi < 5%) |
| **Fedeltà prodotto nei visual stand** (similarità visiva cutout vs output) | ≥ 0.90 |
| **Tasso visual approvati senza rigenerazione** | ≥ 80% |
| **Qualità cutout prodotto** (ispezione visiva campione) | ≥ 95% |

### T5 — Vincolo non negoziabile: sola lettura su Integra

Ogni attività di questa roadmap — sviluppo, test, staging, produzione, agenti IA — rispetta la regola **MAI scrivere sul DB del gestionale**. Verifiche obbligatorie:

- La connessione a Integra usa **solo credenziali con permesso `SELECT`** (o una replica read-only); nessuna credenziale scrivibile di Integra esiste, in nessun ambiente.
- Ogni job/worker/agente che legge da Integra esegue alla partenza un **test di scrittura negata** (una transazione `READ ONLY` con un `UPDATE` che deve fallire). Se non fallisce → processo bloccato + alert.
- I seed/fixture/test scrivono **esclusivamente** nei database applicativi di sviluppo, **mai** verso l'istanza/l'utenza di Integra.
- I log di esecuzione registrano l'esito del check read-only per ogni processo.
- Violazione della regola = incidente di sicurezza (severity critica) e gate di fase **bloccato**.

---

## Fase 0 — Discovery Integra (PostgreSQL)

Obiettivo: accesso in sicurezza al DB PostgreSQL di Integra e mappa completa dei dati.

### Step 0.1 — Contatto vendor e concessioni

Sub-step 0.1.1 — Definire i requisiti di accesso
- **Implementazione**: documento interno di richiesta: nodo/replica, istanza, utenza `decobrands_ro` avente sola `SELECT`, replicazione logica (`pgoutput`), orari di mantenimento, eventuale replica streaming.
- **Verifica (DoD)**: requisiti approvati da direzione IT; RACI su chi concede.
- **Test**: n/a.
- **Deliverable**: `docs/integra-access-requirements.md`.

Sub-step 0.1.2 — Ottenere accesso e credenziali
- **Implementazione**: rilascio utenza RO e vista/permessi sulle tabelle concordate; eventuale setup replica logica verso istanza dedicata. **Solo permessi `SELECT`; nessuna credenziale scrivibile generata o conservata.**
- **Verifica (DoD)**: connettività verificata; conferma scritta che sono abilitate solo le operazioni di lettura; nessuna credenziale con permessi di scrittura nell'inventario.
- **Test**: query di prova `SELECT 1`; controllo della modalità read-only; tentativo `UPDATE` (deve fallire: transazione READ ONLY) su una tabella campione concordata; verifica che il login `decobrands_ro` non possa eseguire `CREATE`/`DROP`/`TRUNCATE`.
- **Deliverable**: credenziali/referenze gestite in secret manager.

### Step 0.2 — Reverse-engineering dello schema

Sub-step 0.2.1 — Inventario tabelle e viste
- **Implementazione**: script di discovery (`information_schema.columns`, `pg_stat_user_tables`, `pg_relation_size`) che produce elenco tabelle con stima righe e volume.
- **Verifica (DoD)**: elenco completo con volume per entità (clienti, ordini, righe, prodotti, fornitori, listini), copertura delle combinazioni con i casi d'uso.
- **Test**: confronto conteggi tra run successive; identifca tabelle "spazzatura" (tabelle di lavoro temporanee del gestionale).
- **Deliverable**: `docs/integra-schema-inventory.json`.

Sub-step 0.2.2 — Mapping tabella → entità del dominio
- **Implementazione**: per ciascuna entità della specifica §5, individuare la/candela tabella sorgente e i campi (con tipi e nullability); annotare API DB (es. date, importi come decimal).
- **Verifica (DoD)**: coverheet completo — ogni campo richiesto di clienti/ordini/righe/prodotti/fornitori/listini ha derivazione sorgente; ogni mapping ha nota di caveat (default, unità).
- **Test**: correzione incrociata conteggi campi con sorgente; periodo di campionamento manuale su 20 record per tabella.
- **Deliverable**: `docs/integra-schema-map.md`.

Sub-step 0.2.3 — Valutazione qualità dati
- **Implementazione**: query di profiling: NULL rate, duplicati (P.IVA, ragione sociale), outlier date/importi, codici prodotto mancanti, email/telefono assenti.
- **Verifica (DoD)**: report qualità con top-10 problemi e soglie; lista clienti "da pulire" pronta.
- **Test**: script di profiling riutilizzabile (job futuro di qualità).
- **Deliverable**: `docs/integra-data-quality.md`.

### Step 0.3 — Progettazione integration/staging

Sub-step 0.3.1 — Disegno viste/staging
- **Implementazione**: schema `staging` sul DB applicativo: tabelle speculari per entità + colonne di processo (`etl_run_id`, `source_updated_at`, `is_dirty`, `checksum`).
- **Verifica (DoD)**: disegno approvato; compatibile con future logical replication o delta.
- **Test**: n/a (progettazione).
- **Deliverable**: migrazioni DDL in `prisma/migrations`.

### Gate 0

- [ ] Connessione RO testata e **read-only verificata** (tentativo di scrittura fallito; nessuna credenziale scrivibile di Integra in circolazione).
- [ ] Test `{SELECT}`/`{UPDATE}`/`{CREATE}` documentati ed eseguiti sul login dedicato.
- [ ] `integra-schema-map.md` revisionato e coerente.
- [ ] Profiling qualità dati disponibile.
- [ ] DDL staging approvato.

---

## Fase 1 — Fondamenta e dati

Obiettivo: scaffold del progetto, infrastruttura e primo ETL funzionante sui dati reali.

### Step 1.1 — Scaffold monorepo

Sub-step 1.1.1 — Inizializzazione workspace
- **Implementazione**: monorepo pnpm (`apps/web` Next.js, `apps/worker` BullMQ, `packages/core` condiviso); tsconfig condiviso; ESLint + Prettier; GH Actions basic.
- **Verifica (DoD)**: `pnpm install` pulito, `pnpm build` OK, lint/typecheck verdi.
- **Test**: CI verde sul primo commit.
- **Deliverable**: repo iniziale.

Sub-step 1.1.2 — Infrastruttura locale (Windows nativo)
- **Implementazione**: installazione/registrazione servizi Windows: PostgreSQL 16 (+pgvector) e Redis come servizi dedicati; `.env` template; script PowerShell `scripts/start-services.ps1`/`status-services.ps1` per avvio/arresto/status; runtime Node avviato con NSSM (o Task Scheduler).
- **Verifica (DoD)**: servizi registrati e avviati **senza Docker/container**; healthcheck verdi (PostgreSQL/Redis); l'app Next parte localmente.
- **Test**: smoke test connessioni (pg, redis) da PowerShell/Node.
- **Deliverable**: servizi + env template + script PS1.

### Step 1.2 — Schema applicativo e connessioni

Sub-step 1.2.1 — Migrazioni prisma dello schema §5
- **Implementazione**: modello `schema.prisma` (master data, arricchiti, embeddings, analitico, prospect, comunicazioni, consenso, log, events); enums stato.
- **Verifica (DoD)**: `prisma migrate dev` applica senza errori; schema coerente con documentazione.
- **Test**: test di migrazione (up/down), FK e constraint.
- **Deliverable**: `packages/db`.

Sub-step 1.2.2 — Connettore Integra PostgreSQL
- **Implementazione**: modulo `pg` client pool con config **strictamente read-only** (credenziali a solo `SELECT`, `default_transaction_read_only = on`, `statement_timeout`, pooling limitato); modulo `staging` reader con query parametrizzate mappate da Step 0.2.2. **Il connettore espone solo operazioni di lettura: nessun metodo scrive.**
- **Verifica (DoD)**: connette al vero DB (staging) e legge entità; nessuna query full scan su tabelle enorme (use filters); **test di scrittura negata incluso nella libreria di test**.
- **Test**: **security/RO**: tentativo `UPDATE`/`INSERT` via connettore deve fallire; **integration**: su istanza PostgreSQL locale Windows con seed × 10k ordini, latenza read < target; **unit**: parser mapping.
- **Deliverable**: `apps/worker/src/etl`.

### Step 1.3 — ETL master data

Sub-step 1.3.1 — ETL clienti
- **Implementazione**: lettura → normalizzazione (trim, case su P.IVA, dedupe per P.IVA+ragione) → upsert in `customers`; log `etl_runs`; dirty flag per embedding.
- **Verifica (DoD)**: numero clienti caricati = conteggio sorgente (± tolleranza 0); duplicati risolti e documentati.
- **Test**: **unit** trasformazione singoli record; **integration** import intero dataset su istanza PostgreSQL locale Windows con dati finti e confronto conteggi; **fixture** campionamento 20 clienti a mano.
- **Deliverable**: ETL clienti + job.

Sub-step 1.3.2 — ETL prodotti, categorie, fornitori, listini
- **Implementazione**: stessa pipeline di 1.3.1; aiuti gerarchia categorie (parent-child), validità listini, note fornitori.
- **Verifica (DoD)**: coerenza FK (ogni riga ordine → prodotto esistente), listini attivi presenti.
- **Test**: **integration** referential integrity; **unit** mapping categorie.
- **Deliverable**: ETL catalogo.

Sub-step 1.3.3 — ETL ordini e righe (incrementale)
- **Implementazione**: snapshot notturno + delta su `updated_at` (o logical replication se disponibile da Step 0); idempotenza (delete+insert per run su finestra) **solo sulle tabelle applicative, MAI sul DB di Integra**.
- **Verifica (DoD)**: riproduzione esatta (righe ordini = fonte); nessuna duplicazione a run ripetute; nessuna scrittura tentata verso Integra (log read-only check tutti verdi).
- **Test**: run 2 volte → conteggio invariato; **chaos**: interruzione a metà → retry non duplica; **guard**: verifica che non esistano `INSERT`/`UPDATE`/`DELETE` nel codice ETL indirizzati alla connessione Integra.
- **Deliverable**: ETL ordini.

### Step 1.4 — Consenso e privacy

Sub-step 1.4.1 — Modello consenso e gestione opt-out
- **Implementazione**: `consent_log`, `do_not_contact`, import marker consensi dal gestionale se esistenti; API opt-out.
- **Verifica (DoD)**: ogni comunicazione filtro consenso; opt-out applicato < 48h.
- **Test**: **unit** helper filtro; **integration** end-to-end opt-out blocca delivery.
- **Deliverable**: modulo consenso.

### Gate 1

- [ ] CI verde (lint, typecheck, test di base).
- [ ] ETL clienti/ordini/prodotti/listini funzionante con dati staging-attually reali.
- [ ] Qualità dati supervisionato e dedupe accettati.
- [ ] Consenso e opt-out operativi.
- [ ] **Vincolo read-only rispettato**: check di scrittura negata incluso nei test e verde; nessuna credenziale scrivibile di Integra in circolazione; nessun `INSERT`/`UPDATE`/`DELETE` nel codice verso Integra.

---

## Fase 2 — Arricchimento semantico

Obiettivo: aumentare la densità semantica con ricerche web tracciabili.

### Step 2.1 — Servizio ricerca web

Sub-step 2.1.1 — Provider e quota
- **Implementazione**: modulo `packages/web-search` (provider Serper di default, astratto per switch a Brave); rate limiter, cache query (hash md5), quota giornaliera configurabile.
- **Verifica (DoD)**: quota rispettata; cache funziona (stessa query → 1 chiamata); backup provider se quota esaurita.
- **Test**: **unit** cache/rate limit; **integration** chiamata reale con mock della rete per CI.
- **Deliverable**: modulo ricerca.

### Step 2.2 — Pipeline enrichment clienti

Sub-step 2.2.1 — Generazione query e ricerca
- **Implementazione**: costruzione chiavi `"{ragione sociale}" {città}` (+ P.IVA); da coda BullMQ priorità attivi.
- **Verifica (DoD)**: tutti i clienti prioritari hanno ≥ 1 ricerca in giornata.
- **Test**: golden set di query attese.
- **Deliverable**: job enrichment.

Sub-step 2.2.2 — Estrazione LLM strutturata
- **Implementazione**: estrazione JSON per `customer_profiles_web` con prompt (appendice B.2); validation (P.IVA check, coerenza città); confidenza.
- **Verifica (DoD)**: campi JSON validi e coerenti; confidenza soglia; fonti valorizzate.
- **Test**: **golden set** 50 profili noti a mano (precision estrazione); **judge LLM** su campione; controllo assenza invenzioni.
- **Deliverable**: profili clienti arricchiti.

Sub-step 2.2.3 — Segnali e notizie
- **Implementazione**: ricerca news "operatore + segnali" (riapertura, ristrutturazione, nuovi marchi), estrazione `segnali`.
- **Verifica (DoD)**: alert costruibili su segnali; dedupe segnali già visti.
- **Test**: golden su eventi campione noti.
- **Deliverable**: segnali.

### Step 2.3 — Pipeline enrichment prodotti e fornitori

Sub-step 2.3.1 — Descrizioni prodotto estese
- **Implementazione**: per top prodotti (row-based): ricerca web codice+descrizione → `product_descriptions_enriched` (materiali, uso, target, keywords); quota limitata.
- **Verifica (DoD)**: copertura ≥ 80% del catalogo a valore (top per fatturato).
- **Test**: golden set descrizioni; convergenza sulle keywords.
- **Deliverable**: catalogo arricchito.

Sub-step 2.3.2 — Profili fornitori
- **Implementazione**: per fornitore top: ricerca → `supplier_profiles_web`.
- **Verifica (DoD)**: profili per fornitori con fatturato; fonte tracciata.
- **Test**: gold set; dedupe.
- **Deliverable**: profili fornitori.

### Step 2.4 — Store e tracciabilità

Sub-step 2.4.1 — Scrivania dati arricchiti + fonti
- **Implementazione**: salvataggio `*_profiles_web` + `web_sources`; versioning; coda revisione umana per bassa confidenza.
- **Verifica (DoD)**: ogni record ha fonti; bassa confidenza in coda; retro-compatibilità versione.
- **Test**: affidabilità riconciliazione fonti (URL coerenti).
- **Deliverable**: dati arricchiti versionati.

### Step 2.5 — Schede arricchite da cataloghi fornitori e immagini prodotto (UC-09)

Obiettivo: costruire le **schede prodotto/fornitore "parlanti"** (UC-09, §6.6) combinando i dati Integra con quelli **non presenti nel gestionale**, estratti da fonti non strutturate sotto presidio umano.

Sub-step 2.5.1 — Parsing cataloghi PDF fornitori
- **Implementazione**: modulo `catalog.ingest` (PDF nativi e scan) → estrazione documentale/Vision: righe, refs, descrizioni, caratteristiche tecniche, foto → `supplier_catalog_items` + `catalog_documents` (candidata versionata, §6.6). Mapping ref → prodotto (§5.1) con confidenza e fonte.
- **Verifica (DoD)**: per i fornitori prioritari: articoli catalogati con ref mappabile; assenza di invenzioni su campione (flag manuale).
- **Test**: **golden set** catalogo noto; **eval** fedeltà descrizioni; **unit** parsing refs.
- **Deliverable**: catalogo fornitori arricchito.

Sub-step 2.5.2 — Estrazione attributi da immagini prodotto
- **Implementazione**: pipeline Vision (segmentazione/cutout + riconoscimento attributi visibili: materiali, finiture, misure, colori) → proposte in `product_extractions` (candidata con confidenza, §6.6 + §11.4).
- **Verifica (DoD)**: ≥ 80% top-SKU a valore con attributi estratti validi; nessun attributo inventato (si estra ggono solo ciò che è visibile).
- **Test**: **eval** fedeltà visiva cutout (≥ 0.90); **golden set** attributi attesi; **judge** su campione.
- **Deliverable**: schede prodotto arricchite.

Sub-step 2.5.3 — Coda di revisione e presidio umano (UC-10)
- **Implementazione**: ogni estrazione entra come **candidata** (§6.7) con confidenza, fonte, modello/versione; l'operatore dati (stagista, §2.1) valida, corregge o scarta via dashboard §11.4; solo l'approvato entra nella scheda versionata.
- **Verifica (DoD)**: coda revisione funzionante; workflow approva/corregge/scarta tracciato; nessun dato IA in produzione senza validazione.
- **Test**: **E2E** flusso candidata → coda → revisione → approvato (dashboard stagista); **unit** transizioni stato.
- **Deliverable**: workflow operatore dati (MVVP UC-10).

### Gate 2

- [ ] ≥ 80% clienti prioritari arricchiti.
- [ ] Enrichment prodotti/fornitori con copertura a valore ≥ soglia.
- [ ] **Schede arricchite (UC-09)**: catalogo fornitori mappato e attributi prodotto estratti validati per top-SKU.
- [ ] **Workflow operatore dati (UC-10)**: dashboard + coda revisione operativa; estrazioni IA approvate solo via presidio umano.
- [ ] Quote web search rispettate e tracciate.

---

## Fase 3 — RAG / Vector Store

Obiettivo: vettorizzare profili e prodotti, retrieval ibrido e qualità misurabile.

### Step 3.1 — Documenti sintetici

Sub-step 3.1.1 — Costruzione profilo cliente sintetico
- **Implementazione**: funzione `buildCustomerDocument(customerId)` → testo strutturato (anagrafica normalizzata+arricchita+RFM provvisorio+affinità+stato). Campi nel formato §7.2.
- **Verifica (DoD)**: documento completo per cliente attivo; nessun dato esterno non tracciato.
- **Test**: **unit** su fixture (deterministico); coerenza campi.
- **Deliverable**: profili sintetici generati (job).

Sub-step 3.1.2 — Documenti prodotto/categoria
- **Implementazione**: stesso pattern per prodotto e categoria (descrizione estesa + performance).
- **Verifica (DoD)**: copertura catalogo.
- **Test**: **unit** fixture.
- **Deliverable**: documenti prodotto.

### Step 3.2 — Embedding pipeline

Sub-step 3.2.1 — Servizio embedding con cache e batch
- **Implementazione**: servizio embedding (`text-embedding-3-small`) con cache, batch, retry e budget; dimension model in tabella.
- **Verifica (DoD)**: batch funzionante; costo contenuto (cache che non richieste).
- **Test**: **integration** embedding su 100 doc, no duplicati vettori.
- **Deliverable**: job embedding.

Sub-step 3.2.2 — Indicizzazione pgvector
- **Implementazione**: scrittura `embeddings_customer_profile` / `embeddings_product`; indice HNSW (`vector_cosine_ops`) o IVFFlat per volume.
- **Verifica (DoD)**: query nearest neighbor veloci; indice correttamente utilizzato (`EXPLAIN`).
- **Test**: perf su dataset 10k vettori (latenza < 50ms p95); correttezza top-k su fixture note.
- **Deliverable**: collezioni vettori.

### Step 3.3 — Retrieval ibrido

Sub-step 3.3.1 — Service retrieval combinato
- **Implementazione**: vector + full-text (tsvector) + filtri strutturati, fusione RRF (pesi configurabili); soglie di confidenza.
- **Verifica (DoD)**: API `search` ritorna candidati con score e contesto.
- **Test**: **eval harness** su golden set: Recall@k ≥ 0.85 per entità; regression neuronali su pesi.
- **Deliverable**: servizio retrieval.

Sub-step 3.3.2 — Profilo per RAG comunicazioni
- **Implementazione**: preparazione contesto (top-k profilo+prodotti+storico+listino) per generatore (Fase 5).
- **Verifica (DoD)**: contesto limitato in tokens, ordine ottimale (profilo → prodotti → storico).
- **Test**: **perf** budget tokens per tipo canale.
- **Deliverable**: context assembler.

### Gate 3

- [ ] Recall@k ≥ 0.85 su golden set.
- [ ] Latenza retrieval sotto soglia.
- [ ] Embedding e documenti versionati; refresh automatico su `dirty`.

---

## Fase 4 — Customer Intelligence

Obiettivo: insight commerciali puntuali, cliente per cliente.

### Step 4.1 — RFM e segmentazione

Sub-step 4.1.1 — Calcolo RFM
- **Implementazione**: job `rfm.compute`: Recency (mesi da ultimo ordine), Frequency (nr ordini/anno), Monetary (da fatturato/media); bucket 1-5; cluster (Champions, Loyal, At-Risk, Churn, New).
- **Verifica (DoD)**: distribuzione sensata (nessun cluster vuoto o > 90%); stabilità run-to-run.
- **Test**: **unit** con scenario noti (es. cliente inattivo → At-Risk); **integration** riapertura dopo nuovo ordine → aggiornato.
- **Deliverable**: `rfm_scores`.

Sub-step 4.1.2 — Segmenti operativi
- **Implementazione**: builder segmenti (UI query) dietro modello `segments` (regole AND/OR ~ campi RFM,categoria,area,stato) → membership calcolata a run.
- **Verifica (DoD)**: segmento "inattivi 6m" restituirà coerentemente i clienti attesi.
- **Test**: **unit** regole; **integration** membership su fixture.
- **Deliverable**: moduli segmento.

### Step 4.2 — Affinità prodotto

Sub-step 4.2.1 — Matrice affinità
- **Implementazione**: co-occorrenza ordini (prodotto-categoria → cliente), confidenze (supporto/confidenza), top-N per cliente e per segmento.
- **Verifica (DoD)**: le raccomandazioni applicate a un cliente hanno fondamento (es. stessa "categoria" acquistata o lookalike).
- **Test**: **unit** con dataset sintetico con pattern noti (es. chi acquista X acquista anche Y).
- **Deliverable**: `product_affinity`.

### Step 4.3 — Insight generator

Sub-step 4.3.1 — Rilevazione trend e anomalie
- **Implementazione**: analisi per categoria (variazioni %, soglie), anomalie (calo/ripresa, cambio mix), benchmark per segmento; output strutturato `customer_insights` + `grounded_facts`.
- **Verifica (DoD)**: insight consistenti coi dati; nessun insight fantasma (verifica su dataset con trend noti).
- **Test**: **golden set** di 30 casi con trend attesi manualmente; métrica precision ≥ 95%.
- **Deliverable**: insight.

Sub-step 4.3.2 — Insight testuali (LLM)
- **Implementazione**: prompt (appendice B.1) per testo 1-3 insight per cliente, con punti dati citati.
- **Verifica (DoD)**: testi coerenti, nessun tecnicismo inventato, tono consulenziale.
- **Test**: **judge LLM** su campione settimanle; human evl.
- **Deliverable**: insight testuali.

### Step 4.4 — Priorità e daily digest

Sub-step 4.4.1 — Score di priorità vendita
- **Implementazione**: `priority = f(RFM, potenziale lookalike, margine, trend, segnali)`; soglie.
- **Verifica (DoD)**: ranking sensato (top clienti contattabili più probabili).
- **Test**: benchmark ranking su dati storici.
- **Deliverable**: priorità.

Sub-step 4.4.2 — Daily digest UI
- **Implementazione**: pagina "5 clienti da contattare oggi" con motivazione; widget scheda 360°.
- **Verifica (DoD)**: flusso venditore: digest → scheda → insight → campagna.
- **Test**: **E2E** Playwright sul flusso.
- **Deliverable**: UI digest.

### Gate 4

- [ ] RFM/segmenti validati su dati reali.
- [ ] Insight precision ≥ 95% su golden set.
- [ ] Priorità e digest usabili, E2E verdi.

---

## Fase 5 — Generazione comunicazioni (MVP)

Obiettivo: primo invio reale con approvazione umana. **Priorità del progetto.**

### Step 5.1 — Template e brand voice

Sub-step 5.1.1 — Modello template e guideline
- **Implementazione**: `communication_templates` (canale, tipo, block oggetto/preheader/corpo/CTA, variabili ammesse, limits); admin UI CRUD.
- **Verifica (DoD)**: template WhatsApp conformi ai requisiti Meta (vedi Fase 6).
- **Test**: **unit** render variabili; sanitizzazione input.
- **Deliverable**: CRUD template.

Sub-step 5.1.2 — Configuratore layout grafici e blocklist (UC-11/UC-12)
- **Implementazione**: `template_layouts` (logica: colonne consentite, block divider, brand ligature/palette/typography §10.8) + **blocklist** di oggetti componibili (tabella articoli, scheda singola, scheda con varianti, slot visual §10.10, blocco contatti/CTA/footer), §6.6; drag&drop ad albero; il **grafico può estendere/rimuovere blocchi**: la generazione IA può solo comporre GLI STESSI, mai inventare ex-novo (blocklist); preview rendering e varianti varianti.
- **Verifica (DoD)**: tipo layout per canale funzionante; iniziano varianti tra template diverse; vincolo‑check block per la generazione IA (nessun blocco fuori blocklist).
- **Test**: **unit** vincolo di composizione; **eval** validazione template generato (funzioni di qualità); **E2E** drag&drop persona.
- **Deliverable**: libreria layout + vincoli di generazione.

### Step 5.2 — Generatore con contesto RAG

Sub-step 5.2.1 — Context assembly + chiamate LLM
- **Implementazione**: orchestratore `comms.generate`: contesto (Step 3.3.2) + vincoli (§5.4 spec) + call LLM (modello config), output JSON (oggetto/preheader/corpo/CTA).
- **Verifica (DoD)**: bozze generate per cliente pilota con 1-3 prodotti motivati; prezzi solo da listino.
- **Test**: **golden set** 20 comunicazioni attese (revisionate a mano); **judge LLM** grounding fatti; **unit** guardia vincoli.
- **Deliverable**: generatore.

Sub-step 5.2.2 — Validazione automatica
- **Implementazione**: suite controlli (grounding, no-hallucination judge, duplicati, frequenza, formato/canale, tonality, blocco).
- **Verifica (DoD)**: nulla di non valido passa alla coda; report motivi scarto.
- **Test**: **unit** ogni regola con casi positivi/negativi; **integration** recap.
- **Deliverable**: validator.

### Step 5.3 — Approval queue

Sub-step 5.3.1 — Coda assegnazione
- **Implementazione**: `approval_queue` (bozza → assign → pending); notifica venditore (in-app/email); azioni Approva/Modifica/Scarta con nota; audit log.
- **Verifica (DoD)**: venditore vede solo i propri; traccia azioni.
- **Test**: **integration** transizioni stato; **E2E** UI coda.
- **Deliverable**: coda approvazione.

### Step 5.4 — Delivery email

Sub-step 5.4.1 — Integrazione provider email
- **Implementazione**: modulo `delivery.email` (Resend/SES): template HTML responsive, SPF/DKIM/DMARC, domini, track open/click; code idempotente di invio.
- **Verifica (DoD)**: invio in sandbox; deliverability metric in dashboard provider; webhook esiti→`delivery_log`.
- **Test**: **integration** send con mock provider / sandbox; test bounce config; **E2E** flusso approva→invia.
- **Deliverable**: delivery email.

Sub-step 5.4.2 — Finestre di invio e dedupe
- **Implementazione**: scheduler con finestre orarie, dedupe (tipo+cliente+finestra), frequenza max per canale.
- **Verifica (DoD)**: nessun doppio invio nella stessa finestra; rispetto rate limit.
- **Test**: **unit** logica finestra; **integration** con clock simulato.
- **Deliverable**: scheduler.

### Step 5.5 — Campagne

Sub-step 5.5.1 — Creazione e scheduling campagne
- **Implementazione**: UI campagna (segmento = Step 4.1.2, template, trigger/schedule, obiettivo, priorità); job `campaign.trigger`.
- **Verifica (DoD)**: crea e genera bozze per i membri del segmento; dashboard stato.
- **Test**: **E2E** creazione→generazione→approvazione→invio pilot.
- **Deliverable**: moduli campagna.

### Step 5.6 — Pilota reale

Sub-step 5.6.1 — Segmento pilota
- **Implementazione**: 50-100 clienti "whitelist" consenzienti; metric base; obiettivo di misura.
- **Verifica (DoD)**: invio completato; esiti catturati.
- **Test**: verifica deliverability, open/click; revisione campione da venditori (confidentiality).
- **Deliverable**: report pilota email.

### Step 5.7 — Visual generativo dello stand (pilota)

Obiettivo: generare l'immagine dello stand del cliente con i prodotti proposti (specifica §10.10), integrata nella comunicazione.

Sub-step 5.7.1 — Catalogo cutout prodotto
- **Implementazione**: pipeline di **segmentazione dei prodotti** dalle immagini ufficiali (tool dedicato o servizio API) → PNG con sfondo trasparente in `product_images` (cutout), con fingerprint ed esito validazione umana.
- **Verifica (DoD)**: cutout validi per il top-catalogo a valore (≥ 80% SKU prioritari); cutout con bordi puliti, senza ombre residue.
- **Test**: **eval** ispezione visiva su campione (qualità cutout ≥ 95% ok); **unit** fingerprint/checksum e dedupe.
- **Deliverable**: catalogo cutout.

Sub-step 5.7.2 — Repository foto stand
- **Implementazione**: import/upload foto stand vuoto dei clienti in `stand_photos` + consenso all'uso (obbligatorio); UI di gestione (associa cliente/espositore, stato).
- **Verifica (DoD)**: ogni foto ha cliente, tipo espositore, consenso e stato valido; nessuna generazione senza consenso.
- **Test**: **E2E** upload→approvazione consenso; **security** blocco generazione senza consenso.
- **Deliverable**: repository stand.

Sub-step 5.7.3 — Servizio di generazione immagini
- **Implementazione**: modulo `image-gen` (provider immagine via API, es. Gemini Image / gpt-image-1) con: pre-flight (cutout/consenso/quota), layout planning (placement), chiamata con prompt strutturato (B.4 spec), retry/budget, upscaling opzionale, salvataggio in `visual_assets` + log `image_generations`.
- **Verifica (DoD)**: generazione end-to-end ≤ 3 min; log completo (input, modello, prompt, costo, esito); budget rispettato.
- **Test**: **unit** pre-flight e routing; **integration** con provider sandbox/mock (CI) + 1 chiamata reale in staging; **perf** tempo/costo.
- **Deliverable**: servizio generazione.

Sub-step 5.7.4 — Validazione fedeltà prodotto
- **Implementazione**: check automatico: similarità visiva cutout vs prodotto nell'output (embedding visivo / CLIP, soglia ≥ 0.90), rilevamento di prodotti alterati/estranei, punteggio qualità; rigenerazione max N (es. 3) se sotto soglia, poi scarta.
- **Verifica (DoD)**: nessun prodotto alterato passa la validazione; fallback senza visual disponibile.
- **Test**: **golden set** di 30 output attesi (ok/bad) con soglie calibrate; **judge immagine** (B.5 spec) su campione.
- **Deliverable**: validator visual.

Sub-step 5.7.5 — Integrazione nelle comunicazioni
- **Implementazione**: la bozza UC-03 può includere il visual (testo+immagine insieme); anteprima e rigenerazione/varianti in coda approvazione; dedupe/frequenza anche per immagini; template email/WhatsApp con allegato immagine.
- **Verifica (DoD)**: flusso "proposta → genera visual → revisiono → approvo → invio con immagine" funzionante; fallback senza visual.
- **Test**: **E2E** flusso completo con immagine; **unit** dedupe immagine/proposta.
- **Deliverable**: comunicazioni con visual.

Sub-step 5.7.6 — Pilota visual (opzionale, post-primo invio)
- **Implementazione**: su una parte del segmento pilota (A/B): proposte con visual vs senza; misura risposta/ordini.
- **Verifica (DoD)**: delta conversione misurato; decisione go/no-go sull'uso sistematico.
- **Test**: analisi statistica campione.
- **Deliverable**: report A/B visual.

### Gate 5

- [ ] Generatore produce bozze valide su golden set.
- [ ] Validatore blocca non-conformi (0 falle in test).
- [ ] Coda approvazione utilizzata; primo invio pilota ok.
- [ ] Metriche di esito registrate.
- [ ] **Visual stand (se attivato)**: cutout validati, consensi foto presenti, generazione ≤ 3 min, fedeltà prodotto ≥ 0.90, flusso con fallback testato.

---

## Fase 6 — WhatsApp Business API

Obiettivo: flussi WhatsApp su whitelist con dedupe multicanale.

### Step 6.1 — Setup Meta Business

Sub-step 6.1.1 — Account e numero
- **Implementazione**: business verification, numero dedicato, display name approvato, webhook; credenziali in secret.
- **Verifica (DoD)**: numero attivo; webhook riceve stati di test.
- **Test**: verifica configurazione sandbox Meta.
- **Deliverable**: setup Meta.

Sub-step 6.1.2 — Template WhatsApp
- **Implementazione**: creazione template per tipologie (proposta novità, riordino, avvicinamento) con variabili `{{1}}..{{n}}`; approvazione Meta.
- **Verifica (DoD)**: template approvati; mapping tipologia→template id.
- **Test**: preview, messaggi di prova su numero interno.
- **Deliverable**: template.

### Step 6.2 — Delivery e webhook

Sub-step 6.2.1 — Modulo invio WhatsApp
- **Implementazione**: `delivery.whatsapp` via Meta Cloud API; retry, idempotenza; enforcement limite 24h window/template.
- **Verifica (DoD)**: invio template con variabili corrette; stati `delivered/read` catturati.
- **Test**: **integration** mock Meta; **E2E** invio test a numero interno.
- **Deliverable**: delivery WhatsApp.

Sub-step 6.2.2 — Webhook esiti
- **Implementazione**: endpoint webhook con verifica firma, aggiornamento `delivery_log`.
- **Verifica (DoD)**: esiti coerenti; niente loop.
- **Test**: **unit** parsing payload; replay protegge.
- **Deliverable**: webhook.

### Step 6.3 — Dedupe multicanale

Sub-step 6.3.1 — Coerenza email/WhatsApp
- **Implementazione**: un cliente non riceve la stessa proposta su 2 canali nella stessa finestra; preferenze canale cliente.
- **Verifica (DoD)**: nessun doppio invio cross-canale nella finestra.
- **Test**: **integration** scenario cross-canale.
- **Deliverable**: dedupe.

### Step 6.4 — Pilota WhatsApp

Sub-step 6.4.1 — Segmento whitelist e misura
- **Implementazione**: 30-60 clienti; metric (read, reply).
- **Verifica (DoD)**: invio completato ed esiti raccolti.
- **Test**: revisione campione, benchmark reply.
- **Deliverable**: report pilota WhatsApp.

### Gate 6

- [ ] Template approvati e numeri attivi.
- [ ] Invio e webhook funzionanti.
- [ ] Dedupe multi-canale verificato.
- [ ] Pilota WhatsApp misurato.

---

## Fase 7 — Prospect & Lookalike

Obiettivo: pipeline prospect con confronto lookalike e affinità prodotti.

### Step 7.1 — Scoperta prospect

Sub-step 7.1.1 — Seed list
- **Implementazione**: import seed list (fiere, contatti raccolti) → `prospects` (pulizia, dedupe coi clienti esistenti).
- **Verifica (DoD)**: nessun prospect già cliente; dedupe email/P.IVA.
- **Test**: **unit** dedupe.
- **Deliverable**: seed.

Sub-step 7.1.2 — Web discovery
- **Implementazione**: ricerche strutturate per settore/area; estrazione operatore (P.IVA, dati chiave) con confidenza; dedupe con clienti e prospect esistenti.
- **Verifica (DoD)**: top-N prospect qualificati con dati; confidenza soglia.
- **Test**: golden set di outlet noti; precision discovery ≥ 80%.
- **Deliverable**: discovery job.

### Step 7.2 — Lookalike matching

Sub-step 7.2.1 — Similarità semantica
- **Implementazione**: `embedding` profilo prospect → cosine vs `embeddings_customer_profile` top-k > soglia; aggregazione attributi dai lookalike.
- **Verifica (DoD)**: per prospect campione i lookalike sono pertinenti (revisione umana).
- **Test**: eval su golden (prospect→lookalike attesi noti); Recall@k.
- **Deliverable**: `prospect_lookalikes`.

Sub-step 7.2.2 — Affinità prodotti prospect
- **Implementazione**: trasferimento matrice dai lookalike (peso = similarità) → `prospect_product_affinity`; filtro disponibilità+margine.
- **Verifica (DoD)**: prodotti affini coerenti col settore del prospect.
- **Test**: eval manuale campione.
- **Deliverable**: affinità prospect.

### Step 7.3 — Score e pipeline

Sub-step 7.3.1 — Score prospect
- **Implementazione**: `score = f(fit, dim, aff, recenza)`, soglie; dashboard pipeline.
- **Verifica (DoD)**: ranking sensato su campione noto.
- **Test**: benchmark storico (clienti attuali come "finti prospect").
- **Deliverable**: score.

Sub-step 7.3.2 — Prima comunicazione prospect
- **Implementazione**: template avvicinamento con prodotti affini; validazione; coda approvazione.
- **Verifica (DoD)**: bozze coerenti e senza invenzioni.
- **Test**: golden set; judge LLM.
- **Deliverable**: prima proposta prospect.

### Gate 7

- [ ] Precision discovery ≥ 80%.
- [ ] Lookalike validati su campione.
- [ ] Affinità prodotti prospect con prodotti reali.
- [ ] Score e prima comunicazione operativi.

---

## Fase 8 — Ufficio Marketing IA

Obiettivo: trasformare i moduli in agenti autonomi supervisionati (specifica §2.3).

### Step 8.1 — Architettura agenti

Sub-step 8.1.1 — Runtime agenti e tool calling
- **Implementazione**: orchestratore (ciclo plan→select→generate→validate→schedule→deliver→measure→learn); runtime (LangGraph o orchestratore custom) con tool definiti: customer_intelligence, product_selector, comms_generator, delivery, analytics.
- **Verifica (DoD)**: esecuzione ciclo completo in L1 (proposta → uomo).
- **Test**: **integration** ciclo con mock; observability (tracce decisioni).
- **Deliverable**: orchestratore.

Sub-step 8.1.2 — Strumenti e permessi per agente
- **Implementazione**: mapping ruolo→tool→permessi (Strategist, Product Curator, Copywriter, Validator, Scheduler, Analyst); scope dati per agente.
- **Verifica (DoD)**: nessun agente accede a ciò che non gli spetta.
- **Test**: **security** test RBAC per agente.
- **Deliverable**: permission map.

### Step 8.2 — Product Curator (selezione articoli)

Sub-step 8.2.1 — Motore di selezione
- **Implementazione**: agente seleziona articoli per cliente/prospect da regole (novità, riordino, cross-sell, margine, scorte) + bordi di varietà; motivazioni strutturate.
- **Verifica (DoD)**: selezione conforme a regole commerciali; motivazioni tracciabili; rispetto limite nr prodotti.
- **Test**: **golden set** selezioni note; **eval** pertinenza (reviewer) su campione; regression regole.
- **Deliverable**: selettore prodotti.

Sub-step 8.2.2 — Gradiente di autonomia
- **Implementazione**: per ambito selettor, livello per cliente/segmento (L1: propone; L2: sceglie entro regole e whitelist; L3: sceglie, log all'analisi).
- **Verifica (DoD)**: comportamenti corretti per livello su scenario test.
- **Test**: scenario matrix L1/L2/L3.
- **Deliverable**: autonomia gradiente.

### Step 8.3 — Autonomia comunicativa

Sub-step 8.3.1 — Copia/validator/Scheduler da runners
- **Implementazione**: composizione di Copywriter (Step 5.2), Validator (Step 5.2.2), Scheduler (Step 5.4.2) come agenti; orchestrazione completa.
- **Verifica (DoD)**: ciclo agent attivo, con approvazione opzionale per livello.
- **Test**: **E2E** traccia ciclo; **guardrail** (nessun invio fuori finestra/regole).
- **Deliverable**: cicli agenti.

Sub-step 8.3.2 — Learning loop (Analyst)
- **Implementazione**: agente Analyst legge `delivery_log` e aggiorna: frequenza per segmento, pesi selezione, note template; con soglia di scostamento per alert umano.
- **Verifica (DoD)**: aggiustamenti tracciate e reversibili; alert su degradazione.
- **Test**: simulazione storica (ottimizzazione su dati passati); A/B.
- **Deliverable**: learning loop.

### Step 8.4 — Governance dell'autonomia

Sub-step 8.4.1 — Guardrail e kill-switch
- **Implementazione**: metriche di quality (grounding, reply rate, rate limit), policy di scala (es. automatico da L2 a L3 solo dopo 4 settimane con success rate ≥ soglia); kill-switch per agente/ambito; audit completo.
- **Verifica (DoD)**: kill-switch disattiva senza perdita dati; policy R=da automatizzare.
- **Test**: **ego** test failure-injection (LLM degraded → blocco).
- **Deliverable**: governance.

### Gate 8

- [ ] Ciclo completo L1→L2 operativo.
- [ ] Product Curator con selezioni validate.
- [ ] Guardrail e kill-switch testati.
- [ ] Learning loop con alert.

---

## Fase 9 — Ottimizzazione, scala, governance

Obiettivo: a regime, costi sotto controllo, qualità monitorata, scalabilità.

### Step 9.1 — Performance e costi

Sub-step 9.1.1 — Ottimizzazione LLM
- **Implementazione**: cache smart, modelli di dimensione per task, batch, monitoraggio costo/record; upgrade downgrade modelli.
- **Verifica (DoD)**: costo per comunicazione < budget settimanale.
- **Test**: **perf** benchmark modelli.
- **Deliverable**: cost dashboard.

Sub-step 9.1.2 — Scaling e osservabilità
- **Implementazione**: prometheus/grafana o OTel; alert (code rosse, errori ETL, saturation).
- **Verifica (DoD)**: metriche di salute; SLO definiti.
- **Test**: load test worker.
- **Deliverable**: monitoring.

### Step 9.2 — Apprendimento e ottimizzazione commerciale

Sub-step 9.2.1 — A/B e auto-tuning
- **Implementazione**: esperimenti su oggetto/CTA/tono; auto-tune frequenza.
- **Verifica (DoD)**: rilevamento di vincitori statisticamente validi.
- **Test**: **eval** su dati storici.
- **Deliverable**: ottimizzazione.

### Step 9.3 — Compliance e governance IA

Sub-step 9.3.1 — Audit e revisione continui
- **Implementazione**: auditoria periodica del comportamento agent; review trimestrale KPI di qualità; registri decisioni.
- **Verifica (DoD)**: report trimestrale pubblicato; nessuna deriva.
- **Test**: campionamento manuale continuo.
- **Deliverable**: audit loop.

Sub-step 9.3.2 — Preparazione a nuove competenze
- **Implementazione**: roadmap V2: multi-lingua, integrazioni di più canali, portale auto-servizio cliente.
- **Verifica (DoD)**: backlog V2 prioritizzato.
- **Test**: n/a.
- **Deliverable**: backlog V2.

### Gate 9

- [ ] Costi/uso sotto budget.
- [ ] SLO rispettati.
- [ ] Governance e audit attivi.
- [ ] Backlog V2 definito.

---

## Matrice delle dipendenze

| Step/fase | Dipende da | Fornisce a |
|---|---|---|
| F0 (Discovery) | vendor/credential | F1 (ETL), connettività |
| F1 (Fundamentals) | F0 | F2, F3, F4 |
| F2 (Enrichment) | F1 (dati), provider | F3 (documenti arricchiti) |
| F3 (RAG) | F2 | F4 (insight contesto), F5 (comms) |
| F4 (CI) | F3 | F5 (affinità/priorità), F7 (lookalike) |
| F5 (Comms MVP) | F3, F4 | F6 (canale), F8 (agenti) |
| F5.7 (Visual stand) | F5 (base), catalogo immagini cutout, foto stand | F5 (comunicazioni arricchite), F8 (Visual Designer) |
| F6 (WhatsApp) | F5 | F8 |
| F7 (Prospect) | F3, F4 | F8 (curator) |
| F8 (Agents) | F5, F6, F7 | F9 |
| F9 (Scale) | F8 | regime |

---

*Roadmap v1.0 — revisionare e aggiornare a ogni gate.*