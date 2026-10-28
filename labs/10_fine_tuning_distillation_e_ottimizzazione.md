# Lab 10 - Fine Tuning Distillation e Ottimizzazione

## Percorso principale - Hugging Face su testi locali

**Qui si usa davvero Hugging Face:** tokenizer, caricamento di pesi preaddestrati, fine-tuning con Trainer, student più piccolo e salvataggio/ricaricamento locale. Il CSV rimane un file fornito dal docente; dal Hub si scaricano soltanto modello e tokenizer. Non servono login, token o pubblicazione sul Hub.

### Materiali e preparazione

Ricevi `10_dataset_feedback_testuale.csv`: 144 brevi feedback sintetici in inglese, con `record_id`, `split`, `text`, `label` (0 negativo, 1 positivo). Gli split già assegnati sono train 96, validation 24, test 24, bilanciati e senza testi duplicati. Si usa l'inglese perché il checkpoint scelto è inglese. È un esercizio su un corpus minuscolo, non una valutazione di un sistema reale di sentiment analysis.

Prepara Python 3.12, PyTorch CPU, Transformers 4.57.1, Accelerate, NumPy e scikit-learn. Il docente deve predisporre dipendenze e cache prima dell'incontro. Il modello di partenza è `prajjwal1/bert-tiny`; in assenza di rete usa la copia locale predisposta. Se mancano anche le dipendenze passa al percorso tabulare seguente, dichiarando che non hai eseguito Hugging Face.

**Tempo:** circa 120 minuti per H1–H6 con ambiente pronto; 60–90 minuti per il confronto tabulare se rimane tempo. Il tempo effettivo dipende dalla macchina; non è richiesta una GPU.

### H1 - Progettare il lettore e il protocollo

Controlla intestazioni, ID e testi univoci, label valide e presenza di entrambe le classi in ogni split. Disegna tre insiemi separati. Fissa seed, lunghezza massima 48 token, batch 8, 5 epoche e budget prima della misura finale.

**Checkpoint:** né i testi né le label del test entrano nel fit, nella scelta dei parametri o nella produzione dei target dello student. La validation è disponibile durante il training.

### H2 - Preparare baseline e checkpoint

Progetta una baseline TF-IDF + LogisticRegression addestrata solo sul train. Poi individua le responsabilità di AutoTokenizer e AutoModelForSequenceClassification: il primo costruisce gli input numerici, il secondo carica il backbone e aggiunge una testa a due classi. La testa nuova è inizializzata casualmente: non chiamarla una baseline di sentiment già addestrata.

**Checkpoint:** annota nome del checkpoint, revisione risolta e hash del CSV. Conserva una copia di un tensore del backbone prima del training.

### H3 - Aggiornare il teacher con Trainer

Costruisci un Dataset PyTorch che restituisca input tokenizzati e labels. Definisci TrainingArguments per CPU, valutazione sulla validation a ogni epoca, nessun invio al Hub e nessun servizio esterno di logging. Collega modello, dataset e tokenizer a Trainer; esegui il training e salva il teacher.

**Checkpoint:** confronta il tensore del backbone prima/dopo: devono essere cambiati pesi preaddestrati, non soltanto il nome del file. Registra accuracy di validation senza modificare il test.

### H4 - Distillare uno student più piccolo

Definisci uno student BERT con 1 strato, hidden size 64 e 2 attention head, contro i 2 strati e hidden size 128 del teacher. Congela il teacher e mettilo in modalità valutazione. Sul solo train combina cross-entropy sulle label e KL tra distribuzioni di logits con temperatura 2; moltiplica il termine KL per il quadrato della temperatura. I target morbidi provengono dal teacher, non dalle predizioni sul test.

**Checkpoint:** la configurazione e il numero di parametri dimostrano che lo student è più piccolo. La sola cross-entropy sulle label non realizza questo trasferimento dal teacher.

### H5 - Misurare gli artefatti salvati

Salva teacher e student con save_pretrained e conserva il tokenizer. Ricarica da disco senza rete. Valuta baseline, teacher e student sul medesimo test, una volta fissati i parametri. Misura byte effettivi degli artefatti, numero parametri per i transformer e latenza CPU sullo stesso batch di 8 testi, con warm-up e ripetizioni; includi la tokenizzazione.

**Checkpoint:** non confondere byte su disco con memoria residente. I tempi locali non includono HTTP. Non è garantito che teacher o student superino la baseline su questo piccolo corpus.

### H6 - Motivare la scelta e conservare le prove

Confronta accuracy rispetto alle label e fedeltà rispetto al teacher. Scrivi quale modello soddisfa i requisiti fissati, oppure che nessuno li soddisfa. Consegna sorgente, report con split/seed/versioni, artefatti e tabella delle misure.

**Cleanup:** conserva gli artefatti finali e il report; rimuovi solo checkpoint intermedi e cache personali non più necessari dopo la consegna.

