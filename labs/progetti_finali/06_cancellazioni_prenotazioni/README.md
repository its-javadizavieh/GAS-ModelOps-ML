# Progetto 6 - Cancellazioni di prenotazioni alberghiere

## Domanda e momento della previsione

**Domanda:** Quali prenotazioni potrebbero essere cancellate prima dell’arrivo?

**Momento di osservazione:** Conferma della prenotazione; lo storico riguarda soggiorni e prenotazioni precedenti.

- **target = 1:** Prenotazione cancellata prima dell’arrivo.
- **target = 0:** Prenotazione non cancellata prima dell’arrivo.

Questi dati sono **interamente sintetici**, creati per il corso. Non rappresentano persone, aziende, macchine, edifici o prenotazioni reali. Le unità e gli intervalli descrivono la simulazione didattica. I risultati non misurano le prestazioni su un dominio reale.

## File da usare insieme

| File | Contenuto |
| --- | --- |
| [dataset.csv](dataset.csv) | 1000 righe, 30 feature numeriche, record_id univoco, split e target binario |
| [richieste_monitoraggio.csv](richieste_monitoraggio.csv) | Quattro richieste coerenti con queste feature: v1 valida, v2 valida, v1 valida e v1 con 29 feature → 422 |

Ricevi anche il [Lab 13](../../13_project_work_pipeline_serving_monitoring_e_aggiornamento.md). Segui le fasi del lab usando i due CSV di questa cartella. Conserva la stessa coppia, lo stesso schema e lo stesso ordine delle feature durante tutto il progetto.

## Split e classi

| Split | target 0 | target 1 | Totale |
| --- | ---: | ---: | ---: |
| train | 463 | 337 | 800 |
| validation | 116 | 84 | 200 |

Lo split stratificato è già distribuito. Usalo per v1 e v2; non rigenerarlo. Record_id, split e target non entrano nelle feature. Non usare la validation per allenare lo scaler o il modello. Il CSV di richieste contiene solo input operativi: non è un altro dataset di training.

## Dizionario delle 30 feature

Il nome indica la misura e, dove previsto, la finestra temporale; `pct` indica percentuali: le quote sono tra 0 e 100, mentre gli scostamenti e le variazioni possono essere negativi secondo gli intervalli della tabella. I punteggi sono indici sintetici nella scala riportata. Per righe con unità «conteggio» i valori sono interi. Tutte le feature sono disponibili al momento della previsione descritto sopra.

**L'ordine seguente è l'ordine della lista features nell'API.** I payload distribuiti rispettano questo ordine; non contengono target, record_id o split.

| Posizione API | Colonna | Unità / scala | Intervallo della simulazione |
| --- | --- | --- | --- |
| 1 | `anticipo_prenotazione_giorni` | giorni | 1–240 |
| 2 | `giorni_finestra_cancellazione_gratuita` | giorni | 0–90 |
| 3 | `notti_prenotate` | notti | 1–20 |
| 4 | `camere_prenotate` | conteggio | 1–8 |
| 5 | `numero_ospiti` | conteggio | 1–12 |
| 6 | `incertezza_itinerario_indice` | punteggio 0–100 | 0–100 |
| 7 | `deposito_pct` | % | 0–100 |
| 8 | `quota_non_rimborsabile_pct` | % | 0–100 |
| 9 | `garanzia_pagamento_indice` | punteggio 0–100 | 0–100 |
| 10 | `prezzo_notte_eur` | EUR | 30–400 |
| 11 | `scarto_prezzo_da_media_zona_pct` | % | -40–70 |
| 12 | `sconto_fedelta_pct` | % | 0–40 |
| 13 | `prenotazioni_precedenti_cliente` | conteggio | 0–30 |
| 14 | `cancellazioni_cliente_precedenti` | conteggio | 0–12 |
| 15 | `quota_cancellazioni_cliente_storica_pct` | % | 0–100 |
| 16 | `soggiorni_completati_precedenti` | conteggio | 0–30 |
| 17 | `mesi_anzianita_cliente` | mesi | 0–96 |
| 18 | `punteggio_fedelta_cliente` | punteggio 0–100 | 0–100 |
| 19 | `modifiche_durante_prenotazione` | conteggio | 0–10 |
| 20 | `minuti_compilazione_prenotazione` | minuti | 1–40 |
| 21 | `alternative_consultate_sessione` | conteggio | 0–20 |
| 22 | `quota_cancellazioni_canale_storica_pct` | % | 0–70 |
| 23 | `giorni_validita_preventivo` | giorni | 1–30 |
| 24 | `richieste_speciali_confermate` | conteggio | 0–8 |
| 25 | `distanza_viaggio_stimata_km` | km | 5–2500 |
| 26 | `indice_disagi_trasporto_previsti` | punteggio 0–100 | 0–100 |
| 27 | `variabilita_tariffe_zona_pct` | % | 0–60 |
| 28 | `offerte_alternative_in_zona` | conteggio | 0–50 |
| 29 | `rischio_meteo_avverso_previsto_pct` | % | 0–100 |
| 30 | `flessibilita_date_viaggio_indice` | punteggio 0–100 | 0–100 |

## Lavoro richiesto al gruppo

1. Descrivere l'uso della previsione e il costo di falsi positivi e falsi negativi. Discutere precision dei contatti preventivi e recall delle cancellazioni; non introdurre overbooking automatico.
2. Seguire S1–S6 del Lab 13: baseline tracciata, API, telemetria, candidata, gate e drill.
3. Conservare la regola comune del laboratorio (accuracy v2 almeno pari a v1) e aggiungere nel report precision, recall e matrice di confusione. Non cambiare la soglia del gate dopo aver visto i risultati.
4. Presentare le misure effettive e i limiti: non sono garantiti né un'accuracy prefissata né il superamento del gate da parte di v2.

## Compatibilità e riproducibilità

Il contratto condiviso del laboratorio prevede 30 feature numeriche e target 0/1. Il gruppo costruisce il proprio lettore, training e servizio seguendo i checkpoint del lab. Cambiano i percorsi dei due CSV e il significato delle feature; modello, schema, ordine e metadati devono restare coerenti in tutte le fasi. Il training legge il file fornito e non deve generare nuovi dati.

Provenienza: generazione sintetica del docente, seed 1306. Ogni soggetto usa gruppi di variabili correlate, effetti non lineari e target probabilistico con rumore, con vincoli di coerenza su contatori, picchi e totali; nessuna feature contiene la label o l'esito futuro osservato. Il seed documenta la preparazione: gli input assegnati restano i CSV distribuiti. Lo split non simula una validazione temporale; una prova su dati futuri reali richiederebbe una valutazione distinta.
