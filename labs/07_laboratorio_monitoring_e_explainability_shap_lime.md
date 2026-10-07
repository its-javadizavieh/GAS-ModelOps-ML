# Lab 07 - Laboratorio Monitoring e Explainability SHAP LIME

## Obiettivo

- Partire da un segnale di monitoring reale e produrre una spiegazione SHAP o LIME che ne chiarisca la causa.
- Collegare il risultato pratico ai micro-argomenti: Laboratorio pratico con MLflow, Grafana, Prometheus; Tecniche SHAP/LIME, casi reali e impatti sul business.

## Durata (timebox)

- 30 minuti.
- 5 minuti setup e lettura scenario.
- 20 minuti esecuzione guidata.
- 5 minuti checkpoint, cleanup e consegna evidenze.

## Prerequisiti

- MLflow, Grafana/Prometheus (o simulazione locale), SHAP, LIME, scikit-learn.
- Conoscenza base di Python, terminale, Git e ciclo di vita del modello, e familiarita con il lab di monitoring della sessione precedente.
- Fallback previsto: dashboard simulata con tabella Markdown e spiegazione SHAP/LIME su un dataset ridotto se mancano risorse cloud.

## Scenario

- La dashboard di monitoring (collegata al lab della sessione 06) segnala che l'accuratezza del modello di scoring crediti e scesa sotto soglia per il batch di questa settimana.
- Prima di decidere un rollback, il team vuole capire QUALE input sta causando l'errore, non solo CHE l'errore esiste.
- Si isola un campione di predizioni sbagliate del batch recente e si genera una spiegazione locale con SHAP o LIME per capire quale feature guida l'errore.
- Il risultato deve collegare esplicitamente il segnale numerico del monitoring alla spiegazione qualitativa prodotta, in un formato presentabile a un responsabile business.

## Step (numerati)

1. Recupera (o simula) il segnale di monitoring: metrica degradata, timestamp e batch di predizioni coinvolto.
2. Seleziona 2-3 predizioni sbagliate o sospette dal batch segnalato come casi da spiegare.
3. Genera una spiegazione SHAP (`shap.Explainer` + `shap.plots.waterfall`) per almeno un caso, mostrando il contributo delle feature.
4. Genera una spiegazione LIME per lo stesso caso o per un secondo caso, e confronta le due spiegazioni.
5. Collega la feature piu influente individuata al possibile drift rilevato dal monitoring (es. la feature che ha causato l'errore e la stessa che mostra drift).
6. Scrivi una nota di impatto business (a chi comunicare, quale azione proporre) e poi esegui il cleanup delle risorse usate.

## Output atteso

- Il grafico SHAP (waterfall o summary) e/o l'output LIME per almeno un caso spiegato.
- Una tabella che collega segnale di monitoring -> caso selezionato -> feature influente -> spiegazione.
- Una nota di impatto business con destinatario e azione proposta.

## Checkpoint

- La spiegazione SHAP/LIME e collegata esplicitamente al segnale di monitoring che l'ha resa necessaria, non generata a caso.
- E chiaro quale predizione e stata spiegata e con quale input.
- La nota di impatto business e comprensibile anche a chi non conosce SHAP/LIME.

## Troubleshooting rapido

- Se `shap.Explainer` e troppo lento sul modello usato, riduci il campione (`X_sample`) a poche righe o usa un explainer piu leggero (es. `TreeExplainer` per modelli ad albero).
- Se LIME restituisce spiegazioni instabili tra due run, aumenta il numero di campioni perturbati (`num_samples`) o fissa il seed.
- Se il segnale di monitoring reale non e disponibile, simula un batch con una feature spostata intenzionalmente (es. shift artificiale su una colonna numerica).
- Se SHAP e LIME indicano feature diverse come piu influenti, documenta la discrepanza invece di sceglierne una a caso.

## Cleanup obbligatorio

- Ferma kernel notebook, processi di calcolo SHAP/LIME e eventuali server di inferenza locali avviati per il test.
- Rimuovi grafici temporanei, file `.html` di LIME e cache degli explainer non necessari alla consegna.
- Se hai usato una dashboard Grafana/Prometheus di supporto, chiudila o rimuovi le regole di alert create per il test.
- Conferma nel deliverable che il cleanup e stato completato o indica cosa non e stato possibile rimuovere.
