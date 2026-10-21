# Lab 08 - Explainability Etica e Sicurezza Dati

## Obiettivo

- Interpretare una predizione con impatto su una persona reale e collegarla a rischi etici, base giuridica GDPR e misure di sicurezza.
- Collegare il risultato pratico ai micro-argomenti: Tecniche SHAP/LIME, casi reali e impatti sul business (angolo etico); Sicurezza dati/modelli, accessi e GDPR.

## Prerequisiti

- Python, SHAP o LIME, checklist GDPR, tabella rischi/accessi.
- Conoscenza base di Python, terminale, Git e ciclo di vita del modello.
- Fallback previsto: dataset didattico anonimizzato e checklist GDPR compilata a mano se mancano dati reali o strumenti di explainability.

## Scenario

- Un modello decide l'approvazione automatica di una richiesta di credito; un cliente contesta il rifiuto e chiede una spiegazione.
- Il team deve produrre una spiegazione comprensibile della decisione, verificare se tra le feature usate ci sono proxy di dati sensibili (es. CAP come proxy di etnia/reddito), e stabilire la base giuridica GDPR del trattamento.
- Va inoltre verificato chi ha accesso al modello, ai dati e ai log della decisione, e se serve una revisione umana prima di comunicare l'esito.
- Il deliverable finale deve essere presentabile sia a un revisore tecnico sia a un responsabile compliance.

## Step (numerati)

1. Genera una spiegazione SHAP o LIME per la predizione contestata, individuando le feature con maggior contributo al rifiuto.
2. Verifica se tra le feature influenti ci sono proxy di categorie protette (es. zona geografica, nome) e segnala il rischio.
3. Identifica la base giuridica GDPR del trattamento (consenso, contratto, interesse legittimo) e verifica se e documentata.
4. Compila una checklist minima di sicurezza: chi puo accedere al modello, ai dati di training e ai log delle predizioni.
5. Decidi se serve una revisione umana della decisione automatica e documenta la motivazione (art. 22 GDPR su decisioni automatizzate).
6. Scrivi la risposta da comunicare al cliente in linguaggio non tecnico, poi esegui il cleanup dei dati usati.

## Output atteso

- La spiegazione SHAP/LIME della predizione contestata con le feature piu influenti evidenziate.
- Una tabella con: feature proxy individuate, base giuridica GDPR, controllo di accesso verificato.
- Una nota su necessita di revisione umana e una bozza di risposta al cliente in linguaggio semplice.

## Checkpoint

- La spiegazione tecnica e stata tradotta in una motivazione comprensibile per una persona non tecnica.
- E stata verificata la presenza di feature proxy di categorie protette tra i fattori piu influenti.
- La base giuridica GDPR e il livello di accesso a dati/modello sono documentati esplicitamente.

## Troubleshooting rapido

- Se non hai dati reali disponibili, usa un dataset pubblico anonimizzato (es. credito/prestiti) e dichiara esplicitamente che e simulato.
- Se SHAP/LIME non individuano feature sospette, verifica comunque manualmente se variabili geografiche o demografiche sono presenti nel modello.
- Se la base giuridica GDPR non e chiara, classifica il caso come "da verificare con DPO" invece di inventare una risposta.
- Se compaiono dati personali reali nel materiale, rimuovili immediatamente e sostituiscili con valori fittizi prima di continuare.

## Cleanup obbligatorio

- Rimuovi ogni dataset con dati personali reali o simulati non necessari alla consegna finale.
- Elimina notebook, log o file temporanei che contengono nomi, CAP o altri identificativi anche fittizi.
- Chiudi eventuali accessi di test creati per verificare i permessi al modello o ai dati.
- Conferma nel deliverable che il cleanup e stato completato o indica cosa non e stato possibile rimuovere.
