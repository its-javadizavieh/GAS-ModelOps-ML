# Lab 11 - Reinforcement Learning e RLHF

## Cosa costruirai

Un assistente viene premiato per risposte brevi e sceglie una risposta corta ma incompleta. Devi confrontare il reward progettato con preferenze umane motivate.

## Materiale iniziale e ambiente

CSV distribuito `11_dataset_preferenze_simulate.csv`: 5 coppie A/B, testo, correttezza, tono e preferenza simulata di riferimento. Conta le parole dal testo. Prima raccogli le preferenze del gruppo usando le sole colonne domanda/testo, poi confrontale con quelle di riferimento.

**Strumenti da predisporre:** Python 3.12 e libreria standard (csv, dataclasses).

**Contratto del dataset:** Ogni `pair_id` deve avere una risposta A e una B alla stessa domanda. `correctness` e una scala 0-1 e `bad_tone` vale 0/1. Le colonne di preferenza sono giudizi simulati di riferimento: confrontali con le scelte indipendenti del gruppo.

Lavora in una cartella personale sotto `materiali_didattici/`, chiamata `lavoro_lab11`; conserva separati input distribuiti e output delle prove. Prima di iniziare verifica Python, ambiente virtuale e dipendenze richieste dalla sessione.

**Consegna:** Scheda stato/azione/reward, coppie A/B, preferenze con motivazione, `sol11_reward_ranking.py` facoltativo e revisione della rubrica.

**Tempo indicativo:** Percorso essenziale: 60-90 minuti con ambiente gia predisposto; estensioni se resta tempo nella sessione.

## Prima del codice: prevedi il risultato

Caso su carta: i numeri seguenti sono una scheda di esercizio, non risultati gia misurati.

| Risposta | Correttezza (0-1) | Parole |
| --- | ---: | ---: |
| A completa | 1 | 80 |
| B incompleta | 0.6 | 10 |

Con reward = correttezza - 0.01 per parola, quale risposta vince? E necessariamente la preferita dall'utente?

Annota la previsione e la motivazione. Confrontale con il risultato solo dopo avere costruito il flusso.

## Costruzione guidata

### S1 - Definisci il compito

Scrivi una domanda concreta, stato disponibile, azioni alternative e qualita desiderate.

**Fermati e verifica:** Le risposte A/B risolvono lo stesso compito e sono confrontabili.

**Se ti blocchi:** Non cambiare la domanda fra le due risposte della coppia.

### S2 - Fissa la rubrica

Definisci scale e pesi per correttezza, lunghezza e tono prima del confronto.

**Fermati e verifica:** Un altro valutatore puo applicare la stessa regola.

**Se ti blocchi:** La lunghezza e osservabile; la correttezza richiede un criterio di valutazione.

### S3 - Prepara le coppie

Leggi le cinque coppie fornite e controlla il caso della risposta breve ma scorretta. Per un'estensione puoi scrivere nuove coppie in una copia separata.

**Fermati e verifica:** Le annotazioni e la provenienza simulata sono esplicite.

**Se ti blocchi:** Inserisci un caso che possa mettere in crisi il reward.

### S4 - Raccogli preferenze indipendenti

Fai scegliere A/B e motivare prima di mostrare il punteggio del reward.

**Fermati e verifica:** Per ogni coppia esistono preferenza e motivo.

**Se ti blocchi:** Mostrare subito il punteggio puo condizionare il valutatore.

### S5 - Calcola e confronta

Calcola reward, vincitore, eventuale pareggio e accordo con la preferenza.

**Fermati e verifica:** Elenca almeno un disaccordo o spiega perche non ne hai osservati.

**Se ti blocchi:** Non forzare i dati per ottenere il risultato desiderato.

### S6 - Rivedi e collega a RLHF

Proponi una modifica della rubrica e descrivi preferenze -> reward model -> aggiornamento politica -> valutazione.

**Fermati e verifica:** Dichiara quali passaggi hai eseguito e quali solo progettato.

**Se ti blocchi:** Ricalcolare un reward manuale non equivale ad addestrare un reward model.

## Cosa consegnare e come valutarlo

| Criterio | Evidenza richiesta |
| --- | --- |
| Costruzione | Componenti del flusso e responsabilita di ciascuno |
| Esecuzione | Input, output ed esito osservati, con versione/configurazione |
| Prova negativa | Un caso che deve fallire o contraddire l'ipotesi iniziale |
| Ragionamento | Confronto tra previsione iniziale e risultato, con una spiegazione |
| Limiti | Parti eseguite, simulate e non eseguite indicate separatamente |

Confrontare punteggi e preferenze non addestra un reward model o una politica RL. Il lab prepara i dati e analizza il segnale che un ciclo RLHF potrebbe usare.

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
