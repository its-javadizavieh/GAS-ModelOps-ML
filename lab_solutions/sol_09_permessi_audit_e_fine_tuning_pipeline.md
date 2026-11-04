# Soluzione Lab 09 - Permessi Audit e Fine Tuning Pipeline

## Percorso di revisione S1-S6

Consultare dopo il tentativo sul [lab studente](../labs/09_permessi_audit_e_fine_tuning_pipeline.md). Gli indizi e i controlli del lab precedono l'implementazione di riferimento.

| Passaggio | Parte da ricostruire               | Controllo richiesto prima di accettare la soluzione                   |
| --------- | ---------------------------------- | --------------------------------------------------------------------- |
| S1        | Elenca azioni e ruoli              | L'autore del training non approva automaticamente il proprio modello. |
| S2        | Costruisci la decisione di accesso | Ruolo sconosciuto e azione non prevista sono negati.                  |
| S3        | Definisci l'audit                  | Un evento negato contiene il motivo e nessuna credenziale.            |
| S4        | Collega permesso e job             | Un attore non autorizzato non esegue il training.                     |
| S5        | Prova un tentativo negato          | Il log contiene entrambe le negazioni e il job consentito.            |
| S6        | Spiega il passaggio a produzione   | Il report distingue simulazione locale e controllo reale.             |

**Limite da discutere:** La matrice in memoria e una simulazione: non sostituisce autenticazione o IAM. Il job sklearn parte da zero e non dimostra fine-tuning di pesi pre-addestrati.

**Risposta al caso su carta:** Train per alice e approve per bob sono consentiti; approve per alice e negato. La verifica deve avvenire prima dell'azione e il diniego va registrato. Stampare un ruolo senza usare il controllo nel flusso non impone alcun permesso.

I valori nelle tabelle e negli output di esempio non attestano una nuova esecuzione. Registrare risultati reali, versioni dell'ambiente e prove negative nella propria consegna. Anche cleanup e accessi vanno verificati: non sono completati per il solo fatto di avere questo documento.

## Preparazione della cartella di lavoro

Crea `lavoro_lab09` direttamente in `materiali_didattici/` e copia i blocchi nei file indicati. Esegui i comandi da quella cartella: `../labs/` identifica i CSV distribuiti. Crea e attiva un ambiente virtuale Python 3.12; i comandi con `python` presuppongono che tale ambiente sia attivo. Registra le versioni installate nella consegna.

**Input assegnato:** CSV distribuito `09_dataset_job_vini.csv`: 178 righe, 13 feature del benchmark Wine, target a 3 classi e split train/validation gia fissato. Il job legge questo file; non usa feedback clienti inesistenti.

Dipendenze:

```bash
python -m pip install numpy scikit-learn
```

## Implementazione di riferimento

### tabular_data.py

Lettore condiviso: verifica schema, feature finite, target intero, record ID univoci e split previsti. Il codice consuma il CSV distribuito senza generarne uno nuovo.

```python
"""Contratto comune dei CSV distribuiti con i lab: nessun download/generazione."""
import csv
import numpy as np


def read_dataset(path, feature_count, allowed_splits):
    with path.open(newline='', encoding='utf-8') as stream:
        reader = csv.DictReader(stream)
        columns = reader.fieldnames or []
        if not {'record_id','split','target'} <= set(columns):
            raise ValueError('Servono record_id, split, target e le feature')
        if len(set(columns)) != len(columns):
            raise ValueError('Intestazioni duplicate')
        features = [c for c in columns if c not in {'record_id','split','target'}]
        rows = list(reader)
    if len(features) != feature_count or not rows:
        raise ValueError('Numero di feature errato o CSV vuoto')
    if len({row['record_id'] for row in rows}) != len(rows):
        raise ValueError('record_id duplicato')
    X = np.asarray([[float(row[c]) for c in features] for row in rows])
    targets = np.asarray([float(row['target']) for row in rows])
    if not np.isfinite(X).all() or not np.isfinite(targets).all():
        raise ValueError('Valori mancanti o non finiti')
    if not np.equal(targets, np.floor(targets)).all():
        raise ValueError('Il target di classificazione deve essere intero')
    splits = np.asarray([row['split'] for row in rows])
    if set(splits) != set(allowed_splits):
        raise ValueError(f'Split attesi: {allowed_splits}; trovati: {set(splits)}')
    return X, targets.astype(int), splits, features
```

### sol09_finetune_job.py

Il controllo di autorizzazione precede davvero il caricamento dei dati e il training. Gli attori sono identita fittizie gia associate a un ruolo nella simulazione.

