# Soluzione Lab 10 - Fine Tuning Distillation e Ottimizzazione

## Percorso principale H1–H6 - Hugging Face

Leggi prima gli indizi H1–H6 del lab. Il CSV testuale è sintetico e locale, con 96/24/24 esempi train/validation/test. Il checkpoint e il tokenizer sono gli unici file scaricati dal Hub; nessun dataset viene scaricato e nessun dato viene inviato al Hub.

### Ambiente e file

Crea lavoro_lab10 in materiali_didattici e attiva Python 3.12. Per una macchina CPU Linux, installa PyTorch dal suo indice CPU; per altre piattaforme usa la distribuzione PyTorch adatta alla macchina. Le versioni sono fissate per mantenere l'API della guida.

```bash
python -m pip install torch==2.9.1 --index-url https://download.pytorch.org/whl/cpu
python -m pip install transformers==4.57.1 accelerate==1.11.0 numpy scikit-learn
```

### hf_finetuning_distillation.py

Trainer aggiorna anche il backbone preaddestrato; la testa di classificazione è nuova. Lo student compatto parte da una configurazione nuova e apprende con loss mista CE/KL dal teacher congelato. La baseline TF-IDF è addestrata solo sul train. I modelli salvati vengono ricaricati prima della valutazione finale.

