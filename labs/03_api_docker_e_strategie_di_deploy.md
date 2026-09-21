# Lab 03 - API Docker e Strategie di Deploy

## Obiettivo

- Confrontare cloud, on-prem e SaaS per un caso di deploy concreto e preparare il contratto API + Dockerfile minimale del servizio scelto.
- Collegare il risultato pratico ai micro-argomenti: API, Docker, cloud vs on-prem vs SaaS.

## Durata (timebox)

- 30 minuti.
- 5 minuti setup e lettura scenario.
- 20 minuti esecuzione guidata.
- 5 minuti checkpoint, cleanup e consegna evidenze.

## Prerequisiti

- Python, Docker (anche solo CLI/documentazione se non installato), terminale.
- Conoscenza base di Python, terminale, Git e ciclo di vita del modello.
- Fallback previsto: usare scheda di simulazione locale, tabelle Markdown o screenshot della documentazione quando tool, credenziali o permessi non sono disponibili.

## Scenario

- Un cliente con dati sanitari sensibili chiede di ospitare il modello di scoring, ma il team non ha ancora deciso se andare su cloud pubblico, infrastruttura on-prem del cliente o un servizio SaaS gestito.
- Il tuo gruppo deve produrre un confronto motivato tra le tre opzioni per questo caso specifico (vincoli: dati sensibili, budget limitato, nessun team infrastrutturale interno) e poi scrivere il contratto API del servizio che verrebbe esposto in ciascuna opzione.
- La decisione finale deve essere accompagnata da un Dockerfile minimale, perche qualunque opzione scelta il servizio deve comunque essere containerizzabile.

## Step (numerati)

1. Costruisci una tabella comparativa cloud vs on-prem vs SaaS con colonne: controllo sui dati, costo iniziale, competenze richieste, tempo di attivazione.
2. Applica la tabella allo scenario (dati sanitari sensibili, budget limitato, nessun team infra) e scegli un'opzione, motivando la scelta in 2-3 frasi.
3. Scrivi il contratto dell'endpoint `/predict`: payload di input atteso (campi e tipi), risposta di successo, risposta di errore per input non valido.
4. Scrivi un `Dockerfile` minimale (base image, dipendenze, comando di avvio) per il servizio, coerente con l'opzione di deploy scelta.
5. Verifica a mano (o con `docker build` se disponibile) che il Dockerfile sia sintatticamente corretto e coerente con il contratto API definito.
6. Esegui il cleanup e annota nel deliverable cosa hai rimosso o lasciato come placeholder.

## Output atteso

- La tabella comparativa cloud/on-prem/SaaS con la decisione motivata per lo scenario dato.
- Il contratto API (`/predict`) con payload, risposta valida e risposta di errore.
- Il Dockerfile minimale coerente con la scelta di deploy.

## Checkpoint

- La scelta tra cloud, on-prem e SaaS e motivata rispetto ai vincoli reali dello scenario (dati sensibili, budget, competenze), non generica.
- Il contratto API distingue chiaramente input valido, output atteso ed errore su input non valido.
- Il Dockerfile contiene base image, dipendenze e comando di avvio, senza elementi superflui.

## Troubleshooting rapido

- Se `docker build` fallisce per base image non trovata, verifica il tag dell'immagine (es. `python:3.11-slim`) e la connessione di rete.
- Se non hai Docker installato, valida il Dockerfile a occhio confrontandolo con un esempio ufficiale della documentazione FastAPI/Docker.
- Se la tabella comparativa risulta troppo generica, aggiungi una riga "rischio principale" specifica per ciascuna opzione nello scenario dato.
- Se il contratto API non copre il caso di errore, aggiungi esplicitamente un esempio di risposta 4xx con messaggio comprensibile.

## Cleanup obbligatorio

- Rimuovi immagini Docker di prova costruite localmente (`docker rmi`) e container fermi non necessari (`docker ps -a`).
- Elimina file temporanei creati per testare il Dockerfile (log di build, cache non necessarie).
- Se hai usato un registro cloud di prova, elimina l'immagine caricata o l'endpoint di test creato.
- Conferma nel deliverable che il cleanup e stato completato o indica cosa non e stato possibile rimuovere.
