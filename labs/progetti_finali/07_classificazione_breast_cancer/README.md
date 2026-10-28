# Progetto 7 - Classificazione tabulare: benchmark Breast Cancer

## Domanda del progetto

Come costruire, tracciare, servire e aggiornare un classificatore su un benchmark tabulare, conservando un collegamento verificabile tra dati, modello e risposta del servizio?

Il compito usa misure già presenti nel benchmark Breast Cancer distribuito con scikit-learn. Le etichette del CSV hanno questa codifica:

- **target = 0:** classe malignant (maligna).
- **target = 1:** classe benign (benigna).

Il dataset è un **benchmark pubblico, non un dataset sintetico**. Il progetto riguarda il processo ModelOps e non costituisce un sistema diagnostico o un'applicazione clinica. Le feature descrivono campioni del benchmark; la classe è l'output da prevedere e va esclusa dagli input.

## Materiale assegnato

| File | Contenuto |
| --- | --- |
| [dataset.csv](dataset.csv) | 569 righe, 30 feature numeriche, record_id, split e target binario |
| [richieste_monitoraggio.csv](richieste_monitoraggio.csv) | Tre richieste valide con versioni v1, v2, v1 e una richiesta incompleta con risposta attesa 422 |

Seguite il [Lab 13](../../13_project_work_pipeline_serving_monitoring_e_aggiornamento.md) con i due CSV di questa cartella. Il dataset di training e i payload appartengono allo stesso soggetto; mantenete la coppia durante tutto il progetto.

## Split distribuito

| Split | target 0 | target 1 | Totale |
| --- | ---: | ---: | ---: |
| train | 170 | 285 | 455 |
| validation | 42 | 72 | 114 |

Lo split è già assegnato e va mantenuto per baseline e candidata. Gli ID sono univoci. record_id, split e target non sono feature. Le richieste di monitoraggio non sono nuovi dati per il training.

## Dizionario e ordine delle feature

I nomi e le scale numeriche sono quelli del benchmark. I prefissi mean, error e worst distinguono tre riassunti della stessa famiglia di misure. Il gruppo non deve aggiungere una conversione di unità né formulare interpretazioni cliniche.

**L'ordine della tabella è l'ordine richiesto dalla lista features dell'API.** Il payload valido contiene 30 valori; quello incompleto ne contiene 29.

| Posizione API | Colonna | Descrizione |
| --- | --- | --- |
| 1 | `mean radius` | Media della misura: raggio |
| 2 | `mean texture` | Media della misura: texture |
| 3 | `mean perimeter` | Media della misura: perimetro |
| 4 | `mean area` | Media della misura: area |
| 5 | `mean smoothness` | Media della misura: regolarità locale |
| 6 | `mean compactness` | Media della misura: compattezza |
| 7 | `mean concavity` | Media della misura: concavità |
| 8 | `mean concave points` | Media della misura: punti concavi |
| 9 | `mean symmetry` | Media della misura: simmetria |
| 10 | `mean fractal dimension` | Media della misura: dimensione frattale |
| 11 | `radius error` | Errore standard della misura: raggio |
| 12 | `texture error` | Errore standard della misura: texture |
| 13 | `perimeter error` | Errore standard della misura: perimetro |
| 14 | `area error` | Errore standard della misura: area |
| 15 | `smoothness error` | Errore standard della misura: regolarità locale |
| 16 | `compactness error` | Errore standard della misura: compattezza |
| 17 | `concavity error` | Errore standard della misura: concavità |
| 18 | `concave points error` | Errore standard della misura: punti concavi |
| 19 | `symmetry error` | Errore standard della misura: simmetria |
| 20 | `fractal dimension error` | Errore standard della misura: dimensione frattale |
| 21 | `worst radius` | Riassunto dei valori più alti della misura: raggio |
| 22 | `worst texture` | Riassunto dei valori più alti della misura: texture |
| 23 | `worst perimeter` | Riassunto dei valori più alti della misura: perimetro |
| 24 | `worst area` | Riassunto dei valori più alti della misura: area |
| 25 | `worst smoothness` | Riassunto dei valori più alti della misura: regolarità locale |
| 26 | `worst compactness` | Riassunto dei valori più alti della misura: compattezza |
| 27 | `worst concavity` | Riassunto dei valori più alti della misura: concavità |
| 28 | `worst concave points` | Riassunto dei valori più alti della misura: punti concavi |
| 29 | `worst symmetry` | Riassunto dei valori più alti della misura: simmetria |
| 30 | `worst fractal dimension` | Riassunto dei valori più alti della misura: dimensione frattale |

## Lavoro richiesto

1. Seguire S1–S6 del Lab 13: contratto dati, baseline, API, telemetria, candidata, gate, ripristino e presentazione.
2. Dichiarare per quale classe si riportano precision, recall e F1. In questa codifica la classe 1 è benigna: non attribuire una metrica della classe 1 alla classe maligna.
3. Usare gli stessi ID validation per v1 e v2 e conservare artefatti, metadati e run separati.
4. Rispettare il protocollo del gate comune e raccogliere le evidenze del drill v1 → v2 → v1. Con tre risposte valide e una 422, l'error rate delle quattro richieste è 0,25.
5. Presentare le prestazioni misurate e i limiti del benchmark; una misura offline non dimostra qualità operativa in un impiego reale.

## Provenienza e conservazione

La coppia CSV già distribuita per il corso è stata organizzata in questa cartella mantenendo invariati dati, split e payload. Non occorre scaricare o generare altri dati. Il gruppo scrive la propria implementazione seguendo il laboratorio e conserva le evidenze della consegna.