```python
"""Hugging Face fine-tuning and distillation; examples come only from local CSV."""
import argparse
import csv
import hashlib
import importlib.metadata
import json
import pickle
import statistics
import time
from pathlib import Path

import numpy as np
import torch
import torch.nn.functional as F
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score
from sklearn.pipeline import make_pipeline
from torch.utils.data import DataLoader, Dataset
from transformers import (AutoModelForSequenceClassification, AutoTokenizer, BertConfig,
                          Trainer, TrainingArguments, set_seed)


class TextRows(Dataset):
    def __init__(self, rows, tokenizer):
        self.tokens = tokenizer([r['text'] for r in rows], padding='max_length',
                                truncation=True, max_length=48, return_tensors='pt')
        self.labels = torch.tensor([int(r['label']) for r in rows])

    def __len__(self):
        return len(self.labels)

    def __getitem__(self, i):
        return {**{k: v[i] for k, v in self.tokens.items()}, 'labels': self.labels[i]}


def read_rows(path):
    with path.open(newline='', encoding='utf-8') as stream:
        reader = csv.DictReader(stream)
        if reader.fieldnames != ['record_id', 'split', 'text', 'label']:
            raise ValueError('Schema CSV non previsto')
        rows = list(reader)
    if not rows or len({r['record_id'] for r in rows}) != len(rows):
        raise ValueError('CSV vuoto o ID duplicati')
    if len({r['text'] for r in rows}) != len(rows):
        raise ValueError('Testi duplicati tra gli esempi')
    if any(not r['text'].strip() or r['label'] not in {'0', '1'} for r in rows):
        raise ValueError('Testo o label non validi')
    if {r['split'] for r in rows} != {'train', 'validation', 'test'}:
        raise ValueError('Split non previsti')
    parts = {s: [r for r in rows if r['split'] == s] for s in ['train', 'validation', 'test']}
    if any({r['label'] for r in part} != {'0', '1'} for part in parts.values()):
        raise ValueError('Ogni split deve contenere entrambe le classi')
    return parts


def hf_predict(model, tokenizer, texts):
    model.eval()
    tokens = tokenizer(texts, padding='max_length', truncation=True,
                       max_length=48, return_tensors='pt')
    with torch.inference_mode():
        return model(**tokens).logits.argmax(-1).numpy()


def timed(call):
    for _ in range(3): call()
    durations = []
    for _ in range(20):
        start = time.perf_counter()
        call()
        durations.append((time.perf_counter() - start) * 1000)
    return statistics.median(durations)


def folder_bytes(folder):
    return sum(p.stat().st_size for p in folder.rglob('*') if p.is_file())


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--data', type=Path, required=True)
    parser.add_argument('--model', default='prajjwal1/bert-tiny')
    parser.add_argument('--revision', default='main', help='Per replicare usa il commit registrato nel report')
    parser.add_argument('--offline', action='store_true')
    parser.add_argument('--out', type=Path, default=Path('hf_output'))
    args = parser.parse_args()
    if args.out.exists():
        raise FileExistsError('Usa --out con una cartella nuova per conservare i risultati precedenti')
    args.out.mkdir(parents=True)
    torch.set_num_threads(2)
    set_seed(42)
    parts = read_rows(args.data)
    train, validation, test = (parts[s] for s in ['train', 'validation', 'test'])
    options = {'local_files_only': args.offline, 'revision': args.revision,
               'cache_dir': str(args.out.parent / 'hf_cache')}
    tokenizer = AutoTokenizer.from_pretrained(args.model, **options)
    teacher = AutoModelForSequenceClassification.from_pretrained(
        args.model, num_labels=2, id2label={0: 'negative', 1: 'positive'},
        label2id={'negative': 0, 'positive': 1}, **options).to('cpu')
    # Classification head is new. The pretrained encoder is updated by Trainer.
    encoder_before = teacher.bert.encoder.layer[0].attention.self.query.weight.detach().clone()
    train_ds, val_ds = TextRows(train, tokenizer), TextRows(validation, tokenizer)
    training = TrainingArguments(
        output_dir=str(args.out / 'trainer'), use_cpu=True,
        num_train_epochs=5, learning_rate=5e-5,
        per_device_train_batch_size=8, per_device_eval_batch_size=8,
        eval_strategy='epoch', save_strategy='no', logging_strategy='epoch',
        report_to='none', push_to_hub=False, disable_tqdm=True, seed=42,
        dataloader_num_workers=0, dataloader_pin_memory=False)
    trainer = Trainer(model=teacher, args=training, train_dataset=train_ds,
                      eval_dataset=val_ds, processing_class=tokenizer,
                      compute_metrics=lambda p: {'accuracy': float((p.predictions.argmax(-1) == p.label_ids).mean())})
    trainer.train()
    encoder_updated = not torch.equal(
        encoder_before, teacher.bert.encoder.layer[0].attention.self.query.weight.detach())
    if not encoder_updated:
        raise RuntimeError('Nessun aggiornamento osservato nel backbone')
    teacher.eval()
    for parameter in teacher.parameters(): parameter.requires_grad_(False)

    student = AutoModelForSequenceClassification.from_config(BertConfig(
        vocab_size=len(tokenizer), hidden_size=64, num_hidden_layers=1,
        num_attention_heads=2, intermediate_size=128, max_position_embeddings=128,
        num_labels=2, id2label={0: 'negative', 1: 'positive'},
        label2id={'negative': 0, 'positive': 1})).to('cpu')
    optimizer = torch.optim.AdamW(student.parameters(), lr=5e-4)
    loader = DataLoader(train_ds, batch_size=8, shuffle=True,
                        generator=torch.Generator().manual_seed(42))
    temperature = 2.
    for _ in range(5):
        student.train()
        for batch in loader:
            inputs = {k: v for k, v in batch.items() if k != 'labels'}
            with torch.no_grad(): soft_teacher = teacher(**inputs).logits
            logits = student(**inputs).logits
            hard_loss = F.cross_entropy(logits, batch['labels'])
            soft_loss = F.kl_div(F.log_softmax(logits / temperature, dim=-1),
                                 F.softmax(soft_teacher / temperature, dim=-1), reduction='batchmean')
            loss = .5 * hard_loss + .5 * temperature ** 2 * soft_loss
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()

    baseline = make_pipeline(TfidfVectorizer(ngram_range=(1, 2)), LogisticRegression(random_state=42))
    baseline.fit([r['text'] for r in train], [int(r['label']) for r in train])
    (args.out / 'baseline.pkl').write_bytes(pickle.dumps(baseline))
    for name, model in [('teacher', teacher), ('student', student)]:
        model.save_pretrained(args.out / name)
        tokenizer.save_pretrained(args.out / name)
    # Reload local artifacts; final metrics refer to what was actually saved.
    teacher = AutoModelForSequenceClassification.from_pretrained(args.out / 'teacher', local_files_only=True)
    student = AutoModelForSequenceClassification.from_pretrained(args.out / 'student', local_files_only=True)
    tokenizer = AutoTokenizer.from_pretrained(args.out / 'teacher', local_files_only=True)
    baseline = pickle.loads((args.out / 'baseline.pkl').read_bytes())
    texts, truth = [r['text'] for r in test], [int(r['label']) for r in test]
    batch = texts[:8]
    teacher_pred = hf_predict(teacher, tokenizer, texts)
    student_pred = hf_predict(student, tokenizer, texts)
    rows = [
        {'model': 'tfidf_baseline', 'accuracy': float(accuracy_score(truth, baseline.predict(texts))),
         'artifact_bytes': (args.out / 'baseline.pkl').stat().st_size,
         'median_ms_batch8': timed(lambda: baseline.predict(batch))},
        {'model': 'hf_teacher', 'accuracy': float(accuracy_score(truth, teacher_pred)),
         'artifact_bytes': folder_bytes(args.out / 'teacher'),
         'parameters': sum(p.numel() for p in teacher.parameters()),
         'median_ms_batch8': timed(lambda: hf_predict(teacher, tokenizer, batch))},
        {'model': 'hf_student', 'accuracy': float(accuracy_score(truth, student_pred)),
         'fidelity': float((teacher_pred == student_pred).mean()),
         'artifact_bytes': folder_bytes(args.out / 'student'),
         'parameters': sum(p.numel() for p in student.parameters()),
         'median_ms_batch8': timed(lambda: hf_predict(student, tokenizer, batch))}]
    result = {'data_sha256': hashlib.sha256(args.data.read_bytes()).hexdigest(),
              'split_ids': {s: [r['record_id'] for r in part] for s, part in parts.items()},
              'checkpoint': args.model, 'requested_revision': args.revision,
              'resolved_commit': getattr(trainer.model.config, '_commit_hash', None),
              'encoder_updated': encoder_updated,
              'versions': {p: importlib.metadata.version(p) for p in ['torch', 'transformers', 'accelerate', 'scikit-learn']},
              'protocol': 'CPU 2 thread; batch8; padding48; warmup3; mediana20; include tokenizzazione, esclude HTTP',
              'teacher_epochs': 5, 'student_epochs': 5, 'temperature': temperature,
              'student_loss': '0.5 CE + 0.5 T^2 KL', 'metrics': rows}
    (args.out / 'report.json').write_text(json.dumps(result, indent=2), encoding='utf-8')
    print(json.dumps(result, indent=2))


if __name__ == '__main__':
    main()
```