**Per approfondire:** [guida ufficiale al fine-tuning, versione 4.57.1](https://huggingface.co/docs/transformers/v4.57.1/en/training) e [model card del checkpoint](https://huggingface.co/prajjwal1/bert-tiny).

## Percorso tabulare di confronto / fallback - scikit-learn

I passaggi S1–S6 e i mini-esercizi su partial_fit e pickle si riferiscono al percorso seguente. Usa `10_dataset_cifre_domini.csv`, già distribuito. Le due prove hanno dati e modelli diversi: confronta i modelli all'interno di ciascun percorso; non confrontare direttamente le loro accuracy.



## Cosa costruirai

Devi adattare un modello gia allenato a un dominio sintetico diverso e addestrare uno student che ne riproduca le probabilita. Scegli il modello con misure di qualita, dimensione reale e latenza.

## Materiale iniziale e ambiente

CSV distribuito `10_dataset_cifre_domini.csv`: 1797 righe, 64 pixel, target 0-9 e split source/target_train/test disgiunti. I pixel source sono normalizzati dividendo per 16; target_train e test usano 0.65 volte il pixel normalizzato piu 0.12. Queste trasformazioni sono gia applicate nel CSV.

**Strumenti da predisporre:** Python 3.12, numpy e scikit-learn.

**Contratto del dataset:** I CSV numerici hanno `record_id` univoco, `split` e `target`; tutte le altre colonne sono le feature nell'ordine dell'intestazione. Escludi questi tre campi dall'input del modello. Usa gli split distribuiti per il confronto del riferimento.

Lavora in una cartella personale sotto `materiali_didattici/`, chiamata `lavoro_lab10`; conserva separati input distribuiti e output delle prove. Prima di iniziare verifica Python, ambiente virtuale e dipendenze richieste dalla sessione.

**Consegna:** `finetuning_distillation.py`, identificativi degli split, configurazioni baseline/teacher/student e tabella di misure.

**Tempo indicativo:** Percorso essenziale: 60-90 minuti con ambiente gia predisposto; estensioni se resta tempo nella sessione.

## Prima del codice: prevedi il risultato

Caso su carta: i numeri seguenti sono una scheda di esercizio, non risultati gia misurati.

| Modello | Accuracy target | Byte serializzati | Latenza ms/batch |
| --- | ---: | ---: | ---: |
| teacher | 0.94 | 90000 | 3.2 |
| student | 0.91 | 18000 | 0.7 |

Il requisito e accuracy >= 0.90 e latenza <= 1 ms/batch. Quale modello e ammissibile? Qual e la perdita in punti percentuali?

Annota la previsione e la motivazione. Confrontale con il risultato solo dopo avere costruito il flusso.

## Costruzione guidata

### S1 - Fissa protocollo e split

Leggi gli split source, target_train e test gia assegnati nel CSV e documenta la trasformazione descritta nella scheda dati qui sopra. Definisci requisiti di qualita e latenza prima della misura finale.

**Fermati e verifica:** Il test non entra ne nel training ne nei target dello student.

**Se ti blocchi:** Non correggere i requisiti dopo aver visto quale modello vince.

### S2 - Allena e conserva la baseline

Addestra un classificatore incrementale sul source e salva la configurazione. Prepara la trasformazione dei dati usando solo il source-training.

**Fermati e verifica:** La baseline resta disponibile per il confronto sul test target.

**Se ti blocchi:** Una copia indipendente evita che gli aggiornamenti modifichino anche la baseline.

### S3 - Adatta il teacher

Copia la baseline e prosegui gli aggiornamenti sul target-training con un numero fissato di passaggi.

**Fermati e verifica:** I pesi iniziali del teacher provengono dalla baseline, non da un nuovo fit da zero.

**Se ti blocchi:** Usa un estimatore che supporti apprendimento incrementale.

### S4 - Costruisci lo student

Ottieni probabilita del teacher sui soli dati di training e allena un modello compatto a riprodurle.

**Fermati e verifica:** Il target dello student e l'output del teacher, non la label del test.

**Se ti blocchi:** Mantieni l'ordine delle classi quando trasformi probabilita in predizioni.

### S5 - Misura in condizioni comuni

Valuta tutti sullo stesso test target, misura byte della serializzazione e latenza con batch fisso, warm-up e ripetizioni.

**Fermati e verifica:** La tabella specifica byte, ms per batch e dimensione del batch.

**Se ti blocchi:** Non trasformare un conteggio di nodi in KB con un fattore arbitrario.

### S6 - Decidi con vincoli espliciti

Calcola perdita di accuracy, riduzione dimensione e fedelta dello student al teacher.

**Fermati e verifica:** Se nessun modello soddisfa i requisiti, dichiaralo senza cambiare i risultati.

**Se ti blocchi:** Distingui accuratezza rispetto alle label e fedelta rispetto al teacher.

## Cosa consegnare e come valutarlo

| Criterio | Evidenza richiesta |
| --- | --- |
| Costruzione | Componenti del flusso e responsabilita di ciascuno |
| Esecuzione | Input, output ed esito osservati, con versione/configurazione |
| Prova negativa | Un caso che deve fallire o contraddire l'ipotesi iniziale |
| Ragionamento | Confronto tra previsione iniziale e risultato, con una spiegazione |
| Limiti | Parti eseguite, simulate e non eseguite indicate separatamente |

Allenare da zero una forest piu grande e riaddestramento/configurazione, non fine-tuning. In questo lab il teacher prosegue davvero l'apprendimento della baseline tramite aggiornamenti incrementali.

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
