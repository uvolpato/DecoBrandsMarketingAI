# Specifica Tecnica — Sistema Marketing e Vendita Assistita da IA

**Progetto:** Decobrands Marketing Intelligence & Communication AI
**Versione:** 1.2
**Data:** 12 settembre 2026
**Stato:** Bozza per revisione

---

> ## VINCOLO NON NEGOZIABILE — SOLA LETTURA SUL GESTIONALE
>
> **I dati contenuti nel database di Integra vanno esclusivamente LETTI.**
> **MAI, in nessun caso, per nessun motivo, in nessun ambiente:** il sistema
> **NON deve alterare, modificare, aggiornare, inserire, cancellare o toccare
> in alcun modo i dati del DB del gestionale.**
>
> - Nessuna scrittura (`INSERT`/`UPDATE`/`DELETE`), nessun `DDL`, nessun trigger,
>   nessun `GRANT`, nessuna modifica di config sul DB di Integra.
> - L'integrazione opera **solo in lettura**, preferibilmente su **replica** o con
>   utenza dedicata `decobrands_ro` avente il solo permesso `SELECT`.
> - Ogni layer del sistema (connettore, ETL, agenti IA, test) deve **assumere e
>   verificare** la read-only-ness della connessione prima di operare.
> - Questo vincolo vale **anche in ambiente di sviluppo, staging e test, e anche
>   per eventuali utenti/agenti IA**: chiunque o qualunque agente violi questa regola
>   è fuori dall'architettura di sistema.
>
> Vedere §4, §16.2 e le check-list della roadmap (`docs/roadmap-implementazione.md`).

---

## Indice

