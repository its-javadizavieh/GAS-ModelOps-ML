# Lab 01 - Ciclo ModelOps e Pipeline Dati Modello Deploy

## Obiettivo

- Mappare il ciclo ModelOps end-to-end (dati -> modello -> deploy) e indicare il primo gate di automazione CI/CD.
- Collegare il risultato pratico ai micro-argomenti: Ciclo completo Dati -> Modello -> Deploy; Principi CI/CD ed automazione per ML.

## Durata (timebox)

- 30 minuti.
- 5 minuti setup e lettura scenario.
- 20 minuti esecuzione guidata.
- 5 minuti checkpoint, cleanup e consegna evidenze.

## Prerequisiti

- Python, notebook o IDE, Git, terminale.
- Conoscenza base di Python, terminale, Git e ciclo di vita del modello.
- Fallback previsto: usare scheda di simulazione locale, tabelle Markdown o screenshot della documentazione quando tool, credenziali o permessi non sono disponibili.

## Scenario

- Un collega ha allenato un classificatore ieri ottenendo accuracy 0.89, ma oggi rilanciando lo stesso comando ottiene 0.81 e nessuno sa dire se e cambiato il dataset, i parametri o l'ambiente.
- Il tuo gruppo deve costruire una mappa minima del ciclo dati -> modello -> deploy con un identificativo per ogni run, cosi il problema diventa diagnosticabile invece che un mistero.
- Alla fine del lab il team deve anche proporre il primo gate CI/CD: una condizione automatica che impedirebbe di "promuovere" un modello peggiore di quello attuale.

## Step (numerati)

1. Crea un file `data.csv` fittizio (o usa un dataset tabellare semplice) e uno script `train.py --data data.csv --out model.pkl` che stampi almeno una metrica (es. accuracy) a fine training.
2. Esegui `train.py` due volte cambiando un solo parametro (es. random seed o una colonna esclusa) e annota le due metriche ottenute: questo dimostra la non ripetibilita se non tracci input e configurazione.
3. Costruisci una tabella "run log" con colonne: id run, dataset usato, parametri, metrica ottenuta, path del modello salvato.
4. Definisci per iscritto un gate CI/CD minimo, es. "il modello viene promosso solo se accuracy >= soglia del run precedente e il file `model.pkl` esiste".
5. Simula l'esecuzione del gate a mano sulle due run del punto 2: dichiara quale run avrebbe superato il gate e perche.
6. Esegui il cleanup e annota nel deliverable cosa hai rimosso o lasciato come placeholder.

## Output atteso

- Un run log (tabella) con almeno due run confrontabili tra loro.
- La definizione scritta del primo gate CI/CD e l'esito della sua applicazione simulata.
- Nota finale con limiti, fallback usato e cleanup eseguito.

## Checkpoint

- Il run log permette di capire, senza rileggere codice, quale run ha prodotto quale metrica.
- Il gate CI/CD proposto e verificabile (una condizione booleana chiara), non generico ("controllare che vada bene").
- Il gruppo sa spiegare cosa manca oggi per rendere il ciclo davvero riproducibile (es. versione dataset, seed, ambiente).

## Troubleshooting rapido

- Se `train.py` non produce output stabile tra due run identiche, verifica che il seed casuale sia fissato esplicitamente nel codice.
- Se manca un dataset reale, genera un CSV sintetico con poche righe (es. `pandas` + valori casuali) invece di bloccare il lab.
- Se il file `model.pkl` non viene scritto, controlla i permessi della cartella di output prima di sospettare un bug nello script.
- Se due run identiche danno metriche diverse senza motivo apparente, sospetta una libreria che introduce non-determinismo (es. split train/test senza seed).

## Cleanup obbligatorio

- Rimuovi i file `model.pkl` generati durante le prove e i CSV temporanei creati per il test.
- Cancella eventuali notebook kernel o processi Python lasciati attivi dopo le esecuzioni ripetute di `train.py`.
- Elimina cronologie di run o log intermedi che non fanno parte del deliverable finale.
- Conferma nel deliverable che il cleanup e stato completato o indica cosa non e stato possibile rimuovere.
