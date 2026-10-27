# Lab 09 - Permessi Audit e Fine Tuning Pipeline

## Cosa costruirai

Un ML engineer puo avviare un job, ma non approvarne il rilascio. Devi verificare che il controllo dei permessi preceda il training e che anche un rifiuto lasci una traccia.

## Materiale iniziale e ambiente

CSV distribuito `09_dataset_job_vini.csv`: 178 righe, 13 feature del benchmark Wine, target a 3 classi e split train/validation gia fissato. Il job legge questo file; non usa feedback clienti inesistenti.

**Strumenti da predisporre:** Python 3.12, numpy e scikit-learn.

**Contratto del dataset:** I CSV numerici hanno `record_id` univoco, `split` e `target`; tutte le altre colonne sono le feature nell'ordine dell'intestazione. Escludi questi tre campi dall'input del modello. Usa gli split distribuiti per il confronto del riferimento.

Lavora in una cartella personale sotto `materiali_didattici/`, chiamata `lavoro_lab09`; conserva separati input distribuiti e output delle prove. Prima di iniziare verifica Python, ambiente virtuale e dipendenze richieste dalla sessione.

**Consegna:** Matrice ruoli/azioni, `sol09_finetune_job.py`, configurazione job ed eventi di audit consentiti e negati.

**Tempo indicativo:** Percorso essenziale: 60-90 minuti con ambiente gia predisposto; estensioni se resta tempo nella sessione.

## Prima del codice: prevedi il risultato

Caso su carta: i numeri seguenti sono una scheda di esercizio, non risultati gia misurati.

| Attore | Ruolo       | Azione richiesta |
| ------ | ----------- | ---------------- |
| alice  | ml_engineer | train            |
| alice  | ml_engineer | approve          |
| bob    | approver    | approve          |

Quali richieste sono consentite? In quale momento deve essere scritto il rifiuto? Un campo ruolo nel log basta a imporre il permesso?

Annota la previsione e la motivazione. Confrontale con il risultato solo dopo avere costruito il flusso.

## Costruzione guidata

### S1 - Elenca azioni e ruoli

Definisci read_dataset, upload, train e approve e giustifica il permesso minimo di ogni ruolo.

**Fermati e verifica:** L'autore del training non approva automaticamente il proprio modello.

**Se ti blocchi:** Parti dai compiti, non dalla comodita di dare accesso a tutti.

### S2 - Costruisci la decisione di accesso

Crea una funzione che riceva attore, ruolo, azione e risorsa e produca consentito/negato.

**Fermati e verifica:** Ruolo sconosciuto e azione non prevista sono negati.

**Se ti blocchi:** Usa un comportamento di default deny.

### S3 - Definisci l'audit

Per ogni tentativo conserva event ID, timestamp reale della simulazione, attore, ruolo, azione, risorsa ed esito.

**Fermati e verifica:** Un evento negato contiene il motivo e nessuna credenziale.

**Se ti blocchi:** Attore e ruolo non sono lo stesso campo.

### S4 - Collega permesso e job

Prima di caricare dati o allenare, richiama il controllo; se consentito, associa configurazione e risultato a un run ID.

**Fermati e verifica:** Un attore non autorizzato non esegue il training.

**Se ti blocchi:** Non basta scrivere consentito alla fine di un training gia effettuato.

### S5 - Prova un tentativo negato

Esegui train con approver e approve con ml_engineer, poi un train autorizzato.

**Fermati e verifica:** Il log contiene entrambe le negazioni e il job consentito.

**Se ti blocchi:** Confronta eventi previsti ed eventi realmente prodotti.

### S6 - Spiega il passaggio a produzione

Descrivi dove servirebbero identita autenticate, storage protetto e approvazione separata per un vero job di fine-tuning.

**Fermati e verifica:** Il report distingue simulazione locale e controllo reale.

**Se ti blocchi:** Il modello allenato da zero e solo il carico di lavoro su cui esercitare la governance.

## Cosa consegnare e come valutarlo

| Criterio       | Evidenza richiesta                                                 |
| -------------- | ------------------------------------------------------------------ |
| Costruzione    | Componenti del flusso e responsabilita di ciascuno                 |
| Esecuzione     | Input, output ed esito osservati, con versione/configurazione      |
| Prova negativa | Un caso che deve fallire o contraddire l'ipotesi iniziale          |
| Ragionamento   | Confronto tra previsione iniziale e risultato, con una spiegazione |
| Limiti         | Parti eseguite, simulate e non eseguite indicate separatamente     |

La matrice in memoria e una simulazione: non sostituisce autenticazione o IAM. Il job sklearn parte da zero e non dimostra fine-tuning di pesi pre-addestrati.

Non sono richiesti numeri identici alla soluzione: valuta la correttezza del procedimento e la tracciabilita delle evidenze.

## Se manca uno strumento

Completa le fasi eseguibili localmente e descrivi input, stato e output delle altre in una tabella marcata **simulazione**. Uno screenshot di documentazione non dimostra che il servizio sia stato eseguito. Indica cosa resta da provare.

## Cleanup e conservazione

- Ferma solo processi, server o container avviati da te durante il lab.
- Rimuovi copie temporanee e output intermedi; conserva input didattici distribuiti e artefatti della consegna.
- Non eliminare credenziali, ambienti o risorse condivise. Non riportare segreti o input personali nei log.
- Nel report indica cosa hai chiuso, cosa hai conservato e perche.

## Quando consultare la soluzione

Prima completa almeno previsione, disegno del flusso e un tentativo dei passaggi S1-S3. La soluzione di riferimento usa la stessa numerazione S1-S6: confronta il passaggio in cui ti sei bloccato, quindi riprendi il tuo lavoro. Il codice completo serve alla revisione finale.
