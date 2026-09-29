# Lab 05 - Versioning Rollback e Prime Metriche

## Obiettivo

- Simulare un rollback motivato da metriche di produzione, non da una sensazione.
- Collegare il risultato pratico ai micro-argomenti: Cos'e il model registry, versionare modelli/dati; Simulazione rollback dopo errore in produzione.

## Durata (timebox)

- 30 minuti.
- 5 minuti setup e lettura scenario.
- 20 minuti esecuzione guidata.
- 5 minuti checkpoint, cleanup e consegna evidenze.

## Prerequisiti

- MLflow Model Registry (o simulazione locale), Git, script `evaluate.py`, tabella metriche.
- Conoscenza base di Python, terminale, Git e ciclo di vita del modello.
- Fallback previsto: usare tag Git locali e una tabella Markdown al posto del registry se MLflow non e disponibile.

## Scenario

- Il modello di scoring `model-v1` e in produzione da due settimane con accuratezza stabile intorno all'85%.
- Il team rilascia `model-v2`, riaddestrato su dati piu recenti, senza accorgersi che una feature e stata calcolata in modo diverso in fase di training.
- Dopo il rilascio, il monitoraggio segnala un calo di accuratezza e un aumento degli errori 5xx sul servizio di inferenza.
- Il team ha 30 minuti per decidere se fare rollback a `model-v1` o tentare un fix rapido, e deve lasciare traccia scritta della decisione.

## Step (numerati)

1. Registra la metrica baseline di `model-v1` (accuratezza, latenza media) in una tabella o con `mlflow.log_metric`.
2. Simula il deploy di `model-v2` e registra le sue metriche osservate: la regressione deve essere visibile nel confronto v1/v2.
3. Definisci in anticipo una soglia di accettazione (es. calo di accuratezza > 3 punti percentuali) e verifica se `model-v2` la supera.
4. Decidi tra rollback immediato e fix rapido: motiva la scelta con la soglia superata, il tempo stimato di fix e il rischio di lasciare v2 attivo.
5. Esegui il rollback (`git tag`/checkout alla versione precedente o cambio di stage nel registry) e verifica che le metriche tornino ai valori di v1.
6. Documenta la decisione in un log: motivo, soglia superata, versione di partenza e versione di arrivo, poi esegui il cleanup.

## Output atteso

- Una tabella o notebook con metriche v1 vs v2 a confronto.
- Un log di rollback con motivo, soglia superata e versione ripristinata.
- Nota finale con limiti, fallback usato e cleanup eseguito.

## Checkpoint

- Le metriche di v1 e v2 sono confrontate sulle stesse condizioni (stesso batch di input).
- La decisione (rollback o fix) e motivata da una soglia definita prima dell'incidente, non a posteriori.
- Il log di rollback riporta versione di partenza, versione di arrivo e motivo in modo leggibile da terzi.

## Troubleshooting rapido

- Se MLflow Model Registry non e raggiungibile, simula gli stage (`Staging`/`Production`/`Archived`) con una tabella Markdown e tag Git.
- Se `evaluate.py` non produce output confrontabile, fissa lo stesso set di dati di validazione per v1 e v2 prima di ricalcolare.
- Se il rollback via `git checkout` genera conflitti, isola il modello in un branch dedicato e ripristina solo l'artifact, non l'intero commit.
- Se le metriche v1/v2 sembrano indistinguibili, verifica di non star confrontando due run sullo stesso batch dati per errore.

## Cleanup obbligatorio

- Ferma eventuali server di inferenza locali avviati per servire `model-v1`/`model-v2`.
- Rimuovi tag Git di test, branch temporanei e artifact `.pkl` non necessari alla consegna.
- Se hai usato un MLflow Tracking Server locale, chiudi il processo e rimuovi la cartella `mlruns` di prova.
- Conferma nel deliverable che il cleanup e stato completato o indica cosa non e stato possibile rimuovere.
