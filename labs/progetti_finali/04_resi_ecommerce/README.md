# Progetto 4 - Resi degli ordini e-commerce

## Domanda e momento della previsione

**Domanda:** Quali ordini potrebbero essere restituiti entro 30 giorni dalla consegna?

**Momento di osservazione:** Conferma dell’ordine; nessun campo descrive il reso o la consegna futura dell’ordine corrente.

- **target = 1:** Reso entro 30 giorni dalla consegna.
- **target = 0:** Nessun reso entro 30 giorni dalla consegna.

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
| train | 477 | 323 | 800 |
| validation | 119 | 81 | 200 |

Lo split stratificato è già distribuito. Usalo per v1 e v2; non rigenerarlo. Record_id, split e target non entrano nelle feature. Non usare la validation per allenare lo scaler o il modello. Il CSV di richieste contiene solo input operativi: non è un altro dataset di training.

## Dizionario delle 30 feature

Il nome indica la misura e, dove previsto, la finestra temporale; `pct` indica percentuali: le quote sono tra 0 e 100, mentre gli scostamenti e le variazioni possono essere negativi secondo gli intervalli della tabella. I punteggi sono indici sintetici nella scala riportata. Per righe con unità «conteggio» i valori sono interi. Tutte le feature sono disponibili al momento della previsione descritto sopra.

**L'ordine seguente è l'ordine della lista features nell'API.** I payload distribuiti rispettano questo ordine; non contengono target, record_id o split.

| Posizione API | Colonna | Unità / scala | Intervallo della simulazione |
| --- | --- | --- | --- |
| 1 | `ordini_precedenti_cliente` | conteggio | 0–60 |
| 2 | `quota_resi_cliente_storica_pct` | % | 0–80 |
| 3 | `mesi_da_primo_ordine` | mesi | 0–72 |
| 4 | `ticket_taglia_precedenti` | conteggio | 0–12 |
| 5 | `cambi_prodotto_precedenti` | conteggio | 0–12 |
| 6 | `quota_ordini_conservati_storica_pct` | % | 10–100 |
| 7 | `quota_resi_prodotto_storica_pct` | % | 0–65 |
| 8 | `valutazione_media_prodotto` | punteggio 1–5 | 1–5 |
| 9 | `recensioni_su_difetti_pct` | % | 0–60 |
| 10 | `reclami_taglia_prodotto_pct` | % | 0–70 |
| 11 | `completezza_scheda_prodotto_pct` | % | 20–100 |
| 12 | `foto_dettaglio_disponibili` | conteggio | 1–15 |
| 13 | `articoli_nell_ordine` | conteggio | 1–12 |
| 14 | `varianti_stesso_articolo` | conteggio | 1–6 |
| 15 | `sconto_ordine_pct` | % | 0–70 |
| 16 | `valore_ordine_eur` | EUR | 5–500 |
| 17 | `quota_articoli_in_promozione_pct` | % | 0–100 |
| 18 | `spesa_rispetto_media_cliente_pct` | % | 30–250 |
| 19 | `consultazioni_guida_taglie` | conteggio | 0–10 |
| 20 | `tempo_scheda_prodotto_sec` | secondi | 5–400 |
| 21 | `confronti_prodotti_sessione` | conteggio | 0–15 |
| 22 | `modifiche_carrello_sessione` | conteggio | 0–15 |
| 23 | `completezza_preferenze_taglia_pct` | % | 0–100 |
| 24 | `compatibilita_taglia_stimata_pct` | % | 10–100 |
| 25 | `distanza_magazzino_destinazione_km` | km | 5–1500 |
| 26 | `giorni_consegna_promessi` | giorni | 1–12 |
| 27 | `quota_danni_vettore_storica_pct` | % | 0–25 |
| 28 | `fragilita_imballo_indice` | punteggio 0–100 | 0–100 |
| 29 | `peso_articolo_kg` | kg | 0.1–12 |
| 30 | `adeguatezza_imballo_indice` | punteggio 0–100 | 10–100 |

## Lavoro richiesto al gruppo

1. Descrivere l'uso della previsione e il costo di falsi positivi e falsi negativi. Discutere precision dei controlli preventivi e recall dei resi; evitare decisioni punitive automatiche.
2. Seguire S1–S6 del Lab 13: baseline tracciata, API, telemetria, candidata, gate e drill.
3. Conservare la regola comune del laboratorio (accuracy v2 almeno pari a v1) e aggiungere nel report precision, recall e matrice di confusione. Non cambiare la soglia del gate dopo aver visto i risultati.
4. Presentare le misure effettive e i limiti: non sono garantiti né un'accuracy prefissata né il superamento del gate da parte di v2.

## Compatibilità e riproducibilità

Il contratto condiviso del laboratorio prevede 30 feature numeriche e target 0/1. Il gruppo costruisce il proprio lettore, training e servizio seguendo i checkpoint del lab. Cambiano i percorsi dei due CSV e il significato delle feature; modello, schema, ordine e metadati devono restare coerenti in tutte le fasi. Il training legge il file fornito e non deve generare nuovi dati.

Provenienza: generazione sintetica del docente, seed 1304. Ogni soggetto usa gruppi di variabili correlate, effetti non lineari e target probabilistico con rumore, con vincoli di coerenza su contatori, picchi e totali; nessuna feature contiene la label o l'esito futuro osservato. Il seed documenta la preparazione: gli input assegnati restano i CSV distribuiti. Lo split non simula una validazione temporale; una prova su dati futuri reali richiederebbe una valutazione distinta.
