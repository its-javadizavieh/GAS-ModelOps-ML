# Progetto 7 - Escalation dei ticket di supporto IT

## Domanda e momento della previsione

**Domanda:** Quali ticket potrebbero richiedere il secondo livello di assistenza entro 24 ore?

**Momento di osservazione:** Conclusione del triage iniziale, prima dell’assegnazione definitiva. Le misure tecniche e lo storico sono già disponibili a questo momento.

- **target = 1:** Escalation al secondo livello entro 24 ore dal triage.
- **target = 0:** Nessuna escalation al secondo livello entro 24 ore dal triage.

I dati sono **interamente sintetici**, creati per il corso. Non rappresentano ticket, utenti, aziende, articoli o magazzini reali. Gli intervalli sono scelte della simulazione didattica; i risultati non misurano prestazioni su un sistema reale.

## File da usare insieme

| File | Contenuto |
| --- | --- |
| [dataset.csv](dataset.csv) | 1000 esempi, 30 feature numeriche, record_id, split e target 0/1 |
| [richieste_monitoraggio.csv](richieste_monitoraggio.csv) | Tre richieste valide v1 → v2 → v1 e una richiesta incompleta con 29 valori → 422 |

Segui il [Lab 13](../../13_project_work_pipeline_serving_monitoring_e_aggiornamento.md) usando i CSV di questa cartella. Il gruppo costruisce il proprio codice e conserva schema, ordine feature e metadati coerenti in tutte le fasi.

## Split e classi

| Split | target 0 | target 1 | Totale |
| --- | ---: | ---: | ---: |
| train | 475 | 325 | 800 |
| validation | 119 | 81 | 200 |

Lo split stratificato è già assegnato: usalo per v1 e v2. record_id, split e target non sono feature. Non adattare trasformazioni sul validation e non aggiungere i payload al training.

## Dizionario delle 30 feature

Tutte le feature sono disponibili al momento della previsione descritto sopra. Le percentuali di quota sono tra 0 e 100; le variazioni possono essere negative quando previsto nella tabella. I punteggi sono indici sintetici. Le quantità discrete sono intere; le altre misure mantengono decimali.

**L'ordine della tabella è l'ordine della lista features dell'API.** I payload distribuiti seguono lo stesso ordine senza includere il target.

| Posizione API | Colonna | Unità / scala | Intervallo della simulazione |
| --- | --- | --- | --- |
| 1 | `sistemi_coinvolti` | conteggio | 1–12 |
| 2 | `dipendenze_tecniche` | conteggio | 0–30 |
| 3 | `passaggi_riproduzione` | conteggio | 1–20 |
| 4 | `complessita_riproduzione_indice` | punteggio 0–100 | 0–100 |
| 5 | `allegati_tecnici` | conteggio | 0–10 |
| 6 | `codici_errore_distinti` | conteggio | 0–15 |
| 7 | `utenti_coinvolti_stimati` | conteggio | 1–2000 |
| 8 | `servizi_interessati` | conteggio | 1–10 |
| 9 | `durata_disservizio_segnalata_min` | minuti | 0–240 |
| 10 | `quota_postazioni_interessate_pct` | % | 0–100 |
| 11 | `criticita_servizio_indice` | punteggio 0–100 | 0–100 |
| 12 | `priorita_business_indice` | punteggio 0–100 | 0–100 |
| 13 | `errori_monitoraggio_ultimi_60min` | conteggio | 0–300 |
| 14 | `allarmi_asset_ultime_24h` | conteggio | 0–100 |
| 15 | `riavvii_asset_ultime_24h` | conteggio | 0–20 |
| 16 | `latenza_servizio_ultimi_15min_ms` | millisecondi | 10–5000 |
| 17 | `perdita_pacchetti_ultimi_15min_pct` | % | 0–40 |
| 18 | `carico_cpu_asset_pct` | % | 0–100 |
| 19 | `similarita_casi_risolti_pct` | % | 0–100 |
| 20 | `articoli_knowledge_base_pertinenti` | conteggio | 0–20 |
| 21 | `risoluzioni_primo_livello_categoria_storico_pct` | % | 0–100 |
| 22 | `recidive_asset_ultimi_30g` | conteggio | 0–15 |
| 23 | `cambi_configurazione_ultimi_7g` | conteggio | 0–20 |
| 24 | `anzianita_versione_software_mesi` | mesi | 0–72 |
| 25 | `ticket_in_coda_primo_livello` | conteggio | 0–300 |
| 26 | `occupazione_team_primo_livello_pct` | % | 0–100 |
| 27 | `specialisti_disponibili` | conteggio | 0–15 |
| 28 | `attesa_media_coda_ultimi_60min` | minuti | 0–240 |
| 29 | `quota_ticket_complessi_in_coda_pct` | % | 0–100 |
| 30 | `carico_turno_indice` | punteggio 0–100 | 0–100 |

## Lavoro del gruppo

1. Definire l'uso della previsione e il costo dei due tipi di errore. Confrontare recall dei ticket che richiedono escalation e precision delle assegnazioni al secondo livello. Discutere il costo delle escalation inutili e dei casi complessi lasciati al primo livello.
2. Seguire S1–S6 del Lab 13: contratto, baseline tracciata, API, osservazione, candidata, gate, drill e presentazione.
3. Conservare il gate comune (accuracy v2 almeno pari a v1 e vincoli operativi definiti nel lab); aggiungere precision, recall e matrice di confusione. Non cambiare la regola dopo aver visto i risultati.
4. Dimostrare il ripristino con le versioni realmente servite. Sulle quattro richieste assegnate, una 422 su quattro corrisponde a error rate 0,25.
5. Consegnare misure effettive, versioni, configurazioni, limiti e contributi individuali; nessuna accuracy o promozione di v2 è garantita.

## Provenienza e riproducibilità

Generazione sintetica del docente, seed 1307. Le feature hanno correlazioni per gruppi e il target dipende da un processo probabilistico con rumore ed effetti non lineari. Nessuna feature contiene l'esito futuro osservato. I CSV distribuiti sono gli input assegnati: il codice del gruppo deve leggerli, senza rigenerare dati o split.

La validation del laboratorio non simula un test temporale futuro. Per il calendario e la scadenza di presentazione segui la guida Lab 13 e il catalogo dei progetti.
