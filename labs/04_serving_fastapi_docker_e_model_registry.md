# Lab 04 - Serving FastAPI Docker e Model Registry

## Obiettivo

- Costruire un endpoint FastAPI `/predict` che serve un modello e restituisce anche la versione del modello caricato, con un registry minimale (anche solo file/tabella) per tracciare le versioni.
- Collegare il risultato pratico ai micro-argomenti: Demo pratica deploy e serving modello con FastAPI+Docker; Cos'e il model registry, versionare modelli/dati.

## Durata (timebox)

- 30 minuti.
- 5 minuti setup e lettura scenario.
- 20 minuti esecuzione guidata.
- 5 minuti checkpoint, cleanup e consegna evidenze.

## Prerequisiti

- FastAPI, Docker (o solo CLI/documentazione), MLflow (facoltativo, anche simulato), Git.
- Conoscenza base di Python, terminale, Git e ciclo di vita del modello.
- Fallback previsto: usare scheda di simulazione locale, tabelle Markdown o screenshot della documentazione quando tool, credenziali o permessi non sono disponibili.

## Scenario

- Il team ha allenato `model_v2` che supera la metrica minima (accuracy >= 0.85) e vuole servirlo con un endpoint FastAPI, ma la settimana scorsa nessuno sapeva dire quale versione del modello rispondeva alle richieste in produzione.
- Il tuo gruppo deve costruire `/predict` in modo che ogni risposta includa esplicitamente la versione del modello caricato, e un registry minimale (anche solo un file `registry.md` o una tabella) che elenchi le versioni disponibili con relative metriche.
- Quando arriva `model_v3` che non supera la soglia, il team deve saper documentare la decisione di NON promuoverlo, mantenendo `model_v2` attivo.

## Step (numerati)

1. Crea (o simula) due modelli `model_v2.pkl` e `model_v3.pkl`, ciascuno con una metrica accuracy associata (es. 0.87 e 0.79).
2. Scrivi un registry minimale (tabella o file `registry.md`) con colonne: versione, path modello, metrica, stato (candidato/promosso/scartato).
3. Costruisci l'endpoint FastAPI `/predict` che carica `model_v2` all'avvio e restituisce nella risposta sia la predizione sia il campo `model_version`.
4. Testa l'endpoint con una richiesta valida (`curl` o client HTTP) e verifica che la risposta contenga `model_version: v2`.
5. Applica il gate di promozione a `model_v3`: confronta la sua metrica con la soglia, decidi se promuoverlo e aggiorna il registry con lo stato risultante.
6. Esegui il cleanup e annota nel deliverable cosa hai rimosso o lasciato come placeholder.

## Output atteso

- Il registry aggiornato con almeno due versioni modello e relativo stato.
- L'endpoint `/predict` funzionante (o descritto in dettaglio) che espone `model_version` nella risposta.
- La decisione documentata su `model_v3` (promosso o scartato) con motivazione basata sulla metrica.

## Checkpoint

- La risposta di `/predict` permette di identificare senza ambiguita quale versione del modello ha prodotto la predizione.
- Il registry distingue chiaramente stato promosso da stato scartato/candidato, con la metrica che giustifica la decisione.
- Il gruppo sa spiegare cosa farebbe per un rollback se `model_v2` iniziasse a comportarsi male in produzione.

## Troubleshooting rapido

- Se l'endpoint non avvia, verifica che il path del modello caricato all'avvio (`model_v2.pkl`) esista e sia leggibile.
- Se la risposta non include `model_version`, controlla che il campo venga aggiunto esplicitamente nel corpo della risposta e non solo nei log.
- Se MLflow non e disponibile, sostituiscilo con un registry simulato in Markdown senza bloccare il lab.
- Se la porta 8000 risulta occupata da un run precedente, identifica il processo e terminalo prima di ripartire con `uvicorn`.

## Cleanup obbligatorio

- Ferma il processo `uvicorn` (o il container FastAPI) lasciato in esecuzione dopo i test dell'endpoint.
- Rimuovi i file `model_v2.pkl` / `model_v3.pkl` di prova se non fanno parte del deliverable consegnato.
- Se hai usato MLflow locale, elimina le run di test dal tracking store o annota che sono solo dimostrative.
- Conferma nel deliverable che il cleanup e stato completato o indica cosa non e stato possibile rimuovere.
