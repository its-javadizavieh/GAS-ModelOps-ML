# Soggetti per il project work finale di gruppo

Il [Lab 13](../13_project_work_pipeline_serving_monitoring_e_aggiornamento.md) è la guida comune per costruire il progetto finale. Ogni gruppo sceglie un soggetto e lavora sulla relativa cartella.

Tutti i materiali dei soggetti sono raccolti qui: otto dataset sintetici. Il gruppo realizza il proprio codice, documenta le scelte e presenta le evidenze del percorso ModelOps.

| N.  | Soggetto e scheda                                                                     | Provenienza        | Righe train / validation | target = 1                          |
| --- | ------------------------------------------------------------------------------------- | ------------------ | ------------------------ | ----------------------------------- |
| 1   | [Abbandono clienti](01_churn_clienti/README.md)                                       | Sintetica          | 800 / 200                | Abbandono entro 30 giorni           |
| 2   | [Manutenzione macchinari](02_manutenzione_macchinari/README.md)                       | Sintetica          | 800 / 200                | Guasto entro 7 giorni               |
| 3   | [Ritardi consegne](03_ritardi_consegne/README.md)                                     | Sintetica          | 800 / 200                | Consegna dopo la scadenza           |
| 4   | [Resi e-commerce](04_resi_ecommerce/README.md)                                        | Sintetica          | 800 / 200                | Reso entro 30 giorni dalla consegna |
| 5   | [Anomalie energetiche](05_anomalie_energetiche/README.md)                             | Sintetica          | 800 / 200                | Anomalia nelle prossime 24 ore      |
| 6   | [Cancellazioni alberghiere](06_cancellazioni_prenotazioni/README.md)                  | Sintetica          | 800 / 200                | Cancellazione prima dell'arrivo     |
| 7 | [Escalation dei ticket IT](07_escalation_ticket_it/README.md) | Sintetica | 800 / 200 | Escalation al secondo livello entro 24 ore |
| 8 | [Esaurimento scorte](08_esaurimento_scorte/README.md) | Sintetica | 800 / 200 | Esaurimento entro 7 giorni |

## Presentazione del 12 novembre 2026

La lezione di **giovedì 12 novembre 2026, dalle 12:00 alle 18:00**, è dedicata alla presentazione dei project work.

- **12:00–13:00:** al massimo un'ora per completare gli ultimi dettagli e preparare la demo.
- **Dalle 13:00:** presentazioni dei gruppi, all'interno della lezione che termina alle 18:00.
- **Entro le 13:00:** progetto, documentazione e presentazione devono essere pronti per essere mostrati e consegnati al docente.

Arrivate con il lavoro principale già svolto. L'ora iniziale serve a rifinire il progetto e controllare i materiali; organizzate lo sviluppo prima di questa data. Ogni componente presenta il proprio contributo. L'ordine e i tempi dei singoli gruppi saranno organizzati dal docente.

## Materiali per ciascun gruppo

Condividere il **Lab 13** e la **cartella del soggetto assegnato**, che contiene:

- README.md: scopo, classi, provenienza, dizionario e split.
- dataset.csv: dati assegnati per training e valutazione.
- richieste_monitoraggio.csv: payload coerenti con lo schema del soggetto.

Il gruppo sceglie e implementa i componenti guidato dai checkpoint del lab. La consegna finale comprende il codice realizzato dagli studenti e le evidenze osservate.

## Contratto condiviso

- 30 feature numeriche in ordine di intestazione, record_id univoco, split e target 0/1.
- Numero di righe e significato delle classi descritti nella scheda del soggetto.
- Baseline e candidata sullo stesso CSV, stesso schema e stessi ID di validation.
- Richieste distribuite: tre casi validi con versioni v1 → v2 → v1, poi un input incompleto con risposta attesa 422.
- Error rate atteso sulle quattro richieste del drill: 0,25.

Usate esclusivamente la coppia CSV della cartella scelta. Tutti i soggetti seguono le stesse fasi di progetto; risultati e interpretazione delle metriche dipendono dai dati e dalla codifica delle classi. Non è garantito che la candidata superi il gate.

La verifica finale comprende quiz individuale dai mini-esercizi delle slide e presentazione del project work di gruppo, senza esame orale separato.
