# Progetto 1 - Abbandono di un servizio in abbonamento

## Domanda e momento della previsione

**Domanda:** Quali clienti potrebbero interrompere l’abbonamento nei prossimi 30 giorni?

**Momento di osservazione:** Fine del mese corrente; tutte le misure descrivono lo storico già disponibile.

- **target = 1:** Abbandono nei 30 giorni successivi.
- **target = 0:** Abbonamento mantenuto nei 30 giorni successivi.

Questi dati sono **interamente sintetici**, creati per il corso. Non rappresentano persone, aziende, macchine, edifici o prenotazioni reali. Le unità e gli intervalli descrivono la simulazione didattica. I risultati non misurano le prestazioni su un dominio reale.

## File da usare insieme

| File                                                     | Contenuto                                                                                                |
| -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| [dataset.csv](dataset.csv)                               | 1000 righe, 30 feature numeriche, record_id univoco, split e target binario                              |
| [richieste_monitoraggio.csv](richieste_monitoraggio.csv) | Quattro richieste coerenti con queste feature: v1 valida, v2 valida, v1 valida e v1 con 29 feature → 422 |

Ricevi anche il [Lab 13](../../13_project_work_pipeline_serving_monitoring_e_aggiornamento.md). Segui le fasi del lab usando i due CSV di questa cartella. Conserva la stessa coppia, lo stesso schema e lo stesso ordine delle feature durante tutto il progetto.

## Split e classi

| Split      | target 0 | target 1 | Totale |
| ---------- | -------: | -------: | -----: |
| train      |      490 |      310 |    800 |
| validation |      123 |       77 |    200 |

Lo split stratificato è già distribuito. Usalo per v1 e v2; non rigenerarlo. Record_id, split e target non entrano nelle feature. Non usare la validation per allenare lo scaler o il modello. Il CSV di richieste contiene solo input operativi: non è un altro dataset di training.

## Dizionario delle 30 feature

Il nome indica la misura e, dove previsto, la finestra temporale; `pct` indica percentuali: le quote sono tra 0 e 100, mentre gli scostamenti e le variazioni possono essere negativi secondo gli intervalli della tabella. I punteggi sono indici sintetici nella scala riportata. Per righe con unità «conteggio» i valori sono interi. Tutte le feature sono disponibili al momento della previsione descritto sopra.

**L'ordine seguente è l'ordine della lista features nell'API.** I payload distribuiti rispettano questo ordine; non contengono target, record_id o split.

| Posizione API | Colonna                             | Unità / scala   | Intervallo della simulazione |
| ------------- | ----------------------------------- | --------------- | ---------------------------- |
| 1             | `accessi_ultimi_30g`                | conteggio       | 0–70                         |
| 2             | `giorni_attivi_ultimi_30g`          | giorni          | 0–30                         |
| 3             | `giorni_da_ultimo_accesso`          | giorni          | 0–45                         |
| 4             | `sessioni_settimanali_medie`        | conteggio       | 0–20                         |
| 5             | `durata_sessione_media_min`         | minuti          | 1–60                         |
| 6             | `variazione_accessi_pct`            | %               | -95–100                      |
| 7             | `ticket_assistenza_90g`             | conteggio       | 0–18                         |
| 8             | `ticket_ancora_aperti`              | conteggio       | 0–8                          |
| 9             | `attesa_assistenza_media_ore`       | ore             | 0–60                         |
| 10            | `errori_app_30g`                    | conteggio       | 0–30                         |
| 11            | `reclami_90g`                       | conteggio       | 0–12                         |
| 12            | `soddisfazione_ultimo_supporto`     | punteggio 1–5   | 1–5                          |
| 13            | `canone_mensile_eur`                | EUR             | 5–100                        |
| 14            | `aumento_canone_ultimo_rinnovo_pct` | %               | 0–30                         |
| 15            | `fatture_scadute_90g`               | conteggio       | 0–6                          |
| 16            | `giorni_ritardo_pagamento_medio`    | giorni          | 0–25                         |
| 17            | `sconto_corrente_pct`               | %               | 0–40                         |
| 18            | `quota_canone_servizi_usati_pct`    | %               | 5–100                        |
| 19            | `anzianita_abbonamento_mesi`        | mesi            | 1–84                         |
| 20            | `rinnovi_precedenti`                | conteggio       | 0–15                         |
| 21            | `mesi_al_rinnovo`                   | mesi            | 0–12                         |
| 22            | `punteggio_fedelta`                 | punteggio 0–100 | 0–100                        |
| 23            | `prodotti_attivi`                   | conteggio       | 1–8                          |
| 24            | `inviti_amici_storici`              | conteggio       | 0–15                         |
| 25            | `funzionalita_usate_30g`            | conteggio       | 0–20                         |
| 26            | `quota_funzioni_avanzate_usate_pct` | %               | 0–100                        |
| 27            | `tutorial_completati`               | conteggio       | 0–12                         |
| 28            | `giorni_da_ultima_nuova_funzione`   | giorni          | 0–100                        |
| 29            | `quota_notifiche_aperte_pct`        | %               | 0–100                        |
| 30            | `download_report_30g`               | conteggio       | 0–25                         |

## Lavoro richiesto al gruppo

1. Descrivere l'uso della previsione e il costo di falsi positivi e falsi negativi. Discutere il recall della classe 1 e il costo dei contatti di retention inutili.
2. Seguire S1–S6 del Lab 13: baseline tracciata, API, telemetria, candidata, gate e drill.
3. Conservare la regola comune del laboratorio (accuracy v2 almeno pari a v1) e aggiungere nel report precision, recall e matrice di confusione. Non cambiare la soglia del gate dopo aver visto i risultati.
4. Presentare le misure effettive e i limiti: non sono garantiti né un'accuracy prefissata né il superamento del gate da parte di v2.

## Compatibilità e riproducibilità

Il contratto condiviso del laboratorio prevede 30 feature numeriche e target 0/1. Il gruppo costruisce il proprio lettore, training e servizio seguendo i checkpoint del lab. Cambiano i percorsi dei due CSV e il significato delle feature; modello, schema, ordine e metadati devono restare coerenti in tutte le fasi. Il training legge il file fornito e non deve generare nuovi dati.

Provenienza: generazione sintetica del docente, seed 1301. Ogni soggetto usa gruppi di variabili correlate, effetti non lineari e target probabilistico con rumore, con vincoli di coerenza su contatori, picchi e totali; nessuna feature contiene la label o l'esito futuro osservato. Il seed documenta la preparazione: gli input assegnati restano i CSV distribuiti. Lo split non simula una validazione temporale; una prova su dati futuri reali richiederebbe una valutazione distinta.