```python
"""Autorizzazione e audit prima del job. Identita e permessi sono simulati."""
import argparse
import json
import uuid
from datetime import datetime, timezone
from pathlib import Path
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler
from tabular_data import read_dataset

ACTORS = {'alice': 'ml_engineer', 'bob': 'approver', 'carla': 'data_owner'}
PERMISSIONS = {'data_owner': {'read_dataset', 'upload'},
               'ml_engineer': {'read_dataset', 'train'},
               'approver': {'approve'}}
DATASET_VERSION = '09_dataset_job_vini.csv'
AUDIT = Path('audit.jsonl')


def event(actor, action, resource, allowed, **fields):
    row = {'event_id': uuid.uuid4().hex,
           'timestamp': datetime.now(timezone.utc).isoformat(),
           'actor': actor, 'role': ACTORS.get(actor, 'unknown'),
           'action': action, 'resource': resource,
           'outcome': 'allowed' if allowed else 'denied', **fields}
    with AUDIT.open('a', encoding='utf-8') as stream:
        stream.write(json.dumps(row) + '\n')
    print(json.dumps(row))
    return row


def authorize(actor, action, resource):
    allowed = action in PERMISSIONS.get(ACTORS.get(actor), set())
    event(actor, action, resource, allowed, reason='role policy')
    return allowed


def run_job(actor, data_path):
    if not authorize(actor, 'train', DATASET_VERSION):
        return None  # nessun caricamento o training dopo il diniego
    if not authorize(actor, 'read_dataset', DATASET_VERSION):
        return None
    run_id = uuid.uuid4().hex
    X,y,splits,_ = read_dataset(data_path,13,{'train','validation'})
    X_train, X_test = X[splits=='train'], X[splits=='validation']
    y_train, y_test = y[splits=='train'], y[splits=='validation']
    model = make_pipeline(StandardScaler(), LogisticRegression(max_iter=500))
    model.fit(X_train, y_train)
    accuracy = float(accuracy_score(y_test, model.predict(X_test)))
    event(actor, 'training_completed', DATASET_VERSION, True, run_id=run_id,
          base_model_version=None, training_mode='from_scratch',
          config={'seed': 42, 'max_iter': 500}, accuracy=accuracy)
    return run_id


def main():
    parser=argparse.ArgumentParser()
    parser.add_argument('--data',type=Path,required=True)
    args=parser.parse_args()
    run_job('bob',args.data)  # negato: approver non carica neppure il CSV
    authorize('alice', 'approve', 'candidate_model')  # negato
    run_id = run_job('alice',args.data)  # consentito, legge il CSV e allena
    if run_id:
        authorize('bob', 'approve', f'run:{run_id}')
    print('Simulazione locale: nessun sistema IAM o registry remoto modificato.')


if __name__ == '__main__':
    main()
```

## Esecuzione e controlli

```bash
python sol09_finetune_job.py --data ../labs/09_dataset_job_vini.csv
```

I comandi di training e verifica terminano; UI e server restano attivi fino a Ctrl+C. Avviali dopo le verifiche. Su Windows usa gli stessi comandi nell'ambiente virtuale attivato.

## Matrice di riferimento e lettura degli eventi

| Ruolo       | read_dataset | upload | train | approve |
| ----------- | ------------ | ------ | ----- | ------- |
| data_owner  | si           | si     | no    | no      |
| ml_engineer | si           | no     | si    | no      |
| approver    | no           | no     | no    | si      |

La prima prova nega train a bob; la seconda nega approve ad alice; la terza esegue il training per alice e bob puo approvare la run identificata. Il log distingue attore, ruolo, tentativo e completamento. Un evento di approvazione in questa simulazione non pubblica un modello su un registry.

Il job e allenato da zero: il nome storico del file non trasforma questo fit in fine-tuning. Per un vero adattamento occorre registrare anche la versione dei pesi iniziali e proseguirne l'apprendimento, come nel Lab 10.

## Checklist della propria consegna

- [ ] Previsione iniziale e motivazione registrate prima di eseguire il riferimento.
- [ ] Passaggi S1-S6 collegati a file, output o evidenze concrete.
- [ ] Caso nominale e prova negativa eseguiti con risultati annotati.
- [ ] Versioni, input e configurazioni sufficienti a ripetere la prova.
- [ ] Simulazione, prova in-process, servizio locale e container distinti nel report.
- [ ] Risorse di prova chiuse e artefatti finali conservati.

Una casella va marcata solo dopo la relativa verifica. I risultati numerici dipendono dall'ambiente e non sono prerequisiti da riprodurre artificialmente.
