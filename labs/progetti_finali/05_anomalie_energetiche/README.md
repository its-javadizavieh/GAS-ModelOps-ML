# Progetto 5 - Anomalie nei consumi energetici di un edificio

## Domanda e momento della previsione

**Domanda:** Quali edifici potrebbero mostrare un episodio di consumo anomalo nelle prossime 24 ore?

**Momento di osservazione:** Fine giornata; consumi e sensori riguardano le ultime 24 ore, il meteo riguarda previsioni disponibili.

- **target = 1:** Episodio anomalo nelle 24 ore successive.
- **target = 0:** Nessun episodio anomalo nelle 24 ore successive.

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
| train | 486 | 314 | 800 |
| validation | 121 | 79 | 200 |

Lo split stratificato è già distribuito. Usalo per v1 e v2; non rigenerarlo. Record_id, split e target non entrano nelle feature. Non usare la validation per allenare lo scaler o il modello. Il CSV di richieste contiene solo input operativi: non è un altro dataset di training.

## Dizionario delle 30 feature

Il nome indica la misura e, dove previsto, la finestra temporale; `pct` indica percentuali: le quote sono tra 0 e 100, mentre gli scostamenti e le variazioni possono essere negativi secondo gli intervalli della tabella. I punteggi sono indici sintetici nella scala riportata. Per righe con unità «conteggio» i valori sono interi. Tutte le feature sono disponibili al momento della previsione descritto sopra.

**L'ordine seguente è l'ordine della lista features nell'API.** I payload distribuiti rispettano questo ordine; non contengono target, record_id o split.

| Posizione API | Colonna | Unità / scala | Intervallo della simulazione |
| --- | --- | --- | --- |
| 1 | `consumo_ultime_24h_kwh` | kWh | 30–1000 |
| 2 | `scarto_consumi_da_media_30g_pct` | % | -40–100 |
| 3 | `potenza_picco_24h_kw` | kW | 5–150 |
| 4 | `quota_consumi_notturni_pct` | % | 0–100 |
| 5 | `picchi_potenza_24h` | conteggio | 0–35 |
| 6 | `variazione_consumo_7g_pct` | % | -40–80 |
| 7 | `tensione_media_v` | V | 195–265 |
| 8 | `deviazione_tensione_24h_v` | V | 0–20 |
| 9 | `microinterruzioni_7g` | conteggio | 0–20 |
| 10 | `squilibrio_fasi_pct` | % | 0–25 |
| 11 | `distorsione_armonica_pct` | % | 0–18 |
| 12 | `fattore_potenza_medio` | rapporto 0–1 | 0.4–1 |
| 13 | `efficienza_hvac_stimata_pct` | % | 20–100 |
| 14 | `ore_hvac_fuori_orario_24h` | ore | 0–12 |
| 15 | `dispersione_termica_indice` | punteggio 0–100 | 0–100 |
| 16 | `scarto_setpoint_interno_c` | °C | 0–10 |
| 17 | `filtri_con_manutenzione_scaduta` | conteggio | 0–12 |
| 18 | `giorni_da_controllo_impianto` | giorni | 0–365 |
| 19 | `gradi_ora_raffrescamento_previsti` | °C·ora | 0–180 |
| 20 | `gradi_ora_riscaldamento_previsti` | °C·ora | 0–180 |
| 21 | `variazione_meteo_prevista_c` | °C | 0–15 |
| 22 | `umidita_esterna_prevista_pct` | % | 10–100 |
| 23 | `occupazione_prevista_pct` | % | 0–100 |
| 24 | `ore_apertura_previste` | ore | 0–24 |
| 25 | `sensori_con_segnalazioni_7g` | conteggio | 0–18 |
| 26 | `campioni_mancanti_24h_pct` | % | 0–25 |
| 27 | `scarto_sensori_ridondanti_pct` | % | 0–30 |
| 28 | `allarmi_impianto_precedenti_30g` | conteggio | 0–25 |
| 29 | `ore_da_ultimo_controllo_sensori` | ore | 0–720 |
| 30 | `qualita_misure_pct` | % | 40–100 |

## Lavoro richiesto al gruppo

1. Descrivere l'uso della previsione e il costo di falsi positivi e falsi negativi. Discutere recall delle anomalie e numero di ispezioni inutili provocate dai falsi positivi.
2. Seguire S1–S6 del Lab 13: baseline tracciata, API, telemetria, candidata, gate e drill.
3. Conservare la regola comune del laboratorio (accuracy v2 almeno pari a v1) e aggiungere nel report precision, recall e matrice di confusione. Non cambiare la soglia del gate dopo aver visto i risultati.
4. Presentare le misure effettive e i limiti: non sono garantiti né un'accuracy prefissata né il superamento del gate da parte di v2.

## Compatibilità e riproducibilità

Il contratto condiviso del laboratorio prevede 30 feature numeriche e target 0/1. Il gruppo costruisce il proprio lettore, training e servizio seguendo i checkpoint del lab. Cambiano i percorsi dei due CSV e il significato delle feature; modello, schema, ordine e metadati devono restare coerenti in tutte le fasi. Il training legge il file fornito e non deve generare nuovi dati.

Provenienza: generazione sintetica del docente, seed 1305. Ogni soggetto usa gruppi di variabili correlate, effetti non lineari e target probabilistico con rumore, con vincoli di coerenza su contatori, picchi e totali; nessuna feature contiene la label o l'esito futuro osservato. Il seed documenta la preparazione: gli input assegnati restano i CSV distribuiti. Lo split non simula una validazione temporale; una prova su dati futuri reali richiederebbe una valutazione distinta.
