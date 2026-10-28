# Progetto 2 - Manutenzione predittiva di macchinari

## Domanda e momento della previsione

**Domanda:** Quali macchinari potrebbero richiedere un intervento per guasto entro 7 giorni?

**Momento di osservazione:** Fine del turno; sensori e contatori descrivono esclusivamente il periodo precedente.

- **target = 1:** Guasto con intervento nei 7 giorni successivi.
- **target = 0:** Nessun guasto nei 7 giorni successivi.

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
| train | 501 | 299 | 800 |
| validation | 125 | 75 | 200 |

Lo split stratificato è già distribuito. Usalo per v1 e v2; non rigenerarlo. Record_id, split e target non entrano nelle feature. Non usare la validation per allenare lo scaler o il modello. Il CSV di richieste contiene solo input operativi: non è un altro dataset di training.

## Dizionario delle 30 feature

Il nome indica la misura e, dove previsto, la finestra temporale; `pct` indica percentuali: le quote sono tra 0 e 100, mentre gli scostamenti e le variazioni possono essere negativi secondo gli intervalli della tabella. I punteggi sono indici sintetici nella scala riportata. Per righe con unità «conteggio» i valori sono interi. Tutte le feature sono disponibili al momento della previsione descritto sopra.

**L'ordine seguente è l'ordine della lista features nell'API.** I payload distribuiti rispettano questo ordine; non contengono target, record_id o split.

| Posizione API | Colonna | Unità / scala | Intervallo della simulazione |
| --- | --- | --- | --- |
| 1 | `temperatura_cuscinetto_media_c` | °C | 20–110 |
| 2 | `temperatura_motore_media_c` | °C | 20–120 |
| 3 | `temperatura_olio_media_c` | °C | 15–95 |
| 4 | `picco_temperatura_24h_c` | °C | 35–140 |
| 5 | `incremento_temperatura_24h_c` | °C | -10–25 |
| 6 | `minuti_sopra_soglia_termica_24h` | minuti | 0–350 |
| 7 | `vibrazione_rms_mm_s` | mm/s | 0.1–10 |
| 8 | `vibrazione_picco_mm_s` | mm/s | 0.5–20 |
| 9 | `rumore_medio_db` | dB | 40–110 |
| 10 | `oscillazione_velocita_pct` | % | 0–15 |
| 11 | `impulsi_anomali_24h` | conteggio | 0–50 |
| 12 | `asimmetria_vibrazioni_pct` | % | 0–45 |
| 13 | `ore_utilizzo_totali` | ore | 100–40000 |
| 14 | `ore_da_cambio_cuscinetto` | ore | 0–8000 |
| 15 | `ore_da_cambio_olio` | ore | 0–2200 |
| 16 | `particelle_olio_ppm` | ppm | 0–90 |
| 17 | `cicli_avvio_totali` | conteggio | 10–12000 |
| 18 | `spessore_residuo_componente_mm` | mm | 1–12 |
| 19 | `carico_medio_pct` | % | 10–100 |
| 20 | `picco_carico_pct` | % | 30–100 |
| 21 | `corrente_media_a` | A | 2–45 |
| 22 | `ore_marcia_ultime_24h` | ore | 0–24 |
| 23 | `avvii_ultime_24h` | conteggio | 0–60 |
| 24 | `pressione_media_bar` | bar | 1–14 |
| 25 | `giorni_da_ultima_manutenzione` | giorni | 0–180 |
| 26 | `interventi_rinviati_90g` | conteggio | 0–10 |
| 27 | `allarmi_precedenti_30g` | conteggio | 0–30 |
| 28 | `ore_fermo_precedenti_90g` | ore | 0–40 |
| 29 | `manutenzioni_programmate_completate_pct` | % | 20–100 |
| 30 | `livello_lubrificante_pct` | % | 5–100 |

## Lavoro richiesto al gruppo

1. Descrivere l'uso della previsione e il costo di falsi positivi e falsi negativi. Discutere recall dei guasti e falsi allarmi che anticipano interventi non necessari.
2. Seguire S1–S6 del Lab 13: baseline tracciata, API, telemetria, candidata, gate e drill.
3. Conservare la regola comune del laboratorio (accuracy v2 almeno pari a v1) e aggiungere nel report precision, recall e matrice di confusione. Non cambiare la soglia del gate dopo aver visto i risultati.
4. Presentare le misure effettive e i limiti: non sono garantiti né un'accuracy prefissata né il superamento del gate da parte di v2.

## Compatibilità e riproducibilità

Il contratto condiviso del laboratorio prevede 30 feature numeriche e target 0/1. Il gruppo costruisce il proprio lettore, training e servizio seguendo i checkpoint del lab. Cambiano i percorsi dei due CSV e il significato delle feature; modello, schema, ordine e metadati devono restare coerenti in tutte le fasi. Il training legge il file fornito e non deve generare nuovi dati.

Provenienza: generazione sintetica del docente, seed 1302. Ogni soggetto usa gruppi di variabili correlate, effetti non lineari e target probabilistico con rumore, con vincoli di coerenza su contatori, picchi e totali; nessuna feature contiene la label o l'esito futuro osservato. Il seed documenta la preparazione: gli input assegnati restano i CSV distribuiti. Lo split non simula una validazione temporale; una prova su dati futuri reali richiederebbe una valutazione distinta.
