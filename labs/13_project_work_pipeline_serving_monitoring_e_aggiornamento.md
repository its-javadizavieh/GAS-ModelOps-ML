# Lab 13 - Preparare e realizzare il project work finale

## Obiettivo e modalità di lavoro

Questo laboratorio guida il **progetto finale di gruppo**: partire da un problema, costruire una pipeline di classificazione, rendere disponibile un servizio e documentare come monitorarlo, aggiornarlo e ripristinarlo.

Ogni gruppo lavora su **un soggetto** del [catalogo dei progetti finali](progetti_finali/README.md). Le fasi e le evidenze richieste sono comuni; domanda applicativa, significato delle feature, interpretazione delle classi e discussione dei risultati dipendono dal soggetto assegnato.

Il lavoro dura **8 ore in due incontri**, con checkpoint intermedi. Il gruppo scrive la propria implementazione: questa guida offre passaggi, domande e controlli, senza una soluzione completa da copiare. Potete riutilizzare quanto avete imparato nei laboratori precedenti, spiegando gli adattamenti fatti.

## 1. Scegliere il soggetto e leggere la scheda

Il docente assegna il soggetto o concorda la scelta con il gruppo. Aprite la cartella corrispondente in `progetti_finali/` e leggete il suo README prima di iniziare.

| Materiale | A cosa serve |
| --- | --- |
| Questo Lab 13 | Organizzazione, fasi di costruzione e criteri della consegna |
| README.md del soggetto | Problema, provenienza dei dati, classi, feature, split e limiti |
| dataset.csv del soggetto | Dati da usare per training e valutazione |
| richieste_monitoraggio.csv dello stesso soggetto | Input per osservare il servizio e provare le transizioni tra versioni |

**Tutti i dataset e i payload dei progetti sono nelle rispettive cartelle di `progetti_finali/`.** Condividete nel gruppo il lab e la cartella assegnata; per svolgere il progetto basta quella cartella.

Usate la stessa coppia CSV durante tutte le fasi. Il numero di righe, il significato del target e la provenienza vanno letti nella scheda del soggetto. Nei materiali attualmente distribuiti le feature sono 30 e il target è binario, ma il vostro lettore deve ricavare nomi e ordine delle feature dall'intestazione.

Leggete i CSV forniti e mantenete gli split assegnati. `record_id`, `split` e `target` hanno ruoli tecnici e non entrano tra le feature. Il CSV delle richieste serve alle prove dell'API; non aggiunge esempi al training.

## 2. Organizzare il gruppo e l'ambiente

Preparate Python, NumPy, scikit-learn, MLflow, FastAPI, Pydantic, HTTPX e Uvicorn. Docker è un'estensione quando disponibile. Annotate le versioni realmente usate.

Create una cartella di lavoro del gruppo, per esempio `lavoro_lab13`, separata dagli input distribuiti. Al suo interno organizzate sorgenti, configurazioni, modelli, metadati, log e report. Conservate i percorsi dei CSV nella configurazione del progetto; chi riceve il README deve poter individuare gli stessi file.

Dividete i compiti tra dati/training, servizio/telemetria e aggiornamento/documentazione. Ogni parte deve essere riletta da un compagno. Tutti devono saper spiegare il flusso completo e il proprio contributo.

| Incontro | Fasi | Tempo indicativo | Checkpoint |
| --- | --- | --- | --- |
| Primo | S1, S2, S3 | 45 + 90 + 105 minuti | Contratto, baseline tracciata e servizio osservabile |
| Secondo | S4, S5, S6 | 90 + 90 + 60 minuti | Decisione sulla candidata, ripristino documentato e presentazione |

## 3. Prima del codice: scrivere una proposta breve

Preparate una pagina con queste risposte:

1. Quale evento volete prevedere e a quale uso serve la previsione?
2. Che cosa significano target 0 e target 1 nel vostro soggetto?
3. Quando avviene la previsione e quali informazioni sono già disponibili in quel momento?
4. Quali conseguenze hanno un falso positivo e un falso negativo?
5. Quali metriche userete e quali limiti operativi deve rispettare il servizio?
6. Quali attività assegnerete a ciascun componente?

Disegnate poi il percorso **dati → training → artefatto e metadati → servizio → osservazioni → decisione di aggiornamento**. Indicate dove viene verificato ogni passaggio e come si recupera la versione precedente.

## 4. Costruzione guidata del progetto

### S1 - Comprendere i dati e progettare i contratti

**Da costruire:** un lettore del CSV e una descrizione condivisa di input, output e metadati.