1. [Contesto e obiettivi](#1-contesto-e-obiettivi)
2. [Attori e casi d'uso](#2-attori-e-casi-duso)
3. [Architettura di sistema](#3-architettura-di-sistema)
4. [Integrazione con il gestionale Integra](#4-integrazione-con-il-gestionale-integra)
5. [Modello dati](#5-modello-dati)
6. [Pipeline di arricchimento semantico](#6-pipeline-di-arricchimento-semantico)
7. [Layer RAG e Vector Store](#7-layer-rag-e-vector-store)
8. [Modulo Customer Intelligence](#8-modulo-customer-intelligence)
9. [Modulo Prospecting](#9-modulo-prospecting)
10. [Modulo Generazione Comunicazioni (priorità MVP)](#10-modulo-generazione-comunicazioni-priorità-mvp)
11. [Frontend](#11-frontend)
12. [Stack tecnologico](#12-stack-tecnologico)
13. [API principali](#13-api-principali)
14. [Workflow ETL e cron](#14-workflow-etl-e-cron)
15. [Roadmap e fasi di sviluppo](#15-roadmap-e-fasi-di-sviluppo)
16. [Sicurezza, privacy e GDPR](#16-sicurezza-privacy-e-gdpr)
17. [Costi e infrastruttura](#17-costi-e-infrastruttura)
18. [KPI e metriche di successo](#18-kpi-e-metriche-di-successo)
19. [Rischi e mitigazioni](#19-rischi-e-mitigazioni)
20. [Appendice A — Esempi di comunicazioni generate](#appendice-a--esempi-di-comunicazioni-generate)
21. [Appendice B — Prompt engineering](#appendice-b--prompt-engineering)

---

## 1. Contesto e obiettivi

### 1.1 Contesto aziendale

Decobrands opera in ambito **B2B**: i clienti sono **rivenditori e showroom** che commercializzano prodotti di decorazione (arredo, complementi, superfici decorative). Il patrimonio informativo aziendale risiede nel gestionale **Integra**, che conserva:

- anagrafiche clienti e fornitori;
- storico ordini e righe d'ordine;
- catalogo prodotti con listini e margini;
- condizioni commerciali e sconti.

L'attività commerciale si basa oggi su un approccio reattivo: il venditore consulta il gestionale, ricostruisce manualmente il quadro del cliente e propone offerte in modo poco strutturato. L'obiettivo è trasformare questo patrimonio dati in **intelligenza di vendita puntuale, cliente per cliente**, supportata da IA.

### 1.2 Obiettivi

1. **Recuperare** tutti i dati rilevanti dal gestionale Integra in modo affidabile e continuo.
2. **Arricchire semanticamente** i dati con ricerche sul web: profilo degli operatori economici (clienti e prospect), descrizioni prodotti, descrizioni fornitori.
3. **Produrre insight di vendita puntuali per ogni cliente**: comportamento d'acquisto, affinità prodotto, stagionalità, potenziale di crescita, segnali d'acquisto.
4. **Cercare prospect** sul mercato e **confrontarli** con clienti già acquisiti simili (lookalike), stimando l'affinità con i prodotti già in catalogo.
5. **Generare comunicazioni personalizzate** (email e WhatsApp Business API) specifiche per cliente o prospect, con workflow di revisione e approvazione.
6. **Mantenere il cliente stimolato** attraverso un flusso continuo di iniziative commerciali pertinenti (nurturing).
7. **Visualizzare la proposta allo stand**: generare immagini fotorealistiche della proposta allestita sull'espositore del cliente (AI generativa: foto dello stand vuoto + immagini dei prodotti selezionati), rendendo le comunicazioni più persuasive e concrete per il B2B.
8. **A tendere (visione)**: far gestire progressivamente tutte le attività comunicative e di selezione degli articoli a **agenti IA strutturati come un ufficio marketing**, con supervisione umana in panier progressivamente ridotta (vedi §2.3 e §15, Fase 9).

### 1.3 Non obiettivi (della versione 1.0)

- **MAI scrivere sul gestionale Integra**: integrazione **esclusivamente read-only** (vedi box "VINCOLO NON NEGOZIABILE" in testa al documento).
- Vendita online diretta B2C.
- Automazione di processi logistici/contabili.
- Delega totale senza supervisione: l'invio e le scelte ad alto impatto restano **humans-in-the-loop** fino a validazione di affidabilità (Fase 9+).

---

## 2. Attori e casi d'uso

### 2.1 Attori

| Attore | Descrizione |
|---|---|
| **Venditore / Agente** | Consulente commerciale che segue i clienti, approva e invia le comunicazioni. |
| **Ufficio marketing** | Gestisce campagne, template, brand voice, segmenti e regole di auto-azione. |
| **Direzione commerciale** | Osserva KPI, potenziale di sviluppo per cliente, stato del canale. |
| **Operatore dati (stagista / back-office)** | Presidio human-in-the-loop sui dati: inserisce, rivede, corregge, validà e controlla le schede (prodotti, fornitori, clienti, foto stand) e gli output di estrazione IA. Lavora su una dashboard dedicata (vedi §11.4) e su code di revisione (§6.7). |
| **Sistema (agenti IA)** | Esegue ETL, arricchimento, analisi, generazione e invio con supervisione. |

### 2.3 Visione a tendere — Ufficio marketing IA multiagente

A tendere (Fase 9), gli agenti IA operano come un **team marketing**, ciascuno con ruolo, obiettivi e strumenti propri, orchestrati da un coordinatore. Il livello di autonomia è graduabile a 3 livelli per ambito:

| Livello | Nome | Descrizione |
|---|---|---|
| **L1** | Assistito | L'agente propone, l'umano decide (MVP). |
| **L2** | Autonomo supervisionato | L'agente agisce entro regole dichiarate; le eccezioni e gli invii promozionali ad alto impatto richiedono approvazione. |
| **L3** | Autonomo | L'agente gestisce l'intero ciclo entro KPI e guardrail; gli umani monitorano e intervengono su alert. |

**Ruoli previsti (architettura di agenti):**

| Agente | Responsabilità | Strumenti | Livello iniziale |
|---|---|---|---|
| **Strategist** | Obiettivi, segmentazione, scelta mix di contatto per cliente, pianificazione timing | Customer Intelligence, profili, KPI | L1 → L2 |
| **Product Curator** | Selezione degli articoli da proporre (novità, riordino, cross-sell) con motivazioni e vincoli di margine/scorte | affinità prodotto, catalogo arricchito, regole commerciali | L1 → L2 |
| **Copywriter** | Generazione bozze personalizzate per canale e tono | RAG, template, brand voice | L1 → L2 |
| **Reviewer / Validator** | Controllo grounding, duplicati, frequenza, tono, conformità | validatori automatici + LLM judge | L1 → L3 |
| **Scheduler** | Timing di invio, finestre, dedupe multicanale, rate limit | delivery service, calendario | L2 → L3 |
| **Analyst / Optimizer** | Misura esiti, calcola conversioni, aggiorna regole e frequenze | delivery log, KPI, learning loop | L2 |

L'orchestrazione segue un ciclo: **plan → select → generate → validate → schedule → deliver → measure → learn**. Ogni decisione è loggata e riconducibile (audit). La selezione degli articoli (Product Curator) è uno dei due domini prioritari di autonomia insieme alla gestione comunicativa: nella fase 9 l'agente propone listini-mix; l'approvazione umana iniziale degrada a supervisione dopo evidence di qualità (es. success rate ≥ soglia su N settimane).

### 2.2 Casi d'uso prioritari

**UC-01 — Scheda cliente 360°.**: Il venditore apre la scheda di un cliente e vede quadro completo: storico acquisti, RFM, categorie preferite, trend, analisi della concorrenza (se ricavabile dal web), suggerimenti commerciali puntuali.

**UC-02 — Insight di vendita per cliente.**: Il sistema produce ogni ciclo commerciale una serie di insight: "Cliente X ha ridotto acquisti su categoria A del 40% negli ultimi 2 trimestri, ma ha aumentato B; suggerito riallineamento su B e proposta sui nuovi articoli della linea B".

**UC-03 — Proposta commerciale mirata.**: Generazione di una comunicazione personalizzata (email o WhatsApp) con prodotti specifici per quel cliente, motivate dai suoi dati e da segnali rilevati (lanci nuovi, scorte, eventi).

**UC-04 — Ricerca prospect e lookalike.**: Il sistema identifica prospect non ancora clienti, li confronta con clienti simili già acquisiti e stima l'affinità con prodotti in catalogo.

**UC-05 — Campagna outbound.**: L'ufficio marketing definisce un segmento (es. clienti inattivi da 6 mesi, categoria acquistata almeno 1 volta), il sistema genera le comunicazioni personalizzate per ogni membro del segmento e le sottopone ad approvazione.

**UC-06 — Nurturing continuo.**: Stato stabile del cliente: il sistema propone periodicamente iniziative (riordino, novità, eventi) mantenendo ritmo e pertinenza.

**UC-07 — Visual della proposta sullo stand.**: Per una proposta commerciale (UC-03), il sistema genera un'immagine dello stand/espositore del cliente **con gli articoli selezionati già allestiti**, partendo dalla foto dello stand vuoto e dalle immagini dei prodotti proposti. Il venditore revisiona, itera (rigenerazione con varianti) e approva prima dell'invio.

**UC-08 — Varianti di allestimento.**: Su richiesta, il sistema produce più varianti di allestimento (disposizione, tema/colori, stagione) per lo stesso cliente, da usare in esperimenti o per presentazioni personalizzate.

**UC-09 — Scheda prodotto/cliente/fornitore completa.**: Il sistema combina per ciascuna entità i dati presenti in Integra con quelli **non presenti nel gestionale**, ricavandoli via estrazione IA da **immagini dei prodotti** (materiali, finiture, misure, colori visibili), da **cataloghi PDF dei fornitori** (descrizioni, refs, caratteristiche tecniche) e dalla ricerca web. Il risultato è una **scheda arricchita e versionata** che rende ogni soggetto "parlante" per la personalizzazione (vedi UC-03) e per il lookalike (vedi UC-04).

**UC-11 — Configuratore template e layout grafici.**: Una figura dedicata (stagista/back-office o ufficio marketing, §2.1) progetta e mantiene — tramite una dashboard dedicata (§11.4) e senza scrivere codice — una libreria di **template e layout riusabili** per ogni tipo di documento (catalogo proposte, email, WhatsApp, scheda articolo). Il template definisce **struttura e grafica** (non i contenuti): blocchi impaginati (intestazione, **tabella degli articoli proposti**, **scheda articolo singola**, **scheda articolo con varianti**, slot per il **visual stand** §10.10, blocco contatti/CTA, footer), regole tipografiche, colori e spaziature in linea col brand (§10.8). Il sistema **popola poi il template con i dati** — cliente per cliente o per segmento — selezionando prodotto, quantità, motivazione e immagine per ogni riga (§6.6, UC-03/UC-07): l'operatore revisiona, rigenera varianti e approva prima dell'invio (§10.19, §11.2).

**UC-11b — Generazione IA del template vincolata alla blocklist.**: Oltre a progettare i template a mano, lo stagista può chiedere all'IA di **proporre un intero template** (o una sua variante), incluso il layout grafico. Vincolo non negoziabile del *template generation* (fondazionale, §2.2): l'IA **può comporre esclusivamente gli oggetti/blocchi messi a disposizione dal configuratore** (§5.6, §10.19) — tabella articoli, scheda singola, scheda con varianti, slot visual, blocchi contatti/CTA/footer — scegliendone ordine, quantità e parametri grafici consentiti, e **non può introdurre blocchi, componenti grafici o struttura di pagina arbitrari non presenti nella libreria**. La proposta IA entra comunque come **candidata versionata** (§6.7): lo stagista la **modifica liberamente** (editing visuale dei blocchi, testi, ordine, struttura, canale), la rigenera con ulteriori vincoli, o la scarta; solo dopo approvazione umana il template diventa riusabile e popolabile (§5.6). Se l'IA non ha blocchi sufficienti per un layout richiesto, **blocca la generazione e segnala la mancanza** (nessuna invenzione di componenti, §19) invece di improvvisare.

**UC-12 — Tabella articoli con colonne a scelta venditore.**: Nell'ambito del template (UC-11), per la **tabella degli articoli proposti** il venditore/ufficio marketing sceglie quali colonne mostrare (ref, descrizione, materiale/finitura, disponibilità, prezzo/margine, motivazione, note, foto cutout) e il loro ordine; la tabella viene poi compilata in automatico dall'IA con i dati della proposta del singolo cliente. Varianti: vista "solo novità" (solo prodotti nuovi con evidenza "novità") e vista "riordino" (solo prodotti già in catalogo con quantità di riordino consigliata).

---

## 3. Architettura di sistema

```
                    ┌──────────────────────────────────────────────┐
                    │                 INTEGRA (gestionale)          │
                    │              DB: PostgreSQL                   │
                    └───────────────┬──────────────────────────────┘
                                    │ SQL (read-only) / replica
                    ┌───────────────▼──────────────────────────────┐
                    │            INTEGRATION DATABASE               │
                    │  Staging tables normalizzate (lettura)        │
                    └───────────────┬──────────────────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐          ┌──────────────────┐        ┌─────────────────┐
│  ETL SERVICE  │          │  WEBS RESEARCH    │        │  LLM SERVICE    │
│  (sync/CDC)   │          │  (Serper/Brave)   │        │  (OpenAI/Anthropic)│
└───────┬───────┘          └────────┬─────────┘        └────────┬────────┘
        │                           │                           │
        └───────────────┬───────────┴───────────┬───────────────┘
                        ▼                       ▼
              ┌─────────────────┐      ┌──────────────────────────┐
              │  PostgreSQL     │      │   EMBEDDING PIPELINE      │
              │  + pgvector     │      │   (profili, prodotti,     │
              │  (master data,  │      │    fornitori, prospect)   │
              │   docs, vectors)│      └────────────┬─────────────┘
              └─────────────────┘                   │
                        ▲                           │
                        │                           │
              ┌─────────▼───────────────────────────▼──────────┐
              │                 BACKEND API (Next.js)            │
              │  Customer Intelligence · Prospecting · Comms     │
              │  AGENT ORCHESTRATOR (ufficio marketing IA)       │
              └─────────┬──────────────────────────┬───────────┘
                        │                          │
                        ▼                          ▼
              ┌────────────────────┐      ┌──────────────────────────┐
              │  FRONTEND          │      │   DELIVERY SERVICE        │
              │  (Dashboard,       │      │   Email (Resend/SES)     │
              │   schede, coda,    │      │   WhatsApp Business API   │
              │   visual stand)    │      └──────────────────────────┘
              └────────────────────┘
                        │
                        ▼
              ┌──────────────────────┐
              │  IMAGE GEN SERVICE   │  (visualizzazione generativa stand)
              │  stand vuoto +       │   es. Gemini Image / gpt-image /
              │  immagini prodotto → │   compositing ibrido + controllo
              │  visual allestito    │
              └──────────────────────┘
```

### 3.1 Principi architetturali

1. **Sola lettura sul gestionale (vincolo assoluto)**: nessuna scrittura su Integra, **MAI** — né `INSERT`/`UPDATE`/`DELETE`, né DDL, né trigger, né modifiche di configurazione. Il sistema estrae dati dall'integration database dedicato (replica read-only o connessione RO) e **scrive esclusivamente nei propri database applicativi**. La read-only-ness è verificata tecnica e operativa (vedi §4.1, §16.2).
2. **Disaccoppiamento**: ogni servizio (ETL, arricchimento, analisi, generazione testuale, generazione immagini, delivery) è indipendente e orchestrato da code/worker.
3. **Human-in-the-loop**: nessuna comunicazione parte senza approvazione (parametrizzabile per segmento/cliente).
4. **Tracciabilità**: ogni insight genera, comunicazione e invio è loggato e riferito ai dati di origine.
5. **Privacy by design**: minimizzazione, pseudonimizzazione, gestione del consenso come previsto da GDPR.

---

## 4. Integrazione con il gestionale Integra

### 4.1 Modalità di accesso

Il database di Integra è **PostgreSQL** (dato confermato). Integra non espone API REST documentate: l'accesso avviene a livello di **SQL diretto in lettura**.

| Modalità | Note |
|---|---|
| **SQL diretto (driver `pg`)** | Connessione diretta all'istanza PostgreSQL di Integra con **utenza dedicata in sola lettura** e schema di sicurezza ridotto (solo `SELECT`, niente `USAGE` su sequenze/tabelle di sistema, `default_transaction_read_only = on`). |
| **Replica PostgreSQL (consigliata)** | Replica fisica o logica (streaming / logical replication) su istanza dedicata: il nostro ETL legge dalla replica, senza carico sul database produttivo. **Pattern di default**. |
| **Viste dedicate** | Viste logiche esposte con i soli campi necessari, per isolare lo schema interno. |
| **File di scambio** | Export CSV/XML schedulati come **fallback** (fragile, ultima risorsa). |

**Decisione di default**: configurare **replica PostgreSQL read-only** (logical replication) verso un'istanza dedicata, oppure — se la concessione di replica non è praticabile — connessione `pg` in sola lettura sul master con query ottimizzate e fuori orario di punta.

### 4.1.1 Differenze da valorizzare (PostgreSQL)

- **parser del driver nativo**: niente layer ODBC intermedio; connessione diretta via `pg` (Node).
- **logical replication / pub-sub**: idoneo per CDC incrementale nativo (`pgoutput`), preferibile ai delta su `updated_at`.
- **pgvector lato stesso motore**: il DB applicativo e il DB di staging possono essere la stessa tecnologia, semplificando repliche e tooling.
- **safety**: utente `decobrands_ro` con `GRANT SELECT` su sole tabelle/viste concordate; transazioni `READ ONLY`; query con `statement_timeout`; nessun `TRIGGER`/`EVENT TRIGGER` sul master.
- **verifiche di fattibilità**: confermare in Fase 0 la disponibilità di replica logica dalla casa produttrice di Integra (mapping delle tabelle interne via `information_schema` e `pg_stat_user_tables`).

### 4.2 Entità da estrarre

| Entità | Campi principali | Uso |
|---|---|---|
| `clienti` | id, ragione sociale, P.IVA/CF, indirizzo, città, provincia, telefono, email, agente, data inserimento, classe/mercato, canale | Profilo, RFM, comunicazioni |
| `contatti_clienti` | email, telefono, ruolo, consenso marketing | Canali di invio |
| `ordini` | id, data, id cliente, importo, sconto, stato, agente, magazzino | Storico, RFM |
| `righe_ordine` | id ordine, id articolo, quantità, prezzo unitario, sconto | Affinità prodotto, segmentazione |
| `prodotti` | codice, descrizione, categoria, sottocategoria, marca/linea, costo, listino, gru, foto | Catalogo, embedding |
| `fornitori` | id, ragione sociale, P.IVA, categoria merce, condizioni | Enrichment web |
| `listini` | id prodotto, prezzo, sconto per classe cliente, validità | Offerte, margini |
| `incassi/fatture` | importi, date scadenza, saldo | Affidabilità, alert |

### 4.3 Strategia di sincronizzazione

> **Ricordo del vincolo**: tutte le operazioni qui descritte avvengono **in lettura sul DB di Integra (o sulla sua replica)**. Il flusso di dati è **one-way**: Integra → staging/applicativo. Non esiste alcun "write-back" verso il gestionale, in nessuna direzione o condizione.

- **Notte (full snapshot delta)**: refresh notturno delle tabelle di staging (clienti, prodotti, listini) a volume pieno ma incrementale per ordini/righe (`updated_at` / deposito movimenti).
- **CDC / eventi (incrementale)**: su PostgreSQL privilegiare la **logical replication** (`pgoutput`) verso l'istanza applicativa per ordini/righe e anagrafiche modificate; in alternativa delta su colonna di modifica. Frequenza: repliche continue, job applicativi ogni 30-60 min.
- **Consistenza**: chiavi naturali (es. codice cliente Integra) mantenute come chiavi esterne nel modello master. Idempotenza su ogni corsa di ETL.
- **Retry e monitoraggio**: code BullMQ con retry, alerting su stato sync, semafori anti-sovrapposizione.
- **Verifica continua di read-only**: ogni job ETL esegue all'avvio un check di scrittura negata sulla connessione a Integra; se il check fallisce (cioè la connessione risulta scrivibile) il job si **blocca** e fa alert.

### 4.4 Cache di sincronizzazione

In PostgreSQL, tabelle `sync.{entità}` speculari delle staging con:

- intestazione di processo (id corsa, timestamp, esito);
- flag `dirty` per documenti da re-embedding;
- log di trasformazione (mapping campi, outlier).

---

## 5. Modello dati

Lo schema applicativo usa **PostgreSQL 16** e gestisce le seguenti famiglie di tabelle.

### 5.1 Master data (normalizzati dal gestionale)

- `customers` — profilo commerciale cliente + dati anagrafici + consenso.
- `customer_contacts` — email, telefono, ruolo, canale preferito.
- `products` — catalogo articoli.
- `product_categories` — gerarchia categorie/sottocategorie.
- `suppliers` — fornitori.
- `price_lists` — listini per classe cliente.
- `orders` / `order_items` — storico vendite.

### 5.2 Dati arricchiti

- `customer_profiles_web` — info web del cliente: sito, anno fondazione, dimensione, sede, social, notizie, mercati serviti, marchi trattati.
- `product_descriptions_enriched` — descrizione estesa generata (materiali, utilizzo, target, contesti d'uso, parole chiave SEO).
- `supplier_profiles_web` — profilo web fornitori.
- `web_sources` — riferimento alla fonte (URL, data, crawl_id), per tracciabilità.

### 5.3 Vettori (pgvector)

- `embeddings_customer_profile` — profilo semantico cliente (1 vettore per cliente + versioni).
- `embeddings_product` — embedding per prodotto.
- `embeddings_category` — embedding per categoria.
- `embeddings_supplier` — embedding per fornitore.
- `embeddings_prospect` — embedding profilo prospect.

Ogni tabella: `entity_id`, `embedding vector(1536)` (o dimensione modello scelto), `model`, `version`, `created_at`.

### 5.3.1 Immagini e visual generativa (stand)

- `product_images` — immagini prodotto: immagine ufficiale, **cutout con sfondo trasparente** (PNG), angolazione, risoluzione, fingerprint/checksum, fonte (gestionale/web), stato (da validare/valido/scartato).
- `stand_photos` — foto degli **espositori/stand vuoti** dei clienti: id cliente, tipo espositore, immagine, condizioni luce, posizione, data, consenso all'uso per generazione, stato.
- `image_generations` — log di ogni generazione: id proposta/comunicazione, input (id foto stand, id prodotti/cutout, varianti), modello usato, prompt, parametri, immagine output (e thumbnails), post-processamento, esito validazione (consistenza prodotti, geometria), costo, revisore e stato (bozza/approvato/scartato/rigenera).
- `image_generation_variants` — varianti di allestimento per la stessa proposta (disposizioni, temi, stagioni) con score di qualità.
- `visual_assets` — binari immagini e relativi metadati (storage locale Windows su filesystem, riferimenti da DB).

### 5.4 Modello analitico

- `rfm_scores` — punteggi Recency/Frequency/Monetary per cliente.
- `customer_clusters` — segmentazione (per comportamento, categoria, area).
- `product_affinity` — matrice affinità prodotto-categoria per cliente/segmento.
- `customer_insights` — insight strutturati cliente per cliente (vedi §8).
- `seasonality` — curve stagionali per categoria/cliente.

### 5.5 Prospecting

- `prospects` — prospect scoperti (R2B: ragione sociale, P.IVA se nota, area, dimensione).
- `prospect_lookalikes` — mapping prospect → clienti simili (score di similarità).
- `prospect_product_affinity` — prodotti/categorie stimati affini al prospect.
- `prospect_score` — score complessivo (fit, dimensione, probabilità conversione).

### 5.6 Comunicazioni e campagne

- `campaigns` — campagna, segmento, obiettivo, stato, date.
- `communications` — bozze generate per cliente/prospect: canale, prodotto suggestionato, stato (bozza → revisione → approvato → inviato → esito), tracciabilità.
- `communication_templates` — template con variabili e linee guida.
- `delivery_log` — esito invio (email: open/click/bounce; WhatsApp: delivered/read), webhook provider.
- `approval_queue` — coda di approvazione con assegnazione al venditore.
- `consent_log` — consensi marketing per cliente/canale con data e fonte.
- `events` — eventi generici per nurturing (riordino, inattività, lancio, scadenza listino).

### 5.7 Schema ER sintetico

```
customers 1─N customer_contacts
customers 1─N orders 1─N order_items N─1 products
products N─1 product_categories
products N─1 suppliers
products 1─N price_lists
products 1─N embeddings_product
customers 1─N customer_profiles_web
customers 1─N rfm_scores
customers 1─N customer_insights
prospects N─M customers (prospect_lookalikes)
prospects 1─N prospect_product_affinity
campaigns 1─N communications N─1 customers
campaigns 1─N approval_queue
communications 1─N delivery_log
```

---

## 6. Pipeline di arricchimento semantico

### 6.1 Obiettivo

Aumentare la densità semantica dei dati master con conoscenza esterna, coerente e tracciata. L'arricchimento è asincrono, a coda, con retry e rate limiting.

### 6.2 Fonti web

| Dato | Fonti | Note |
|---|---|---|
| Profilo operatori economici | Poste.it, registroimprese, Visura, PagineGialle, sito aziendale, LinkedIn, stampa locale | Ragione sociale, P.IVA, settore, dimensione, anno fondazione |
| Notizie/sviluppi clienti | Google News, stampa di settore | Apertura nuova sede, ristrutturazione, cambio marchi |
| Descrizione prodotti | Sito produttore, cataloghi, e-commerce, marketplace | Materiali, finiture, misure, contesti d'uso, target |
| Profilo fornitori | Sito, catalogo, fiere | Categorie merce, novità, posizionamento |
| Prospect | Stesse fonti operatori + verifiche P.IVA | Arricchimento qualificazione |

### 6.3 Pipeline a stadi

1. **Ingestion**: web search (Serper/Brave) per chiavi di ricerca costruite da dati anagrafici (es. `"{ragione sociale}" + {città}`).
2. **Crawl/parse**: estrazione testo significativo dalle pagine pertinenti (URL ritorno della ricerca), pulizia HTML, dedupe.
3. **Extraction**: LLM strutturato (capabilities + tool call con output JSON) estrae i campi definiti per entità.
4. **Validation**: controlli (formato P.IVA, coerenza città/indirizzo, punteggio di confidenza).
5. **Store**: scrittura in `*_profiles_web` + `web_sources` con tracciabilità sorgenti.
6. **Embedding**: aggiornamento dei vettori connessi (vedi §7).
7. **Human review su bassa confidenza**: record con confidenza sotto soglia finiscono in coda di verifica.

### 6.4 Esempi di estrazione

**Cliente (showroom):**

```json
{
  "ragione_sociale": "GALLERIA MOSAICO SRL",
  "settore": "Showroom superfici e arredo",
  "citta": "Milano",
  "anno_fondazione": 2005,
  "dimensione": "pmi",
  "mercati_serviti": ["architetti", "privati", "contract"],
  "marchi_trattati": ["Casa.it", "Fornasetti", "Decobrands"],
  "segnali": { "ricostruita": true, "sede_riaperta_2025": true },
  "confidenza": 0.93,
  "fonti": ["https://..."],
  "data_estrazione": "2026-09-12"
}
```

**Prodotto:**

```json
{
  "codice": "DB-208",
  "descrizione_estesa": "Pannello decorativo in fibrocemento 3D...",
  "materiali": ["fibrocemento", "acciaio inox"],
  "destinazioni_uso": ["rivestimento pareti", "controssoffitti", "facciate"],
  "target_suggested": ["showroom bagno", "architetti", "contract"],
  "keywords": ["parete effetto 3d", "rivestimento decorativo"],
  "confidenza": 0.85,
  "fonti": ["https://..."],
  "data_estrazione": "2026-09-12"
}
```

### 6.5 Volume e rate limiting

- Budget giornaliero query web (es. 5.000/day con Serper), code prioritarie: clienti attivi > clienti inattivi > prospect > fornitori.
- Cache delle ricerche per query identica (retry in finestre successive).
- Embedded deduplication e merge per P.IVA.

### 6.6 Estrazione dati non strutturati (immagini, cataloghi, foto stand)

Parte dei dati delle **schede prodotto/cliente/fornitore (UC-09)** non esiste in Integra e va ricavata in automatico da fonti non strutturate, sotto presidio umano (§2.1, UC-10). Pipeline dedicata, per tipo di fonte:

| Fonte | Input | Estrazione IA | Output atteso | Uscita |
|---|---|---|---|---|
| **Immagini prodotto** | file immagine (PNG/JPG) dal gestionale o da fornitori | Vision LLM: riconoscimento materiali, finiture, misure visibili, colori, forma | attributi prodotto proposti (fedeltà alta a ciò che è visibile) | `product_extractions` (candidata) |
| **Cataloghi PDF fornitori** | file PDF (scansioni o nativi) | estrazione documentale + Vision/OCR: righe catalogo, refs, descrizioni, caratteristiche tecniche, foto | schede articolo fornitore + mappatura a prodotti | `supplier_catalog_items` + `catalog_documents` (candidata) |
| **Foto stand vuoto** | foto dell'espositore del cliente | classificazione: tipo espositore, dimensioni, vincoli, condizioni luce | metadati stand per UC-07/UC-08 | `stand_photos` (candidata) |
| **Foto prodotti proposti** | immagini articoli già in catalogo | verifica di fedeltà (embedding visivo cutout vs output, §10.10) | cutout validati per visual | `product_images` stato `valido` |

**Presidio umano (UC-10)**: ogni estrazione entra come **candidata** con punteggio di confidenza, fonte, modello e versione; finisce in una **coda di revisione (§6.7)**. L'operatore dati (stagista) la valida, corregge o scarta; solo dopo l'approvazione l'attributo entra nella scheda versionata (§5.3).

### 6.7 Coda di revisione dati (operatore / stagista) e presidio umano

Workflow human-in-the-loop sui dati, gestito dall'operatore dati (§2.1) tramite la dashboard §11.4:

1. **Arrivo**: ogni estrazione IA (§6.6), scheda completata, immagine generata (§10.10) o dato inserito manualmente entra in coda con stato `da revisionare`.
2. **Filtri** per: entità (prodotto/cliente/fornitore/stand), tipo attività (inserimento/correzione/validazione/approvazione), priorità (confidenza bassa, dati critici), origine (IA/manuale).
3. **Revisione**: l'operatore apre la scheda, confronta con la fonte (immagine/PDF/gestione), corregge o approva; per gli output IA può richiedere **rigenerazione** con feedback (es. "misura dichiarata errata: la ref dice 120×60").
4. **Esito**: `approvato` → aggiorna la scheda master (§5.3) con traccia; `corretto` → salva correzione (storico delta); `scartato` → log dei motivi; `posticipato` → ritorno in coda con priorità.
5. **KPI operativi**: codice §18.x, ad es. **copertura dati** (quota di prodotti con scheda arricchita, target ≥ 80% top-catalogo) e **tempo medio di svuotamento coda** (target ≤ 1 ciclo commerciale).

Il presidio umano dello stagista è quindi **strutturale per il MVP** (non opzionale): garantisce qualità e grounding prima che i dati vengano usati per insight, lookalike e comunicazioni.

> **Nota organizzativa**: anche in futuro con maggiore autonomia IA (Fase 9), l'operatore dati mantiene un ruolo di **oversight sulla qualità dati**; ciò che cambia è solo la frequenza d'intervento (sempre eccezioni, mai per routine).

---

## 7. Layer RAG e Vector Store

### 7.1 Progettazione

- **Database**: PostgreSQL 16 con estensione **pgvector**.
- **Modello di embedding**: configurabile via env (default `text-embedding-3-small`, 1536 dim; alternativa `3-large` o modello open-source locale se richiesto da vincoli di costo/privacy).
- **Documenti indicizzati**:
  - profilo cliente arricchito (testo sintetico generato da dati gestionali + web);
  - descrizione prodotto estesa;
  - descrizione categoria;
  - profilo fornitore;
  - profilo prospect (in fase di qualificazione).

### 7.2 Costruzione del documento indicizzato

Per ogni cliente viene generato un **documento sintetico** (profilo) che unisce:

- dati anagrafici e commerciali (fatturato, categorie acquistate, RFM, stagionalità);
- affinità prodotto;
- informazioni web arricchite (settore, dimensioni, marchi trattati, segnali);
- stato relazione (attivo/inattivo/nuovo).

Questo documento è la base per:
- similarità cliente-cliente (lookalike);
- similarità prospect-cliente;
- retrieval contestuale per la generazione delle comunicazioni.

Analogamente, un documento sintetico per categoria/prodotto riassume descrizione estesa, materiali, utilizzo, target e performance commerciali.

### 7.3 Retrieval ibrido

Per ogni richiesta contestuale si esegue:

1. **Vector search** (cosine similarity, HNSW/IVFFlat index) nella collezione pertinente;
2. **Keyword/full-text search** (tsvector di PostgreSQL) per corrispondenze esatte su nomi, codici, categorie;
3. **Structured filter** (RFM, segmento, categoria, area, stato) applicato prima del ranking.

I risultati vengono **fusi via RRF (Reciprocal Rank Fusion)** o weighted ranking configurabile, con soglie di confidenza.

### 7.4 RAG utilizzato nella generazione

Il generatore di comunicazioni usa il pattern Retrieval-Augmented Generation:

- **Query di intent** (es. "proposta riordino categoria superfici decorative per cliente attivo in Lombardia con churn categoria qualificato");
- **Retrieval contestuale**: profilo cliente, affinità, prodotti candidati, storia recente, template associato;
- **Context assembly**: costruzione del prompt con i soli documenti pertinenti (top-k) e i vincoli (brand voice, canale, lunghezza, no-hallucination su prezzi/margini);
- **Grounding**: ogni asserzione fattuale deve potersi riferire a un dato di origine (profilo, ordine, prodotto). Vincolo hard: **mai inventare prezzi, sconti o dati anagrafici**; prezzi sempre letti dal listino attivo.

### 7.5 Versioning e manutenzione

- Ogni embedding ha `version` e `model` per permettere refresh mirato.
- Re-embedding automatico dei record `dirty` dopo ETL o enrichment.
- Shadow compare tra versioni modello prima della promozione.

---

## 8. Modulo Customer Intelligence

### 8.1 Scheda cliente 360°

Funzionalità della scheda:

1. **Profilo**: anagrafica, contatti, consensi, dati web arricchiti.
2. **Storico**: ordini, importi, frequenza, ticket medio, curve di vendita.
3. **RFM**: punteggi 1-5 e cluster (Champions, Loyal, At-Risk, Churn, New).
4. **Categorie preferite**: top categorie per ricorrenza e margine.
5. **Affinità prodotto**: prodotti consigliati (cross-sell/upsell) con motivazione basata su storico.
6. **Stagionalità**: mesi/periodi di picco per il cliente.
7. **Segnali**: variazioni anomale (inattività, calo categoria, cambio mix), eventi esterni dal web.
8. **Insight testuali**: 1-3 insight in linguaggio naturale generati da LLM con dati alla mano.
9. **Storico comunicazioni**: messaggi inviati, esiti, reazioni.

### 8.2 Calcolo insight (periodico)

Job giornaliero/settimanale `insight-generator` per ogni cliente attivo:

- analisi di trend per categoria (confinienza > soglia);
- rilevamento anomalie (calo ricorrenza, aumento mix, stagione in avvicinamento);
- confronto con segmento omogeneo (benchmark);
- generazione insight strutturati JSON + testo, con `grounded_facts[]` (lista dei dati di supporto).

**Esempio insight:**

```json
{
  "customer_id": "C-123",
  "date": "2026-09-12",
  "type": "CROSS_SELL_OPPORTUNITY",
  "confidence": 0.87,
  "text": "Il cliente ha incrementato del 22% gli acquisti della categoria 'superfici lavabili' negli ultimi 2 trimestri. Il nuovo articolo DB-880 nella stessa categoria non è stato ancora acquistato. Proposta mirata suggerita.",
  "grounded_facts": ["ordini 2026-Q1/Q2 categoria 42", "DB-880 mai in ordini del cliente", "similar_customers: 34/100 hanno cross-sold DB-880"],
  "suggested_product_codes": ["DB-880"],
  "suggested_campaign": "CROSS_SELL_NOVITA"
}
```

### 8.3 Priorità vendita

Score di priorità per cliente (utilizzato nel frontend e nelle campagne):

```
priority = f(RFM, potenziale stimato, margine medio, trend, segnali esterni)
```

Il potenziale stimato deriva dai clienti lookalike con fatturato superiore (median benchmark). Il risultato alimenta la lista "clienti da contattare oggi".

---

## 9. Modulo Prospecting

### 9.1 Obiettivo

Trovare prospect (showroom/rivenditori non ancora clienti) e:

1. qualificarne il profilo (settore, dimensione, area, marchi trattati);
2. confrontarli con clienti già acquisiti simili (**lookalike matching**);
3. stimare l'affinità con prodotti in catalogo (**product affinity prediction**);
4. assegnare uno score di priorità e generare la prima comunicazione di avvicinamento.

### 9.2 Scoperta prospect

- **Seed list**: elenchi forniti dall'ufficio commerciale (fiere, contatti raccolti, provider dati).
- **Web discovery**: ricerca strutturata per settore/area (es. "showroom superfici decorative Milano", "rivenditore pavimenti brescia"), crawl siti, estrazione P.IVA e dati chiave.
- **Dedupe coi clienti esistenti**: un operatore che risulta già cliente non può comparire tra i prospect.

### 9.3 Lookalike matching

Usa embedding del **profilo semantico**:

- prospetto `p` → embedding profilo prospect;
- per ogni cliente `c` → embedding profilo cliente;
- `similarity(p, c)` via cosine/pgvector, top-k > soglia;
- aggregazione: profilo aggregato dei k lookalike → attributi attesi (categorie, ticket medio, mix) con pesi in base a similarità.

### 9.4 Affinità prodotti per prospect

Dai lookalike si ricostruisce la matrice di affinità trasferita:

- top categorie acquistate dai lookalike → categorie probabili per il prospect (score = Σ similarità × peso categoria);
- top prodotti consigliati = prodotti più acquistati dai lookalike nella finestra recente, filtrati per disponibilità e margine;
- output `prospect_product_affinity` con score normalizzato.

### 9.5 Score prospect

```
score = w_fit * fit(segmento/profilo) + w_dim * dimensione stimata
      + w_aff * affinità prodotti + w_time * recenza segnale
```

Le soglie di "workable" (es. score ≥ 60) determinano la pipeline di avvicinamento.

### 9.6 Prima comunicazione

Generazione della comunicazione di apertura: reference ai prodotti affini (con rename/descrizione corretta), accenno a casi simili ("clienti come voi nel settore X hanno scelto..."), valore e invito. Approvazione richiesta prima dell'invio (vincolo MVP).

---

## 10. Modulo Generazione Comunicazioni (priorità MVP)

Questo è il modulo **prioritario** della prima versione.

### 10.1 Tipologie di comunicazione (MVP)

| Tipo | Trigger | Canale | Contenuto |
|---|---|---|---|
| **Proposta su novità (cross-sell)** | nuove referenze in categoria amata | Email / WhatsApp | 1-3 prodotti nuovi coerenti col profilo |
| **Proposta riordino** | scadenza ciclo di consumo, giacenze | Email | articoli di riacquisto con quantità suggerite |
| **Re-engagement inattivi** | nessun ordine da N mesi | Email | valore della relazione, novità, invito |
| **Sconto/condizioni dedicate** | segmento/cliente prioritario | Email | condizioni listino attive applicate |
| **Evento/fiera** | calendario eventi | Email | invito con focus prodotti esposti |
| **Prospect avvicinamento** | nuovo prospect qualificato | Email | presentazione con prodotti affini |
| **Cocktail/periodica leggera** | cadenza programmata | WhatsApp | notizia/consiglio di prodotto breve e pertinente |
| **Proposta con visual stand** | proposta commerciale (UC-03 + UC-07) | Email / WhatsApp | testo + **immagine generata dello stand con i prodotti proposti** (vedi §10.10) |

### 10.2 Architettura di generazione

```
Campagna o trigger evento
        │
        ▼
[SELECT segment/customer] ──► [RAG context assembly] ──► [LLM generation]
                                                              │
        ◄───────────────────────────────────────────────────  │
        ▼
[Validazione automatica]  (fatti, prezzi, lunghezza, canale, no-duplicati, tono)
        │
        ▼
[Human approval]  ──► [Delivery service] ──► [Esiti / delivery log]
        │
        ▼
[Feedback → apprendimento template/tono]
```

### 10.3 Contesto RAG per comunicazione

Ogni bozza riceve:

1. **Profilo cliente** (§7.2), top-k chunk pertinenti.
2. **Prodotti candidati**: calcolati da affinità + nuovi arrivi + disponibilità + margine (mai più di 3 prodotti con forti motivazioni).
3. **Storico comunicazioni recenti**: per evitare ripetizioni e tempistiche troppo ravvicinate.
4. **Template/guideline**: struttura, brand voice, regole canale (es. WhatsApp ≤ 900 caratteri, tono informale).
5. **Listino attivo**: per citare condizioni reali (mai inventate).

### 10.4 Vincoli di generazione (hard)

- Nessun prezzo/sconto inventato: solo dati da `price_lists` attivi.
- Nessun dato anagrafico inventato: solo da master + profilo web validato.
- Italiano coerente con brand voice configurabile (per Decobrands: tono consulenziale, tecnico ma accessibile).
- Lunghezza per canale (email ≤ ~150 parole soggetto+titolo; WhatsApp ≤ ~900 caratteri).
- Evitare proposizioni speculative per il cliente (niente "so che vuoi...").
- **Sempre** un chiaro invito all'azione e un referente/contatto commerciale.
- Ogni comunicazione deve superare validazione automatica prima dell'approvazione umana.

### 10.5 Validazione automatica

Lista controlli eseguibili a basso costo prima della coda di approvazione:

| Controllo | Descrizione |
|---|---|
| Grounding check | ogni prezzo/codice/quantità presente nella bozza esiste nei dati |
| No-hallucination | LLM judge su asserzioni fattuali non tracciabili |
| Duplicati | stessa proposta inviata di recente? (soppressione) |
| Frequenza | rispetta il rate limit per canale/cliente |
| Formato/canale | lunghezza, emoji consentite, link sanitizzati |
| Tonality | check brand voice (baseline positive, no clickbait) |
| Blocco | clienti gemellati se consentito; lista di non-contatto |

### 10.6 Approval workflow (human-in-the-loop)

1. Bozza generata → stato `draft`.
2. Validazione ok → coda `approval_queue` assegnata al venditore di riferimento.
3. Venditore: revisiona, modifica (accettate), approva o scarta con nota.
4. Approvato → piano di invio (orario finestra, dedupe) → delivery.
5. Esito catturato (open/click/read/reply) → feedback al profilo.

Configurazione per il futuro (vedi §2.3): gli agenti dell'**ufficio marketing IA** ereditano questo workflow. Livello L1 (MVP): l'agente propone e l'umano approva. Livello L2: **auto-invio** per segmenti a basso rischio e B2B consolidato (whitelist), sempre disattivabile, con eccezioni ad alto impatto in approvazione. Livello L3: ciclo completo autonomo entro KPI e guardrail.

### 10.7 Delivery service

**Email** — provider transazionale/invio (es. Resend o Amazon SES): template HTML responsive, autenticazione SPF/DKIM/DMARC, domini dedicati, tracking open/click.

**WhatsApp Business API** — via Meta Cloud API:

- numeri verificati / display name approvato;
- **template WhatsApp** pre-approvati da Meta (message template con variabili: `{{1}} nome`, `{{2}} prodotto`, `{{3}} link`);
- limiti di messaggistica (24h window + template), tipo conversation;
- webhook per stato `delivered`/`read`.

**Coerenza multicanale**: un cliente non riceve la stessa proposta su due canali nella stessa finestra (dedup per tipo+cliente+finestra temporale configurabile).

### 10.8 Template e brand voice

- `communication_templates` con:
  - nome, canale, tipo;
  - block di contenuto (oggetto, preheader, corpo, CTA);
  - variabili ammesse;
  - regole di tono e lunghezza.
- Il tono è gestito via linee guida nel prompt (per modello/LLM indipendente).
- Base: consulenziale e professionale; per WhatsApp più conciso.

### 10.9 Apprendimento

Log di esiti (open, click, risposta, ordine generato) alimenta:

- metriche per template e tipologia;
- aggiustamento automatico della frequenza (se saturazione, riduci);
- segnali per perfezionare le proposte (quali prodotti/CTA convertono).

### 10.10 Visualizzazione generativa dello stand (AI Image)

Obiettivo: **fotorealismo nella proposta B2B** — mostrare al cliente come lo stand/espositore attuale ospiti gli articoli proposti. Da foto dello stand vuoto (o esistente) + immagini dei prodotti selezionati → immagine dello stand allestito con i prodotti proposti, con fedeltà sui prodotti (stesso colore/finitura) e coerenza con spazio reale.

#### 10.10.1 Fattibilità

La tecnica è **fattibile** con i modelli di generazione/edit immagine attuali, che supportano input multipli (foto di scena + immagini di riferimento prodotto). Approccio consigliato **ibrido**:

1. **Cutout prodotto**: immagini prodotto con sfondo trasparente (PNG `product_images`, estratte con segmentazione dedicata o fornite dal catalogo/brand).
2. **Allestimento guidato (compositing controllato)**: sistema geometrico che definisce posizione/perspective/size dei prodotti nello stand (bounding box sull'area espositiva), preservando il layout reale.
3. **Generazione/fusione**: modello di image-editing (es. Gemini Image / gpt-image-1 e simili) che integra i cutout nella scena con luci/ombre coerenti, oppure **inpainting** su maschere, oppure compositing + post-process (blend, color match).
4. **Verifica di fedeltà**: confronto tra i prodotti inseriti e le immagini originali (embedding visivo / CLIP-similarity per prodotto; controllo che il modello non abbia alterato design/finitura).

**Vantaggi dell'approccio ibrido**: geometria e disposizione controllabili (un allestimento fisico non inventa posizioni assurde), prodotto non "ricreato" ma quello reale, consistenza marca, iterazioni rapide. Limite: qualità luce/riflessi dipendente dal modello; per stand critici si può usare generazione libera con reference per esplorare layout creativi.

**Modello sale**: per volumi B2B contenuti (decine-centinaia di immagini/anno) i costi API immagine sono trascurabili (vedi §17). Se in futuro i volumi crescono molto (migliaia), valutare modello self-hosted (es. Stable Diffusion + ControlNet + LoRA dei prodotti).

#### 10.10.2 Input

- Foto dello stand/espositore vuoto (o attuale) del cliente (`stand_photos`), con metadati (posizione, angolo, luce).
- Set di prodotti selezionati (da Product Curator / campagna), con `product_images` (cutout PNG preferito) e varianti.
- Vincoli: numero prodotto per stand (tipicamente 3–8), marca/canale, tema stagionale, orientamento (verticale per VS/immagine email, quadrato per social/WhatsApp).
- Consenso cliente all'uso della foto dello stand a fini di proposta (tracciato, revocabile).

#### 10.10.3 Pipeline

1. **Pre-flight**: verifica cutout disponibili (manca → genera cutout con tool di segmentazione), verifica consenso foto stand, verifica quota/costo.
2. **Layout planning**: placement dei prodotti sulla foto (zona espositiva), opzioni di disposizione.
3. **Generation**: chiamata al servizio immagine (modello config, prompt strutturato, reference/cutout), retry e budget.
4. **Post-process**: upscaling opzionale, rimozione artefatti, crop/formati per il canale.
5. **Validation**: check fedeltà prodotto (similarità visiva vs originali), check "nessun prodotto estraneo/locale crollato", punteggio qualità; se sotto soglia → rigenera (max N tentativi) o scarta.
6. **Human review**: il venditore vede la variante in coda approvazione della comunicazione; può rigenerare, scegliere tra varianti o usare il fallback senza visual.
7. **Store & traccia**: `image_generations` + `visual_assets`, costo, prompt, modello, esito.

#### 10.10.4 Integrazione con le comunicazioni

- Il visual è **parte opzionale** della comunicazione UC-03/UC-07: la bozza (testo + immagine) viaggia insieme; in coda approvazione il venditore vede testo e visual in anteprima.
- Formati: email (oggetto/preheader + visual nel corpo + CTA), WhatsApp (una o più immagini con didascalia — 3 max per template media).
- Il fallback senza visual deve sempre esistere (se immagine non generata/rifiutata la comunicazione resta valida).
- In futuro (L2/L3 ufficio marketing IA): il Product Curator propone i prodotti; un agente "Visual Designer" compone e propone la variante migliore; approvazione o solo supervisione a seconda del livello.

#### 10.10.5 Qualità e vincoli

- **Fedeltà prodotto** sopra ogni cosa: mai alterare design, colori, finiture dei prodotti reali; i cutout sono predefiniti e non soggetti a interpretazione libera del modello.
- Coerenza dello spazio: niente prodotti che fluttuano, si sovrappongono in modo irreale o invadono aree non espositive.
- Niente loghi/marchi estranei, niente testi (salvo segnaletica/promo reale del cliente consentita).
- Consenso foto stand + termini provider immagine documentati.
- Audit: ogni output registrato con input, modello, prompt, costo e revisione umana.

#### 10.10.6 Metriche del visual

| Metrica | Soglia |
|---|---|
| Fedeltà prodotto (similarità visiva cutout vs output) | ≥ 0.90 (embedding visivo) |
| Tasso di visual approvati senza modifiche | ≥ 80% |
| Tempo di generazione end-to-end | ≤ 3 min |
| Costo per immagine | budget definito (vedi §17) |

---

## 11. Frontend

Single-page **Next.js (App Router)** — verticale, ruoli e scope.

### 11.1 Pagine principali

| Pagina | Contenuto |
|---|---|
| **Dashboard** | KPI canale: clienti attivi, inattivi, potenziale, campagne in corso, coda approvazione, invii/risultati |
| **Clienti** | lista con filtro (RFM, segmento, area, agente, stato) e priorità |
| **Scheda cliente** | 360°: profilo, insight, affinità, storico, comunicazioni, consensi |
| **Prospect** | lista, score, lookalike, affinità prodotti, stato pipeline |
| **Campagne** | creazione, segmentazione (query builder), scheduling, stato |
| **Coda approvazione** | revisione/approvazione comunicazioni davanti al venditore |
| **Template** | gestione template e brand voice |
| **Analytics** | performance canale: open/click/risposta, conversioni, revenue |
| **Consensi** | gestione consenso e opt-out per cliente/canale (privacy) |
| **Attività/Log** | ETL, arricchimento, errori, retry |

### 11.2 Esperienza venditore (priorità)

- **Daily digest**: "5 clienti da contattare oggi" con motivazione (insight).
- **Modifica inline** delle bozze prima dell'approvazione.
- **One-click approva e invia** (o programma).
- Cronologia completa per cliente.

### 11.3 UX note

- Lang: italiano.
- Componenti: shadcn/ui + Tailwind.
- Dark/light mode neutra, mobile responsive (il venditore usa tablet in fiera).
- Dati reali con skeleton loading; WebSocket/SSE per aggiornamenti coda.

### 11.4 Dashboard operatore dati (stagista / back-office)

Il **presidio umano sui dati** (§2.1, UC-10, §6.7) è esercitato dall'operatore dati (stagista/back-office) tramite una dashboard dedicata. Accesso riservato al ruolo operatore (§12.5 RBAC); separata dalla vista venditore (§11.2) e dalle viste di direzione.

**Aree principali:**

| Area | Funzione |
|---|---|
| **Code di revisione** | Coda unificata con filtri per entità (prodotto/cliente/fornitore/foto stand), tipo attività (inserimento/correzione/validazione/approvazione/estrazione IA), priorità, origine (IA/manuale), stato. Recap grafico "coda oggi". |
| **Screener estrazioni IA** | Elenco candidati (§6.6): anteprima input (immagine/PDF/foto stand), output estratto, confidenza, fonte; azioni: **approva — correggi — scarta — rigenera**. Comparatore affiancato input vs output. |
| **Schede prodotto/fornitore/cliente** | Dettaglio scheda arricchita versionata (§5.3): campi Integra vs campi estratti IA vs campi manuali, evidenza sorgente per ciascun valore; editing manuale; flag "da validare". |
| **Inserimento manuale** | Form guidati per immissione dati non presenti in nessuna fonte (refs, caratteristiche, foto, dati fornitori) con obbligo di fonte/stato. |
| **Foto e visual stand** | Upload foto stand (consenso + metadati §5.3.1), revisione cutout prodotto (§6.6), validazione visual generati (§10.10), rigenerazioni. |
| **Qualità e copertura** | Metriche copertura dati per entità (es. % prodotti con scheda completa, % con cutout valido), confidenza media, code arretrate, errori ricorrenti. |
| **Cronologia / audit** | Log versioni, chi ha fatto cosa, delta tra versioni, undo. |

**Workflow in code** (§6.7): ogni elemento ha stato `da revisionare → approvato/corretto/scartato/rigenera`; il passaggio è tracciato con autore, data, nota. La dashboard guida l'operatore verso le priorità (dati critici, confidenza bassa, cross-check con catalogo).

**Presidio sulla generazione**: anche gli output di immagine (§10.10) e di copy (§9.4) transitano da questa dashboard dove serve controllo operatore, prima dell'approvazione finale del venditore (§11.2).

---

## 12. Stack tecnologico

### 12.1 Stack consolidato per MVP

| Componente | Tecnologia | Nota |
|---|---|---|
| Runtime | Node.js 20 LTS+ | Monorepo pnpm |
| Framework | Next.js 14/15 (App Router) | Backend + frontend |
| API/Backend | Next.js Route Handlers / Server Actions | oppure servizi worker separati |
| ORM | Prisma | Schema migrazioni |
| DB | PostgreSQL 16 + **pgvector** | Master data, vettori |
| Coda | Redis + **BullMQ** | ETL, enrichment, jobs |
| Cache | Redis | Rate limiting, dedupe |
| LLM | OpenAI GPT-4o-mini/Anthropic Claude options | Generazione, estrazione, judge |
| Embedding | text-embedding-3-small (default) | 1536 dim |
| Web search | Serper (riferimento) o Brave | Enrichment, prospect |
| Email | Resend o Amazon SES | Transazionale + template |
| WhatsApp | Meta WhatsApp Business Cloud API | Template approvati |
| **AI Image (generativa)** | Gemini Image / gpt-image-1 (o simili) + compositing ibrido (segmentazione cutout, placement, inpainting); verifica fedeltà via embedding visivo | Visual proposte su stand (vedi §10.10) |
| Auth | NextAuth/Auth.js + RBAC | Ruoli venditore/ufficio/direzione |
| Infra (Windows nativo) | Windows Server/S11 come piattaforma unica; PostgreSQL e Redis come servizi Windows; app Node runtime avviate come servizi/scaffolding dedicati (es. NSSM o Task Scheduler); reverse proxy IIS/ARR opzionale per terminazione TLS | **Nessun Docker/container**: tutto lo stack gira native su Windows |
| Monitoraggio | OpenTelemetry + Prometheus/Grafana (opzionale v1.1) | Log strutturati |

### 12.2 Deployment locale (Windows, senza Docker)

L'intero stack gira **nativamente su Windows** — l'uso di Docker/container non è previsto in nessun ambiente (locale, staging, produzione).

- **PostgreSQL 16 + pgvector**: installazione Windows nativa (zip/binari EDB oppure pacchetto standard) registrata come servizio Windows (`pg_ctl register` / installer). Nota: pgvector è incluso con PostgreSQL 16+ (contrib standard).
- **Redis**: servizio Windows nativo (binari/port ufficiale per Windows oppure Memurai come drop-in compatibile, per evitare WSL). Senza WSL o container.
- **Applicazione Next.js / worker BullMQ**: avvio tramite servizi registrati (es. NSSM / `sc.exe`) o Task Scheduler; log su file a rotazione; variabili d'ambiente in `.env` gestito.
- **Reverse proxy**: IIS + ARR (opzionale, per terminazione TLS e dominio) se richiesto dal cliente.
- **Script `scripts\*.ps1`**: gestione avvio/arresto/status dei servizi e dei processi app.
- **Seed script** con dati demo/anonimizzati per sviluppo e test (nel DB locale Windows).

---

## 13. API principali

Voci di API REST esposte dal backend (prefix `/api/v1/`), dashboard inclusa.

### 13.1 Customer Intelligence

- `GET /customers` — lista con filtri (RFM, segmento, area, stato).
- `GET /customers/:id` — scheda 360° aggregato.
- `GET /customers/:id/insights` — insight del cliente.
- `GET /customers/:id/affinity` — prodotti raccomandati con motivazioni.
- `GET /customers/:id/timeline` — eventi e comunicazioni.

### 13.2 Prospect

- `GET /prospects` — lista con score e filtri.
- `GET /prospects/:id` — dettaglio, lookalike, affinità prodotti.
- `POST /prospects/:id/lookalikes` — lancia ricalcolo lookalike.

### 13.3 Campaign e comunicazioni

- `POST /campaigns` — crea campagna con segmento e template.
- `GET /campaigns/:id/communications` — elenco bozze.
- `POST /communications/:id/approve` — approva (o `POST /communications/:id/reject`).
- `POST /communications/:id/schedule` — programma invio.
- `GET /communications/:id` — dettaglio e delivery log.

### 13.4 Amministrazione

- `POST /sync/run` — esegue corsa ETL manuale.
- `GET /sync/status` — stato ultima corsa.
- `GET /delivery/stats` — metriche invii/esiti.

---

## 14. Workflow ETL e cron

### 14.1 Job schedulati (BullMQ repeatable)

| Job | Cadenza | Descrizione |
|---|---|---|
| `etl.full.sync` | notte (01:00) | sync master data da staging Integra |
| `etl.incremental` | ogni 30-60 min (day) poi nightly | ordini/righe delta |
| `enrich.web.customers` | notte, limitato | aggiornamento profili web clienti priorità |
| `enrich.web.products` | settimanale | nuove descrizioni/novità |
| `enrich.web.suppliers` | settimanale | profili fornitori |
| `embedding.refresh` | post-etl | refresh vettori `dirty` |
| `insight.generator` | giornaliero (04:00) | insight/priorità per cliente |
| `campaign.trigger` | continuo (eventi) | lancia generazioni da trigger |
| `delivery.process` | continuo (coda) | invio approvati in finestra |
| `delivery.webhook.ingest` | continuo | esiti provider |
| `prospect.discovery` | settimanale | discovery + enrichment prospect |

### 14.2 Esecuzione e monitoraggio

- Ogni job: idempotente, con `attempts` e `backoff`.
- Tabelle di stato sync e log errori nel DB.
- Alert via (email/Slack) su fallimento ETL o coda piena.

---

## 15. Roadmap e fasi di sviluppo

La roadmap è descritta nel dettaglio (step, sub-step, verifica e test per ogni attività) in un documento dedicato:

> **`docs/roadmap-implementazione.md`** — guida operativa step-by-step per implementazione, verifica e test di ogni fase.

Sintesi delle fasi:

| Fase | Nome | Focus | Deliverable principale |
|---|---|---|---|
| **0** | Discovery Integra | Accesso DB PostgreSQL, mapping schema | `docs/integra-schema-map.md`, connettività testata, credenziali RO |
| **1** | Fondamenta e dati | Scaffold monorepo, DB/pgvector/Redis, ETL master data | Scheda cliente 360° con dati reali, qualità dati sotto controllo |
| **2** | Arricchimento semantico | Web research clienti/prodotti/fornitori, pipeline con tracciabilità | Profili arricchiti ≥80% clienti prioritari |
| **3** | RAG / Vector store | Documenti sintetici, embedding, retrieval ibrido | Retrieval con metrica di qualità ≥ soglia |
| **4** | Customer Intelligence | RFM, affinità, insight generator, priorità, daily digest | Insight puntuali e priorità vendita |
| **5** | Generazione comunicazioni (MVP) | RAG+LLM, validazioni, approval queue, delivery email, campagne, **visual generativo stand (§10.10)** | Primo invio reale email su segmento pilota (+ pilota visual opzionale) |
| **6** | WhatsApp Business API | Meta setup, template, delivery, webhook, dedupe | Flussi WhatsApp su whitelist |
| **7** | Prospect & lookalike | Seed list, web discovery, lookalike, affinità, score | Pipeline prospect con score e prime proposte |
| **8** | Ufficio Marketing IA | Agent orchestrator, Product Curator, autonomia L1→L3 | Ciclo plan→...→learn autonomo supervisionato |
| **9** | Ottimizzazione, scala, governance | Auto-invio, apprendimento, costi, compliance | Auto-invio L2/L3 entro KPI e guardrail |

Ogni fase termina con un **gate di accettazione** (criteri verificabili, test superati, metriche minime) prima di passare alla successiva. Dettagli operativi nel documento di roadmap.

---

## 16. Sicurezza, privacy e GDPR

### 16.1 Trattamento dati

- Finalità: profilazione commerciale e comunicazioni marketing (base giuridica: **legittimo interesse** per B2B con verifica bilanciamento / **consenso** dove richiesto).
- Consenso marketing tracciato in `consent_log` (canale, data, fonte), opt-out revocabile in qualsiasi momento e rispettato entro 48h.
- Categorie di dati: anagrafica, commerciale, web-pubblici. Niente dati sensibilissimi.
- Conservazione: dati di eligibilità per durata conforme; comunicazioni fino a revoca invio.

### 16.2 Sicurezza tecnica

- **Sola lettura assoluta sul gestionale** (`VINCOLO NON NEGOZIABILE`): DB replica read-only dedicata (mai il master produttivo) o connessione `pg` con utenza `decobrands_ro` a solo `SELECT`, `default_transaction_read_only = on`; **nessuna credenziale di Integra con permessi di scrittura** deve mai essere creata, memorizzata o usata.
- Verifica di read-only programmatica: **test di scrittura che deve fallire** alla partenza di ogni processo/worker/agente che tocca Integra (vedi `docs/roadmap-implementazione.md` Step 1.2.2).
- Segmentation: API con RBAC (venditore/ufficio/direzione/admin).
- Secrets in env/secret manager, mai nel codice.
- Audit log di ogni azione su comunicazioni e consensi.
- Data at rest encrypted; TLS in transito.
- Legame con integrazioni esterne (web search): URL sanitizzati, no dati anagrafici non necessari inviati a terze parti.

### 16.3 Accuratezza IA

- Grounding rigido nelle comunicazioni (no-hallucination sui fatti).
- Human-in-the-loop sull'invio.
- Possibilità di marcare insight/comunicazioni come errati per feedback.

---

## 17. Costi e infrastruttura

### 17.1 Voci di costo stimate

| Voce | Stima/range | Note |
|---|---|---|
| Infrastruttura servizi (Windows Server/S11, PostgreSQL 16, Redis, runtime Node) | Costo licenze/hardware esistenti; eventuali VM Windows cloud 50–200 €/mese | **Nessun Docker**: stack nativo Windows |
| LLM (generazione+estrazione+insight) | 100–600 €/mese | Dipende da volumi e modelli |
| Embedding | 10–50 €/mese | text-embedding-3-small, cache |
| Web search (Serper) | 30–100 €/mese | con cache e rate limit |
| Email (Resend/SES) | 5–50 €/mese | Volumi B2B contenuti |
| WhatsApp Business API | costo conversazione | Tariffe per conversazione, ~0.03–0.08 € |
| AI Image generativa (visual stand) | 0.05–0.30 €/immagine | Volume B2B contenuto (decine–centinaia/anno); rigenerazioni + upscaling inclusi nei costi |
| Totale indicativo | **200–1.000 €/mese scalabile** (+ immagine se usata) | Da validare in Fase 0/1 |

### 17.2 Ottimizzazioni

- Cache dei risultati LLM (hash del prompt) per job ripetitivi.
- Embedding con batch e cache.
- Modelli piccoli per task estrattivi, large solo quando necessario.
- Rate limit web search e uso intelligente dei budget giornalieri.

---

## 18. KPI e metriche di successo

### 18.1 Adozione/utilizzo

- copertura dati arricchiti (clienti con profilo web valido ≥ 80%);
- scheda 360° aperta ≥ N volte/settimana per venditore;
- % comunicazioni revisionate/modificate (human review utile);
- tempo medio di approvazione.

### 18.2 Performance commerciale

- reply rate / risposta a WhatsApp;
- open & click rate email (benchmark ≥ ~25% open, ≥ 3% click per segmento);
- ordini generati direttamente da comunicazione (attribuzione);
- NRR/espansione su clienti target di cross-sell;
- recovery di clienti inattivi (re-engagement);
- tasso conversione prospect → primo ordine;
- **delta conversione tra proposte con visual stand vs senza visual** (misura dell'incremento di risposta/ordine).

### 18.3 Qualità IA

- tasso di allucinazioni rilevate (target < 1% sulle comunicazioni approvate);
- falsi positivi insight (target < 5%);
- soddisfazione venditori (survey);
- **fedeltà prodotto nei visual stand** (similarità visiva output vs cutout, target ≥ 0.90);
- **tasso visual approvati senza rigenerazione** (target ≥ 80%).

---

## 19. Rischi e mitigazioni

| Rischio | Probabilità | Impatto | Mitigazione |
|---|---|---|---|
| Accesso DB Integra non concedibile | M | Alto | Verifica pre-progetto Fase 0; fallback export file (CSV/XML) schedulati |
| **Scrittura accidentale sul DB di Integra (violazione del vincolo read-only)** | **B** | **Critico** | Utenza a solo `SELECT` (mai credenziali scrivibili); connessioni con `default_transaction_read_only=on`; check di scrittura-negata all'avvio di ogni job/agente; replica read-only; audit; la regola è testata e parte del gate di ogni fase |
| Schema Integra non documentato | A | Medio | Reverse-engineering congiunto coi vendor; mapping in doc dedicato |
| Hallucination LLM | M | Alto | Grounding + validazioni; human approval obbligatoria |
| Frequenza invii percepita spam | M | Medio | Rate limit, feedback esiti, ottimizzazione segmenti |
| Dati "sporchi" dal gestionale (duplicati, codici errati) | A | Medio | Dedupe, pulizia master data, metriche di quality |
| Costo API fuori budget | M | Medio | Budget/cap giornalieri, cache, modello size tuning |
| Dipendenza da API serch/provider | B | Basso | Switch serch provider, modello LLM agnostic (interfaccia) |
| Conformità GDPR comunicazioni | M | Alto | Consenso rigoroso, opt-out, DPA, DPIA se richiesto |
| **Visual generativo: prodotto alterato o allestimento non realistico** | M | Medio | Cutout predefiniti (mai ricreati dal modello), compositing controllato, verifica di fedeltà via embedding visivo, rigenerazione limitata, fallback senza visual, revisione umana |
| **Visual generativo: uso di foto stand senza consenso o diritti** | B | Alto | Consenso cliente esplicito tracciato e revocabile; DPA provider; politiche di utilizzo immagini documentate |

---

## Appendice A — Esempi di comunicazioni generate

### A.1 Email — Proposta su novità (cross-sell) per cliente attivo

**Oggetto:** Una novità pensata per la tua clientela — linea "Superfici Lavabili"

**Preheader:** La linea che hai già scelto si arricchisce di 3 referenze in linea con il tuo mix attuale.

---

Gentile [Nome],

abbiamo visto che negli ultimi mesi la tua clientela ha scelto con soddisfazione la categoria *superfici lavabili* ([codice categoria]). Per questo vogliamo segnalarti tre novità che completano quella fascia:

1. **Pannello [DB-880]** — finitura opaca effetto pietra, ideale per vetrine e campioni.
2. **[DB-882]** — variante 3D per ambienti contract.
3. **[DB-884]** — pelle tattile con fondo in tessuto tecnico.

Le referenze sono disponibili da subito, con foto e campioni ordinabili. Il tuo listino attivo resta invariato; se ti interessa, posso farti avere i dati tecnici completi o organizzare una visita.

A presto,
[Nome venditore] — Decobrands
[tel] | [email]

---

### A.2 WhatsApp — Proposta riordino

> Ciao [Nome], passò da te per un rapido segnale: la linea [categoria] che ordini di solito a [periodo] è pronta per il riordino. Le referenze più richieste ([DB-208], [DB-210]) sono disponibili e il tuo listino è attivo.
> Se vuoi, preparo subito una proposta. Quando preferisci? — [Venditore], Decobrands

### A.3 Email — Re-engagement cliente inattivo

**Oggetto:** Da parte nostra, un pensiero per rimettersi in contatto

Gentile [Nome],

sono passati [N mesi] dall'ultimo ordine. In questo periodo abbiamo ampliato la linea [settore] e introdotto nuove finiture che stanno interessando molto showroom come il tuo.

Ti lascio qui un riepilogo delle novità [link catalogo]. Se preferisci, organizzo io una chiamata di 10 minuti per capire se c'è qualcosa che possa esserti utile oggi.

Un cordiale saluto,
[Nome venditore] — Decobrands

### A.4 WhatsApp — Prospect avvicinamento

> Salve [Nome], sono [Venditore] di Decobrands. Vediamo che trattate superfici decorative e arredo per il contract: la nostra linea [categoria] è molto scelta da showroom simili al vostro ([Città], [segmento]).
> Se ti va, ti mando una sintesi con foto e prezzi di listino dei 3 prodotti più affini alla vostra proposta. Hai 5 minuti questa settimana? — Distinti saluti

### A.5 Email — Proposta con visual stand (UC-07)

**Oggetto:** Ho allestito il tuo stand con le novità — guarda come può diventare

Gentile [Nome],

sulla base di come state esponendo oggi le superfici decorative, ti preparo un'anteprima: **il tuo stand [tipo espositore] allestito con le tre novità** della linea [categoria] ([DB-880], [DB-882], [DB-884]).

[IMMAGINE — visual generato: foto dello stand del cliente con i prodotti proposti]

Questa è solo una delle disposizioni possibili: posso preparartene altre 2 varianti (una più compatta per lo scaffale, una in versione stagionale) se preferisci.

Disponibilità immediata, stesso listino. Ti interessa che ti invii i dati tecnici e i campioni?

A presto,
[Nome venditore] — Decobrands

---

## Appendice B — Prompt engineering

### B.1 Prompt di generazione comunicazione (schema)

```
Sistema: Sei il copywriter commerciale di Decobrands, azienda B2B
che vende [Superfici/Arredo/Decor] a showroom e rivenditori.
Tono: consulenziale, tecnico ma accessibile, mai spammoso.
Regole:
- Usa SOLO i fatti nel contesto fornito.
- Non inventare prezzi, sconti, codici o dati anagrafici.
- Modi di dire: il cliente deve sentirsi supportato, mai inseguito.
- Invita a un'azione chiara e singola.
- Rispetta lunghezza per [canale]: email ≤ ~150 parole oggetto+preheader+corpo;
  WhatsApp ≤ ~900 caratteri.
- Non ripetere proposte già inviate negli ultimi [N] giorni (vedi contesto).

Contesto (dati veri):
- Profilo cliente: {customer_profile}
- Affinità prodotti: {product_affinity_top3}
- Listino attivo / condizioni: {price_list_snapshot}
- Storico comunicazioni recenti: {recent_communications}
- Template e brand voice: {template_guidelines}

Output: solo il testo della comunicazione (oggetto, preheader, corpo, CTA),
formato JSON. Nessun contenuto fuori schema.
```

### B.2 Prompt di estrazione web (schema)

```
Sistema: Estrai da questa pagina web i seguenti campi, per un operatore
economico B2B. Rispondi in JSON. Se un campo non è presente, usa "": null.
Campi: ragione_sociale, piva, settore, citta, anno_fondazione,
dimensione, mercati_serviti[], marchi_trattati[], segnali{},
confidenza (0-1), fonti[] (URL esatti).
Vincolo: nessuna inventiva; se non evidente, confidenza bassa.
```

### B.3 Prompt judge / rilevazione allucinazioni

```
Sistema: Sei un revisore. Data la comunicazione e i dati di origine,
elenca qualsiasi asserzione fattuale non supportata dai dati.
Output JSON: {ok: bool, issues: [{quote, fact, grounded}]}.
```

### B.4 Prompt di generazione visual stand (schema)

```
Sistema: Sei il Visual Designer di Decobrands. Devi allestire lo stand
del cliente con i prodotti reali proposti, mantenendo fedeltà assoluta
dei prodotti (design, colori, finiture come nei cutout forniti).

Input forniti:
- Foto dello stand vuoto: {stand_photo}
- Cutout prodotti (PNG sfondo trasparente): {product_cutouts: [{code, image}]}
- Vincoli: max {max_products} prodotti; disposizione {layout_style}
  (suggested: alternanza altezze, frontali, non sovrapposti)
- Tema/ambientazione: {theme}; formato output: {format_ratio}

Regole:
- Non modificare i prodotti: usa SOLO i cutout forniti.
- Non aggiungere prodotti, loghi, testi o elementi estranei non presenti
  nello stand o nei prodotti.
- Luci/ombre coerenti con lo stand reale; non far fluttuare i prodotti.
- Non ostruire ingressi/aree non espositive.

Output: immagine allestita + annotazioni di placement usate.
```

### B.5 Prompt judge / verifica fedeltà visual

```
Sistema: Confronta il prodotto nel visual generato (immagine) con il
cutout originale (immagine). Rispondi in JSON:
{product_code, ok: bool, fidelity_score: 0-1, note: string}.
ok=false se design/colore/finitura alterati o se il prodotto è irriconoscibile.
```

---

*Fine specifica v1.2.*