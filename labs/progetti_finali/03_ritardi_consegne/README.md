# Progetto 3 - Ritardi nelle consegne logistiche

## Domanda e momento della previsione

**Domanda:** Quali spedizioni potrebbero essere consegnate dopo la scadenza promessa?

**Momento di osservazione:** Momento della partenza; tempi e ritardi storici riguardano soltanto consegne precedenti.

- **target = 1:** Consegna successiva alla scadenza promessa.
- **target = 0:** Consegna entro la scadenza promessa.

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
| 1 | `distanza_pianificata_km` | km | 10–1300 |
| 2 | `numero_fermate_pianificate` | conteggio | 1–40 |
| 3 | `indice_congestione_prevista` | punteggio 0–100 | 0–100 |
| 4 | `quota_percorso_urbano_pct` | % | 0–100 |
| 5 | `deviazione_prevista_km` | km | 0–70 |
| 6 | `tempo_viaggio_stimato_ore` | ore | 0.5–20 |
| 7 | `pacchi_in_coda_hub` | conteggio | 10–1500 |
| 8 | `occupazione_hub_pct` | % | 10–100 |
| 9 | `attesa_carico_prevista_min` | minuti | 0–150 |
| 10 | `addetti_disponibili_pct` | % | 35–100 |
| 11 | `mezzi_disponibili_pct` | % | 25–100 |
| 12 | `variazione_volumi_hub_pct` | % | -40–80 |
| 13 | `ritardi_vettore_90g_pct` | % | 0–60 |
| 14 | `ritardi_rotta_90g_pct` | % | 0–70 |
| 15 | `ritardo_medio_rotta_90g_min` | minuti | 0–100 |
| 16 | `tentativi_falliti_zona_90g_pct` | % | 0–40 |
| 17 | `puntualita_vettore_90g_pct` | % | 20–100 |
| 18 | `esperienza_autista_rotta_mesi` | mesi | 1–60 |
| 19 | `pioggia_prevista_mm` | mm | 0–60 |
| 20 | `vento_previsto_km_h` | km/h | 0–90 |
| 21 | `rischio_neve_previsto_pct` | % | 0–100 |
| 22 | `cantieri_noti_sul_percorso` | conteggio | 0–15 |
| 23 | `tratti_con_limitazioni` | conteggio | 0–12 |
| 24 | `indice_criticita_meteo_prevista` | punteggio 0–100 | 0–100 |
| 25 | `margine_piano_rispetto_scadenza_ore` | ore | 0.2–16 |
| 26 | `flessibilita_finestra_consegna_ore` | ore | 0.5–12 |
| 27 | `peso_totale_carico_kg` | kg | 10–2000 |
| 28 | `saturazione_mezzo_pct` | % | 10–100 |
| 29 | `colli_fragili_pct` | % | 0–100 |
| 30 | `documenti_da_completare` | conteggio | 0–10 |

## Lavoro richiesto al gruppo

1. Descrivere l'uso della previsione e il costo di falsi positivi e falsi negativi. Discutere recall dei ritardi e precision delle notifiche preventive inviate ai clienti.
2. Seguire S1–S6 del Lab 13: baseline tracciata, API, telemetria, candidata, gate e drill.
3. Conservare la regola comune del laboratorio (accuracy v2 almeno pari a v1) e aggiungere nel report precision, recall e matrice di confusione. Non cambiare la soglia del gate dopo aver visto i risultati.
4. Presentare le misure effettive e i limiti: non sono garantiti né un'accuracy prefissata né il superamento del gate da parte di v2.

## Compatibilità e riproducibilità

Il contratto condiviso del laboratorio prevede 30 feature numeriche e target 0/1. Il gruppo costruisce il proprio lettore, training e servizio seguendo i checkpoint del lab. Cambiano i percorsi dei due CSV e il significato delle feature; modello, schema, ordine e metadati devono restare coerenti in tutte le fasi. Il training legge il file fornito e non deve generare nuovi dati.

Provenienza: generazione sintetica del docente, seed 1303. Ogni soggetto usa gruppi di variabili correlate, effetti non lineari e target probabilistico con rumore, con vincoli di coerenza su contatori, picchi e totali; nessuna feature contiene la label o l'esito futuro osservato. Il seed documenta la preparazione: gli input assegnati restano i CSV distribuiti. Lo split non simula una validazione temporale; una prova su dati futuri reali richiederebbe una valutazione distinta.
