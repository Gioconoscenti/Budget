# Contesto Progetto — bilancio-personale.html

## Cos'è

Web-app finanza personale, single-file HTML (≈3650 righe), no framework, no backend.

**Stack:** HTML + CSS + JS vanilla + Chart.js
**Dati:** via CSV importato

## Funzionalità implementate

- Dashboard panoramica con KPI (entrate, uscite, saldo netto, risparmio %, media mensile)
- Filtri globali: periodo (Tutti / Questo mese / Mese scorso / Quest'anno / Anno select / Multi-mese a pannello), acquirente (Tutti / Gioacchino / Alessandra), conto (select)
- Conto Economico trimestrale/annuale, solo conto ordinario, con filtro acquirente. Espansione a 3 livelli: sezione → tipologia → singole transazioni
- Sezione Transazioni: tabella paginata, ordinamento colonne, resize colonne, filtri tipo/categoria/conto/ricerca testo
- Pannello AI (Gemini) in Panoramica: chip rapidi + input libero, contesto finanziario completo passato all'API
- Sezione Investimenti, Sezione Banche, Import CSV, Regole di categorizzazione

## Gemini API

- Chiave: `<INSERIRE_CHIAVE_IN_VARIABILE_D_AMBIENTE_O_SECRET>`
- Auto-discovery modello: prova in sequenza `gemini-2.0-flash` → `gemini-2.0-flash-lite` → `gemini-1.5-flash` → `gemini-1.5-flash-8b` → `gemini-pro`
- Modello trovato cachato in `window._geminiModel`

> ⚠️ **Nota di sicurezza:** non ho inserito la chiave API reale in questo file. Il repository `Gioconoscenti/Budget` è **pubblico**, quindi qualsiasi chiave committata in chiaro (anche in un file `.md` o direttamente nell'HTML) è visibile a chiunque e può essere raccolta da bot che scansionano GitHub in pochi minuti. Se la chiave che avevi condiviso è già stata committata nel codice sorgente, ti consiglio di:
> 1. Revocarla/rigenerarla subito da Google AI Studio
> 2. Spostarla fuori dal codice (es. richiesta lato server, o inserimento a runtime dall'utente in un campo non salvato)
> 3. Se necessario, ripulire la history del repo (es. `git filter-repo`) dato che rimane visibile nei commit passati anche se la rimuovi ora

## Variabili di stato chiave

| Variabile | Descrizione |
|---|---|
| `period` | preset periodo (`'thismonth'` / `'lastmonth'` / `'thisyear'` / `''`) |
| `yearF` | anno selezionato dal select |
| `selectedMonths` | `Set()` con numeri mese 1-12 (multi-selezione) |
| `ceAgg` | `'trim'` o `'anno'` (aggregazione conto economico) |
| `ceAcquirente` | `''` / `'gioacchino'` / `'alessandra'` |
| `ceExpandState` | oggetto con stato espansione righe CE |
| `contoG`, `acquirenteG` | filtri globali |
| `allTx`, `filteredTx` | array transazioni grezze e filtrate |

## Funzioni chiave

| Funzione | Descrizione |
|---|---|
| `applyFilters()` | applica tutti i filtri e chiama `runSection()` |
| `setPeriod(btn, p)` | cambia tab periodo, resetta mesi |
| `setYear(y)` | seleziona anno, resetta mesi |
| `applyMonths()` | applica selezione mesi senza toccare period/yearF |
| `_resetMonths()` | svuota `selectedMonths` e aggiorna UI |
| `renderConto()` | ridisegna tabella conto economico |
| `toggleCERow(id)` | espande/collassa riga CE |
| `setCEAcquirente(val)` | filtro acquirente nel CE |
| `askAI(question)` | chiama Gemini con contesto finanziario |
| `buildFinancialContext()` | costruisce il testo-contesto per Gemini |
| `fillDrops()` | popola i select dinamici (conti, categorie, anni) |

## Repository

[github.com/Gioconoscenti/Budget](https://github.com/Gioconoscenti/Budget)

## Bug risolti nella sessione

- `mo-sel` null reference (select mese rimosso, sostituito con pannello multi-mese)
- Override font nav laterale in media query (`nav .nav-btn { font-size: 7.5px }`)
- `selectedMonths` era nel ramo `else` di `applyFilters` → ignorato con tab preset attivo
- `applyMonths()` azzerava `yearF` → Anno + Mesi incompatibili
- `fillDrops()` cercava ancora `g-acq-sel` rimosso
