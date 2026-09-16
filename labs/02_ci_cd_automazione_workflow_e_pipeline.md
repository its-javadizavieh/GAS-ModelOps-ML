# Lab 02 - CI CD Automazione Workflow e Pipeline

## Obiettivo

- Disegnare una pipeline ML a stadi (test, dati, training, soglia metrica, promozione) con controlli automatici verificabili.
- Collegare il risultato pratico ai micro-argomenti: Principi CI/CD ed automazione per ML; Esercizi su workflow e pipeline.

## Durata (timebox)

- 30 minuti.
- 5 minuti setup e lettura scenario.
- 20 minuti esecuzione guidata.
- 5 minuti checkpoint, cleanup e consegna evidenze.

## Prerequisiti

- Git, terminale, notebook o IDE, tabella Markdown.
- Conoscenza base di Python, terminale, Git e ciclo di vita del modello.
- Fallback previsto: usare scheda di simulazione locale, tabelle Markdown o screenshot della documentazione quando tool, credenziali o permessi non sono disponibili.

## Scenario

- Il repository del team ha uno script `pipeline.py` che allena un modello, ma finora chiunque lo lancia a mano senza controlli: un data scientist ha appena "promosso" un modello che falliva silenziosamente su input vuoti.
- Il tuo gruppo deve trasformare il lancio manuale in una sequenza di stadi con un test automatico (`pytest`), una validazione minima dei dati, il training e un gate sulla metrica prima di consegnare l'artefatto.
- Ogni stadio deve poter fallire in modo comprensibile: se lo stadio 2 fallisce, lo stadio 3 non deve partire.

## Step (numerati)

1. Scrivi un test minimo con `pytest` che verifichi che il dataset `data.csv` abbia le colonne attese e nessun valore nullo nella colonna target.
2. Scrivi (o simula in pseudo-codice) `pipeline.py --stage validate` che esegue il test del punto 1 e si interrompe con un messaggio chiaro se fallisce.
3. Aggiungi lo stadio `--stage train` che allena il modello solo se lo stadio precedente e passato, e stampa la metrica ottenuta.
4. Aggiungi lo stadio `--stage gate` che confronta la metrica con una soglia fissa (es. accuracy >= 0.75) e decide se promuovere l'artefatto.
5. Esegui la sequenza completa `pytest tests/ && python pipeline.py --stage train` e poi il gate, annotando l'esito di ciascuno stadio in un log.
6. Simula un fallimento intenzionale (es. rimuovi una colonna dal CSV) e verifica che lo stadio di validazione blocchi la pipeline prima del training.

## Output atteso

- Un log degli stadi eseguiti con esito (pass/fail) per ciascuno.
- Il codice o pseudo-codice dei 3 stadi (validate, train, gate) con l'ordine di dipendenza esplicito.
- Nota finale con limiti, fallback usato e cleanup eseguito.

## Checkpoint

- La pipeline si ferma effettivamente quando uno stadio fallisce, invece di proseguire con dati o modello non validati.
- Il gate sulla metrica usa una soglia numerica esplicita, non un giudizio informale.
- Il gruppo sa indicare quale stadio aggiungerebbe per primo in un progetto reale (es. controllo drift, test su modello serializzato).

## Troubleshooting rapido

- Se `pytest` non trova i test, verifica il nome del file (deve iniziare con `test_`) e la cartella indicata nel comando.
- Se lo stadio di validazione dati non blocca nulla, controlla che il codice ritorni un exit code diverso da zero in caso di fallimento.
- Se manca un ambiente CI reale (GitHub Actions, ecc.), simula la sequenza eseguendo gli stadi in ordine manualmente e documentando l'output.
- Se la soglia del gate sembra arbitraria, giustificala con un confronto rispetto a un baseline gia noto (es. modello attuale in produzione).

## Cleanup obbligatorio

- Rimuovi i file di test temporanei, i dataset modificati per simulare il fallimento e i modelli generati durante le prove.
- Ferma eventuali processi Python o watcher lasciati attivi dopo l'esecuzione ripetuta della pipeline.
- Elimina log intermedi degli stadi che non fanno parte del deliverable finale.
- Conferma nel deliverable che il cleanup e stato completato o indica cosa non e stato possibile rimuovere.