### Avvio e uso senza rete

```bash
python hf_finetuning_distillation.py --data ../labs/10_dataset_feedback_testuale.csv
```

La prima esecuzione scarica pesi e tokenizer nella cache locale hf_cache. Predisponila prima del laboratorio. Per ripetere senza rete usa --offline, --out con una cartella nuova e --revision con il resolved_commit salvato in report.json; in alternativa --model può indicare una copia locale del checkpoint. Nessuna autenticazione è necessaria per questo modello pubblico. Non salvare una testa casuale spacciandola per classificatore già addestrato.

Consegna report.json e cartelle teacher/student. Riporta hardware, versioni, batch e protocollo. Una mediana su batch testuale non è una latenza HTTP; byte degli artefatti e memoria del processo sono grandezze diverse. La riduzione dei parametri non garantisce migliore accuracy o minore latenza sulla tua macchina.

Riferimenti: [fine-tuning ufficiale v4.57.1](https://huggingface.co/docs/transformers/v4.57.1/en/training), [model card](https://huggingface.co/prajjwal1/bert-tiny).

## Percorso tabulare S1–S6 - confronto e fallback

I mini-esercizi su partial_fit e pickle si riferiscono a questo percorso. Non confrontare direttamente accuracy ottenute sui due diversi dataset.


## Percorso di revisione S1-S6

Consultare dopo il tentativo sul [lab studente](../labs/10_fine_tuning_distillation_e_ottimizzazione.md). Gli indizi e i controlli del lab precedono l'implementazione di riferimento.

| Passaggio | Parte da ricostruire          | Controllo richiesto prima di accettare la soluzione                                 |
| --------- | ----------------------------- | ----------------------------------------------------------------------------------- |
| S1        | Fissa protocollo e split      | Il test non entra ne nel training ne nei target dello student.                      |
| S2        | Allena e conserva la baseline | La baseline resta disponibile per il confronto sul test target.                     |
| S3        | Adatta il teacher             | I pesi iniziali del teacher provengono dalla baseline, non da un nuovo fit da zero. |
| S4        | Costruisci lo student         | Il target dello student e l'output del teacher, non la label del test.              |
| S5        | Misura in condizioni comuni   | La tabella specifica byte, ms per batch e dimensione del batch.                     |
| S6        | Decidi con vincoli espliciti  | Se nessun modello soddisfa i requisiti, dichiaralo senza cambiare i risultati.      |

**Limite da discutere:** Allenare da zero una forest piu grande e riaddestramento/configurazione, non fine-tuning. In questo lab il teacher prosegue davvero l'apprendimento della baseline tramite aggiornamenti incrementali.

**Risposta al caso su carta:** Solo lo student soddisfa entrambi i vincoli della scheda. La perdita e 0.03, tre punti percentuali. I valori sono un esercizio decisionale, non misure garantite dall'implementazione.

I valori nelle tabelle e negli output di esempio non attestano una nuova esecuzione. Registrare risultati reali, versioni dell'ambiente e prove negative nella propria consegna. Anche cleanup e accessi vanno verificati: non sono completati per il solo fatto di avere questo documento.

## Preparazione della cartella di lavoro

Crea `lavoro_lab10` direttamente in `materiali_didattici/` e copia i blocchi nei file indicati. Esegui i comandi da quella cartella: `../labs/` identifica i CSV distribuiti. Crea e attiva un ambiente virtuale Python 3.12; i comandi con `python` presuppongono che tale ambiente sia attivo. Registra le versioni installate nella consegna.

**Input assegnato:** CSV distribuito `10_dataset_cifre_domini.csv`: 1797 righe, 64 pixel, target 0-9 e split source/target_train/test disgiunti. Intensita normalizzate e trasformazione target sono gia applicate nel file; non ripeterle nel codice.

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

### finetuning_distillation.py

Il teacher nasce da una copia della baseline e continua gli aggiornamenti sui dati target. Lo student apprende le probabilita del teacher con una regressione multivariata.

```python
"""Adattamento incrementale reale e distillazione delle probabilita."""
import argparse
import copy
import json
import pickle
import time
from pathlib import Path
from statistics import median
import numpy as np
from sklearn.linear_model import SGDClassifier
from sklearn.metrics import accuracy_score
from sklearn.preprocessing import StandardScaler
from sklearn.tree import DecisionTreeRegressor
from tabular_data import read_dataset

SEED = 42


def latency_ms(predict, batch, repeats=50):
    for _ in range(5):
        predict(batch)
    durations = []
    for _ in range(repeats):
        start = time.perf_counter()
        predict(batch)
        durations.append((time.perf_counter() - start) * 1000)
    return median(durations)


def main():
    parser=argparse.ArgumentParser()
    parser.add_argument('--data',type=Path,required=True)
    args=parser.parse_args()
    X,y,splits,_=read_dataset(args.data,64,{'source','target_train','test'})
    source_idx=np.flatnonzero(splits=='source')
    target_idx=np.flatnonzero(splits=='target_train')
    test_idx=np.flatnonzero(splits=='test')
    # Normalizzazione di intensita e dominio sono gia applicati nel CSV.
    scaler = StandardScaler().fit(X[source_idx])
    source = scaler.transform(X[source_idx])
    target = scaler.transform(X[target_idx])
    test = scaler.transform(X[test_idx])
    y_test = y[test_idx]
    baseline = SGDClassifier(loss='log_loss', alpha=.0001, random_state=SEED,
                             learning_rate='constant', eta0=.005, max_iter=1000, tol=.001)
    baseline.fit(source, y[source_idx])
    teacher = copy.deepcopy(baseline)
    rng = np.random.default_rng(SEED)
    for _ in range(20):
        order = rng.permutation(len(target))
        teacher.partial_fit(target[order], y[target_idx][order])
    distillation_X = np.vstack([source, target])
    soft_targets = teacher.predict_proba(distillation_X)
    student = DecisionTreeRegressor(max_depth=6, min_samples_leaf=8, random_state=SEED)
    student.fit(distillation_X, soft_targets)
    def student_predict(values):
        return teacher.classes_[np.argmax(student.predict(values), axis=1)]
    batch = test[:32]
    rows = []
    for name, model, predict in [('baseline', baseline, baseline.predict),
                                  ('teacher', teacher, teacher.predict),
                                  ('student', student, student_predict)]:
        bundle = {'model': model, 'scaler': scaler, 'classes': teacher.classes_}
        rows.append({'model': name,
                     'accuracy': float(accuracy_score(y_test, predict(test))),
                     'serialized_bytes': len(pickle.dumps(bundle)),
                     'median_ms_per_batch': latency_ms(predict, batch),
                     'batch_size': len(batch)})
    fidelity = float(np.mean(student_predict(test) == teacher.predict(test)))
    print(json.dumps({'splits': {'source': len(source_idx), 'target_train': len(target_idx),
                                'test': len(test_idx)}, 'seed': SEED,
                      'measurements': rows, 'student_fidelity': fidelity,
                      'accuracy_drop_teacher_to_student': rows[1]['accuracy']-rows[2]['accuracy']}, indent=2))
    print('Misure di qualita sul test finale: non usarle per scegliere nuove configurazioni.')


if __name__ == '__main__':
    main()
```

## Esecuzione e controlli

```bash
python finetuning_distillation.py --data ../labs/10_dataset_cifre_domini.csv
```

I comandi di training e verifica terminano; UI e server restano attivi fino a Ctrl+C. Avviali dopo le verifiche. Su Windows usa gli stessi comandi nell'ambiente virtuale attivato.

## Protocollo di confronto

I tre insiemi source, target-training e test sono disgiunti. La trasformazione del dominio e deterministica; lo scaler e appreso solo sul source. La baseline resta intatta, il teacher continua l'apprendimento con partial_fit e lo student imita probabilita prodotte solo su dati di training.

| Misura              | Come interpretarla                                               |
| ------------------- | ---------------------------------------------------------------- |
| accuracy target     | Correttezza rispetto alle label reali del test target            |
| student_fidelity    | Quante predizioni dello student coincidono con il teacher        |
| serialized_bytes    | Byte effettivi della serializzazione di modello, scaler e classi |
| median_ms_per_batch | Mediana dopo warm-up, stesso batch da 32 campioni                |

Non e garantito che l'adattamento migliori la baseline o che lo student soddisfi i requisiti. La dimensione serializzata non coincide con memoria residente. La latenza riportata riguarda il modello su feature gia trasformate: per un budget di servizio va misurato anche preprocessing e HTTP.

## Checklist della propria consegna

- [ ] Previsione iniziale e motivazione registrate prima di eseguire il riferimento.
- [ ] Passaggi S1-S6 collegati a file, output o evidenze concrete.
- [ ] Caso nominale e prova negativa eseguiti con risultati annotati.
- [ ] Versioni, input e configurazioni sufficienti a ripetere la prova.
- [ ] Simulazione, prova in-process, servizio locale e container distinti nel report.
- [ ] Risorse di prova chiuse e artefatti finali conservati.

Una casella va marcata solo dopo la relativa verifica. I risultati numerici dipendono dall'ambiente e non sono prerequisiti da riprodurre artificialmente.