1. Leggete intestazioni e scheda del soggetto. Distinguete feature, identificativi, split e target; scrivete ordine, tipi e significato delle feature.
2. Progettate controlli per file vuoto, colonne duplicate, ID mancanti o ripetuti, valori non finiti, classi e split inattesi. Decidete quali informazioni deve restituire il lettore a training e valutazione.
3. Contate le righe e le classi per ogni split. Chiedetevi se una feature contiene informazioni che diventano disponibili soltanto dopo l'evento da prevedere.
4. Definite i metadati necessari: versione, run ID, fingerprint del CSV e dell'artefatto, ordine delle feature, ID di valutazione, configurazione e versioni software.
5. Scrivete prima delle misure la regola per confrontare baseline e candidata. Il protocollo comune del laboratorio richiede accuracy candidata almeno pari alla baseline, p95 locale ≤ 120 ms ed error rate ≤ 0,01 su richieste valide. Aggiungete una metrica motivata dal vostro soggetto, dichiarando la classe a cui si riferisce.

**Checkpoint:** un compagno può leggere il contratto, individuare i dati assegnati e spiegare perché gli input delle richieste non appartengono al training.

**Se vi bloccate:** iniziate dall'elenco delle colonne e dalla descrizione del risultato atteso del lettore. Il fingerprint si calcola sui byte del file: nome e numero di righe non bastano a identificarlo.

### S2 - Costruire e conservare la baseline

**Da costruire:** training ripetibile, valutazione e salvataggio di una prima versione v1.

1. Scegliete un modello semplice e motivate la scelta. Una pipeline con StandardScaler e LogisticRegression è un punto di partenza possibile per i dati numerici distribuiti.
2. Separate i dati usando la colonna split. Adattate sul solo train tutte le trasformazioni che apprendono parametri.
3. Valutate sulla validation e registrate accuracy, metrica scelta e matrice di confusione. Interpretate gli errori rispetto al problema del gruppo.
4. Collegate la run MLflow a configurazione, dataset e metriche. Salvate l'intera pipeline di inferenza e i suoi metadati, per esempio come `model.pkl` e `model_metadata.json` nella cartella della versione.
5. Progettate un caricatore che verifichi identità, schema e fingerprint prima di usare l'artefatto. Conservate v1 senza sovrascriverla quando preparerete v2.

**Checkpoint:** da una riga del report si risale a versione, run, dati, split e artefatto. Una copia dell'artefatto che non corrisponde ai metadati deve essere rifiutata con un motivo esplicito.

**Se vi bloccate:** separate lettura, fit, valutazione e salvataggio in responsabilità chiare. Il modello servito deve includere la stessa preparazione dei dati usata nel training.

### S3 - Costruire il servizio e osservarne le richieste

**Da costruire:** API con validazione, identità del modello e telemetria.

1. Definite come il servizio identifica la versione attiva. La selezione deve determinare quale artefatto viene realmente caricato.
2. Progettate il contratto di predict: feature numeriche finite, quantità e ordine coerenti con i metadati. Un input incompleto deve produrre 422 prima di chiamare il modello.
3. Fate restituire a health almeno versione e run ID; la risposta di predict deve contenere predizione, versione, run ID e request ID. Documentate anche lo schema atteso.
4. Registrate timestamp, request ID, versione, esito HTTP e durata, comprese le richieste respinte dalla validazione. Conservate i metadati operativi senza payload grezzi nei log.
5. Definite una finestra esplicita per le metriche: conteggio richieste, errori con stato ≥ 400, error rate e p95 in millisecondi. Separate health e lettura delle metriche dal conteggio delle predizioni.
6. Provate prima il contratto con un client in-process. Eseguite separatamente avvio HTTP e container, se disponibili, annotando ciò che avete realmente verificato.

**Checkpoint:** una richiesta valida identifica v1 e la sua run; una richiesta incompleta produce 422 e compare nel log. Un request ID collega risposta ed evento registrato.

**Se vi bloccate:** definite prima una richiesta valida e una incompleta a partire dai payload assegnati. Un client in-process verifica l'applicazione; rete e packaging richiedono prove distinte.

### S4 - Preparare e valutare una candidata

**Da costruire:** una versione v2 e una decisione motivata sul suo possibile utilizzo.

1. Formulate un'ipotesi: quale limite della baseline volete affrontare e con quale modifica? Potete cambiare modello o configurazione; documentate la scelta.
2. Allenate la candidata sullo stesso train. Conservate CSV, ordine delle feature e ID di validation della baseline. Valutate la vera v1 già salvata senza ricrearla.
3. Registrate run e artefatti di v2 in una cartella distinta. Confrontate accuracy, metrica del soggetto e matrice di confusione con lo stesso protocollo.
4. Misurate il servizio candidato in un'istanza di prova, mantenendo identificabili le misure della versione. Per la probe comune usate un payload valido assegnato, 3 richieste di warm-up e 30 misure locali; annotate p95, error rate, ambiente e fingerprint del modello misurato.
5. Costruite il gate: ogni requisito deve avere misura, soglia, esito e motivazione. Una probe mancante o attribuita a un altro artefatto non autorizza l'attivazione.

