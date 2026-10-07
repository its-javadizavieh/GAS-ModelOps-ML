# Lab 06 - Monitoring Logging Drift e Notifiche

## Obiettivo

- Definire soglie di drift e una regola di notifica automatica collegata a un log leggibile.
- Collegare il risultato pratico ai micro-argomenti: Monitoraggio metriche e prestazioni; Gestione model/data drift e notifiche automatiche; Laboratorio pratico con MLflow, Grafana, Prometheus.

## Durata (timebox)

- 30 minuti.
- 5 minuti setup e lettura scenario.
- 20 minuti esecuzione guidata.
- 5 minuti checkpoint, cleanup e consegna evidenze.

## Prerequisiti

- MLflow (tracking metriche), Prometheus (raccolta metriche), Grafana (dashboard/alert), notebook o script Python.
- Conoscenza base di Python, terminale, Git e ciclo di vita del modello.
- Fallback previsto: simulare drift_score e soglie con uno script Python e una tabella Markdown se Prometheus/Grafana non sono disponibili.

## Scenario

- Un modello di classificazione clienti e in produzione da un mese; il team riceve solo log grezzi, senza soglie ne notifiche.
- Negli ultimi giorni la distribuzione di una feature chiave (eta media dei clienti) si e spostata rispetto al training set.
- Il team deve costruire un controllo minimo: calcolare un indicatore di drift, fissare una soglia, collegarla a una notifica con causa e azione proposta.
- L'obiettivo e evitare sia il rumore (troppi falsi allarmi) sia il silenzio (drift reale non rilevato).

## Step (numerati)

1. Calcola un indicatore di drift (es. distanza tra distribuzioni, PSI o una differenza di media semplice) tra dati di training e batch recente.
2. Fissa una soglia motivata da dati storici (es. `drift_score > 0.08`) e documenta perche quel valore e stato scelto.
3. Implementa la logica soglia -> notifica: se il drift supera la soglia invia un alert, altrimenti registra un log "sotto soglia".
4. Simula l'esposizione della metrica a Prometheus (o una tabella locale) e una dashboard Grafana (o uno screenshot/mockup) con la serie temporale del drift.
5. Genera un alert di prova e documenta canale, responsabile e azione correttiva proposta.
6. Esegui cleanup di dashboard, scrape config e processi di test, poi annota cosa e stato rimosso.

## Output atteso

- Uno script o notebook che calcola il drift score e applica la soglia definita.
- Una tabella o screenshot della dashboard con la serie temporale della metrica monitorata.
- Un log di notifica con timestamp, causa (soglia superata) e azione correttiva proposta.

## Checkpoint

- La soglia di drift e motivata con dati storici, non scelta arbitrariamente.
- L'alert generato e collegato a un canale e a un responsabile che lo riceve.
- Il log distingue chiaramente i casi "sotto soglia" (nessuna azione) da quelli "sopra soglia" (notifica inviata).

## Troubleshooting rapido

- Se Prometheus non e installato, simula lo scrape con un dizionario Python che accumula valori nel tempo e stampali come serie.
- Se Grafana non e raggiungibile, sostituisci la dashboard con un grafico Matplotlib o una tabella Markdown con timestamp e valore.
- Se il drift_score resta sempre a zero, controlla di non confrontare due volte lo stesso batch di dati recenti.
- Se l'alert genera troppo rumore, alza la soglia o aggiungi un requisito di persistenza (es. 3 batch consecutivi sopra soglia).

## Cleanup obbligatorio

- Ferma il processo Prometheus/exporter locale e il container Grafana se avviati.
- Rimuovi dashboard di test, regole di alert temporanee e file di configurazione scrape non necessari.
- Elimina script o notebook con dati di batch simulati non richiesti nella consegna finale.
- Conferma nel deliverable che il cleanup e stato completato o indica cosa non e stato possibile rimuovere.