**Checkpoint:** il gruppo sa spiegare perché v2 è accettata o bloccata. Un miglioramento di accuracy non compensa automaticamente una violazione della latenza. La decisione del gate da sola non cambia la versione attiva.

**Se vi bloccate:** preparate una tabella con una riga per requisito. La validation serve al confronto didattico; non dichiaratela un test finale indipendente. Un payload ripetuto in-process non rappresenta il carico reale del servizio.

### S5 - Dimostrare attivazione e ripristino

**Da costruire:** operazioni esplicite di cambio versione e rollback, con evidenze osservabili.

1. Conservate lo stato attivo e la versione precedente; registrate per ogni transizione azione, stato prima/dopo, run e fingerprint.
2. Se la candidata supera il gate, documentate l'attivazione locale. Se viene bloccata, mantenete la decisione e svolgete il cambio versione soltanto come **drill locale dichiarato**.
3. Leggete il CSV delle richieste del vostro soggetto. Mostrate una predizione con v1, una con v2, una dopo il ripristino di v1, quindi la richiesta incompleta.
4. Correlate risposte e log. Le quattro richieste distribuite prevedono tre risposte valide e una 422: con errore definito come stato ≥ 400, l'error rate del drill è 0,25. Tenete separate le misure della probe da quelle del drill.
5. Controllate che lo stato finale e l'artefatto effettivamente servito siano quelli della baseline ripristinata.

**Checkpoint:** consegnate evidenze della sequenza v1 → v2 → v1, dell'input respinto e del registro delle transizioni. La sola stampa di «rollback» non dimostra che il modello usato dal servizio sia cambiato.

**Se vi bloccate:** distinguete il blocco di una candidata prima dell'attivazione dal ripristino dopo il cambio versione. Partite da una richiesta di controllo e verificate il modello realmente usato dopo ciascuna operazione.

### S6 - Preparare la consegna e la presentazione

**Da costruire:** una consegna riproducibile e una presentazione del lavoro del gruppo.

1. Scrivete il README del progetto: soggetto, scopo, classi, input, ambiente, procedura di esecuzione, risultati, stato finale e cleanup.
2. Organizzate le evidenze: run, configurazioni, tabella metriche, gate, richieste, log e transizioni. Riportate separatamente prova in-process, avvio HTTP e container; indicate «non eseguito» quando manca la prova.
3. Preparate la presentazione seguendo il percorso del progetto: problema → dati → baseline → servizio → candidata → decisione → ripristino → limiti.
4. Indicate il contributo di ciascun componente e le revisioni fatte nel gruppo. Tutti devono poter ricostruire una predizione fino a modello, run e dati.
5. Fermate processi e container avviati durante il lavoro. Conservate input assegnati, sorgenti, dipendenze, versioni dei modelli con metadati e le evidenze essenziali della consegna.

**Checkpoint:** un compagno segue il README e comprende come ripetere il procedimento. Il report distingue le metriche operative disponibili subito dalla qualità supervisionata, che richiede le label.

**Se vi bloccate:** iniziate dalle evidenze raccolte e collegate ogni risultato alla domanda iniziale. Descrivete le limitazioni e gli esiti osservati, anche quando la candidata non migliora.

## 5. Che cosa consegnare

| Parte | Contenuto richiesto |
| --- | --- |
| Progetto | Sorgenti del gruppo, configurazione e dipendenze |
| Dati e contratto | Soggetto scelto, riferimenti ai file assegnati, split, classi e ordine delle feature |
| Esperimenti | Baseline e candidata, run, parametri, metriche e interpretazione degli errori |
| Servizio | Contratto API, caso valido, caso non valido e livelli di esecuzione effettivamente provati |
| Aggiornamento | Gate motivato, stato attivo, registro e prova del ripristino |
| Documentazione | README, limiti, cleanup e contributi dei componenti |
| Presentazione | Sintesi del problema, delle scelte e delle evidenze osservate |

La valutazione considera coerenza del flusso, tracciabilità, correttezza dei controlli, interpretazione dei risultati e qualità della documentazione. Non sono richieste metriche identiche tra soggetti o gruppi, né il superamento del gate da parte della candidata.

## 6. Verifica finale e supporto durante il lavoro

La verifica finale comprende **quiz individuale tratto esclusivamente dai mini-esercizi delle slide e presentazione del project work di gruppo**. Non è previsto un esame orale separato.

Durante il laboratorio portate al docente la previsione iniziale, il tentativo fatto e il punto preciso in cui siete bloccati. Usate i laboratori precedenti come riferimento per i singoli componenti e spiegate come li avete adattati al vostro soggetto. Il codice e le evidenze del progetto finale devono essere costruiti e documentati dal gruppo.
