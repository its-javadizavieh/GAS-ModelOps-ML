# Lab 12 - Configurare ed eseguire MLflow e Kubeflow Pipelines

## Obiettivo e lavoro richiesto

Esegui quattro esperimenti MLflow, confronta le run, recupera un modello dallo store e produci nuove predizioni. Poi compila ed esegui localmente una pipeline a tre componenti con l'SDK Kubeflow Pipelines.

**I due script completi sono inclusi nella sezione «Codice fornito».** Prepara l'ambiente, configura percorsi ed esecuzioni, lancia i programmi e documenta ciò che osservi. La scelta finale dello stack è una breve interpretazione delle prove raccolte.

## S1 - Prepara una cartella autonoma e l'ambiente

Crea una cartella personale chiamata `lavoro_lab12` con questi file:

| File | Come ottenerlo |
| --- | --- |
| Questo documento | Copia del materiale distribuito dal docente |
| `12_dataset_tracking_vini.csv` | CSV fornito dal docente, da conservare invariato |
| `mlflow_tracking.py` | Salva una sola volta il primo blocco Python della sezione «Codice fornito» |
| `kfp_pipeline.py` | Salva una sola volta il secondo blocco Python della stessa sezione |

Esegui tutti i comandi **dalla cartella lavoro_lab12**, con script e CSV nella stessa cartella. Non occorre scaricare tutto il repository.

Usa Python 3.12 e un terminale Linux/macOS; su Windows usa WSL2 per eseguire anche KFP, che avvia una shell POSIX. **Crea l’ambiente virtuale in un percorso senza spazi**: il launcher di KFP può fallire se il percorso dell’interprete contiene spazi. I file del progetto possono invece restare in una cartella con spazi.

```bash
python -m venv ~/lab12_venv
source ~/lab12_venv/bin/activate
```

Poi, dalla cartella `lavoro_lab12`, installa le dipendenze e registra le versioni:

```bash
python -m pip install numpy==2.5.4 scikit-learn==1.9.1 "mlflow==3.18.0" "kfp==2.18.0"
python -m pip freeze > ambiente.txt
```

Predisponi i pacchetti prima dell'incontro se la postazione è senza rete. Il runner KFP scelto usa sottoprocessi e l'ambiente locale già preparato. Non servono un cluster Kubernetes, Docker o credenziali cloud per questo percorso. Vertex AI resta un confronto progettuale.

### Configurazione da registrare

| Impostazione | Valore o posizione |
| --- | --- |
| Input | CSV assegnato: 178 righe, 13 feature, target 0/1/2 |
| Split | Colonna già distribuita: 124 train e 54 validation |
| Percorso dati | Argomento `--data` dei comandi che lo richiedono |
| Tracking URI | SQLite assoluto nella cartella di mlflow_tracking.py |
| Esperimento | lab12-esperimenti-locali |
| Artefatti MLflow | Cartella artifacts accanto allo script |
| UI | Processo locale sulla porta 5000 |
| Risultati KFP | Cartella nuova scelta con `--out` |

Leggi il CSV e individua ID, split, target e feature. I tre campi tecnici non entrano negli input del modello. Lo script verifica il contratto e mantiene lo split assegnato.

**Documenta:** ambiente, disposizione dei file, conteggi e configurazione. Mantieni invariato il CSV durante tutte le prove.

## S2 - Esegui le quattro run MLflow

```bash
python mlflow_tracking.py train --data 12_dataset_tracking_vini.csv
```

Il programma allena due pipeline StandardScaler + LogisticRegression con C 0,1 e 1, e due RandomForest con 100 alberi e profondità 3 e 5. Usa seed 42, stesso train e stessa validation. Lo scaler viene adattato nel fit sul solo train.

Ogni run registra parametri, accuracy, F1 macro, modello e metadati con hash e ordine delle feature. `session.json` conserva i quattro run ID, il tracking URI e il percorso del dataset.

**Documenta:** comando, URI effettivo, quattro run ID e configurazioni. Se ripeti train, vengono create nuove run e session.json si aggiorna: usa gli ID della sessione che stai documentando.

## S3 - Osserva la UI e confronta le run

In un **secondo terminale**, dalla stessa cartella e con lo stesso ambiente attivo, avvia:

```bash
python mlflow_tracking.py ui
```

Apri http://127.0.0.1:5000. Se compare la vista GenAI, passa a **Model training → lab12-esperimenti-locali → Runs**. Apri una run per vedere **Metrics**, **Parameters** e **Artifacts**. Il comando usa lo stesso database assoluto del training, anche quando il percorso contiene spazi. Seleziona le quattro run di session.json e osserva parametri, metriche e artefatti. Il processo UI resta attivo fino a Ctrl+C.

Nel terminale di lavoro esegui il confronto tramite client:

```bash
python mlflow_tracking.py compare
```

Il programma rilegge lo store, salva `comparison.csv` e seleziona la run in `selected_run.json` secondo la regola già fissata: massima F1 macro, poi accuracy, poi nome in caso di pareggio.

**Documenta:** vista della UI, tabella prodotta, run selezionata e motivo della selezione. Controlla che UI e client riportino gli stessi valori. La validation usata per scegliere una configurazione non è un test finale indipendente.

## S4 - Recupera il modello e produci predizioni

```bash
python mlflow_tracking.py restore --data 12_dataset_tracking_vini.csv
```

Il nuovo processo recupera modello e metadati tramite run ID, verifica hash e schema, carica la pipeline e produce predizioni sulla validation. Nella cartella `restored/<run_id>/` trovi `predictions.csv`, `restore_report.json` e gli artefatti recuperati.

Confronta accuracy ricalcolata e registrata e verifica che le righe di predizione identifichino la run usata. Il file modello nello store costituisce l'input di questa prova di recupero.

Esegui anche il caso negativo con un identificativo inesistente:

```bash
python mlflow_tracking.py restore --data 12_dataset_tracking_vini.csv --run-id 00000000000000000000000000000000
```

In questo tentativo è atteso un errore. Conservalo nel report e distinguilo dal recupero valido; non creare un modello sostitutivo per far apparire riuscito il comando.

**Documenta:** sorgente del modello, run ID, percorso delle predizioni, confronto delle metriche ed errore del caso negativo.

## S5 - Esegui la pipeline Kubeflow locale

```bash
python kfp_pipeline.py --data 12_dataset_tracking_vini.csv --out kfp_output
```

Il programma fornito esegue due operazioni: compila `wine_pipeline.yaml`, poi avvia la pipeline con `SubprocessRunner(use_venv=False)`. I componenti sono già definiti e collegati:

| Componente | Input e risultato da osservare |
| --- | --- |
| prepare_data | Legge il CSV e produce artefatti train, validation e contratto |
| train_model | Riceve il train e salva la pipeline scaler/classificatore |
| evaluate_model | Ricarica il modello e calcola le metriche sulla validation |

Conserva YAML, log dei tre task, artefatti nella sottocartella runs e `kfp_result.json`. Per ripetere usa una nuova cartella `--out`. L'opzione `--compile-only` produce il solo YAML e non completa la prova di esecuzione.

Per osservare un fallimento di input, usa un file inesistente e una cartella output nuova:

```bash
python kfp_pipeline.py --data file_inesistente.csv --out kfp_failure_output
```

È atteso il fallimento di prepare_data. Osserva il punto di interruzione e distingui questo tentativo dalla pipeline completata.

**Documenta:** comando, cartella, task eseguiti, risultato finale e fallimento osservato. I sottoinsiemi train/validation prodotti dalla pipeline derivano dal CSV assegnato; non sono altri dati scaricati.

Le prove MLflow e KFP sono separate e usano lo stesso input: il runner non registra automaticamente i task nello store MLflow. L'esecuzione locale non dimostra deployment Kubernetes, retry distribuiti, accessi multiutente o IAM cloud.

## S6 - Documenta, interpreta e consegna

Prepara un report breve con sei sezioni: ambiente/configurazione, run MLflow, confronto, recupero del modello, esecuzione KFP, problemi e limiti.

Allega `ambiente.txt`, i run ID, una vista della UI, comparison.csv, selected_run.json, predizioni e report di recupero, YAML e risultato KFP. Indica i percorsi degli artefatti e conserva database e store necessari a ripetere la prova.

Concludi con tre brevi decisioni: quale capacità serve a un team piccolo che condivide esperimenti, a un team con cluster e job dipendenti, e a un team che valuta servizi gestiti con IAM e budget. Motiva ogni scelta con due requisiti e un limite osservato. La prova Vertex AI è progettuale, non un'esecuzione cloud.

Ferma la UI con Ctrl+C. Registra gli errori e le fasi eventualmente rimaste non eseguite; conserva gli output della consegna e rimuovi soltanto intermedi personali non necessari.

## Codice fornito

Copia ciascun blocco completo nel file indicato, una sola volta. Mantieni gli script separati: ciascuno ha il proprio punto di avvio.

### mlflow_tracking.py

```python
"""Real MLflow runs, programmatic comparison and restored-model inference."""
import argparse
import csv
import hashlib
import json
import pickle
import platform
import subprocess
import sys
from pathlib import Path

import mlflow
import numpy as np
import sklearn
from mlflow import MlflowClient
from sklearn.ensemble import RandomForestClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score, f1_score
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler

BASE = Path(__file__).resolve().parent
URI = 'sqlite:///' + (BASE / 'mlflow.db').as_posix()
EXPERIMENT = 'lab12-esperimenti-locali'


def read_data(path):
    with path.open(newline='', encoding='utf-8') as stream:
        reader = csv.DictReader(stream)
        columns = reader.fieldnames or []
        rows = list(reader)
    if len(columns) != len(set(columns)) or not {'record_id', 'split', 'target'} <= set(columns):
        raise ValueError('Intestazione non valida')
    features = [c for c in columns if c not in {'record_id', 'split', 'target'}]
    if len(features) != 13 or not rows:
        raise ValueError('Servono il CSV distribuito e 13 feature')
    ids = np.asarray([r['record_id'] for r in rows])
    X = np.asarray([[float(r[c]) for c in features] for r in rows])
    targets = np.asarray([float(r['target']) for r in rows])
    split = np.asarray([r['split'] for r in rows])
    if len(set(ids)) != len(ids) or any(not value for value in ids):
        raise ValueError('ID vuoti o duplicati')
    if not np.isfinite(X).all() or not np.isfinite(targets).all() or not np.equal(targets, np.floor(targets)).all():
        raise ValueError('Feature o target non validi')
    if set(split) != {'train', 'validation'} or set(targets) != {0., 1., 2.}:
        raise ValueError('Split o classi non previsti')
    return X, targets.astype(int), split, features, ids


def client_and_experiment():
    mlflow.set_tracking_uri(URI)
    client = MlflowClient(tracking_uri=URI)
    exp = client.get_experiment_by_name(EXPERIMENT)
    if exp is None:
        experiment_id = client.create_experiment(EXPERIMENT, artifact_location=(BASE / 'artifacts').as_uri())
    else:
        experiment_id = exp.experiment_id
    return client, experiment_id


def train(data):
    _, experiment_id = client_and_experiment()
    X, y, split, features, ids = read_data(data)
    tr, va = split == 'train', split == 'validation'
    specs = [
        ('logreg_C_0_1', make_pipeline(StandardScaler(), LogisticRegression(C=.1, max_iter=1000, random_state=42)), {'algorithm': 'logreg', 'C': .1}),
        ('logreg_C_1', make_pipeline(StandardScaler(), LogisticRegression(C=1., max_iter=1000, random_state=42)), {'algorithm': 'logreg', 'C': 1.}),
        ('forest_depth_3', RandomForestClassifier(n_estimators=100, max_depth=3, random_state=42), {'algorithm': 'forest', 'max_depth': 3, 'n_estimators': 100}),
        ('forest_depth_5', RandomForestClassifier(n_estimators=100, max_depth=5, random_state=42), {'algorithm': 'forest', 'max_depth': 5, 'n_estimators': 100}),
    ]
    run_ids = []
    for name, model, params in specs:
        with mlflow.start_run(experiment_id=experiment_id, run_name=name) as run:
            model.fit(X[tr], y[tr])
            pred = model.predict(X[va])
            scores = {'accuracy': float(accuracy_score(y[va], pred)),
                      'f1_macro': float(f1_score(y[va], pred, average='macro'))}
            raw = pickle.dumps(model)
            meta = {'dataset_sha256': hashlib.sha256(data.read_bytes()).hexdigest(),
                    'model_sha256': hashlib.sha256(raw).hexdigest(),
                    'run_id': run.info.run_id, 'feature_names': features,
                    'train_ids': ids[tr].tolist(), 'validation_ids': ids[va].tolist(),
                    'split': 'colonna CSV assegnata, nessun nuovo split', 'metrics': scores,
                    'python': platform.python_version(), 'sklearn': sklearn.__version__, 'mlflow': mlflow.__version__}
            mlflow.log_params({**params, 'seed': 42, 'train_rows': int(tr.sum()), 'validation_rows': int(va.sum())})
            mlflow.set_tag('data_sha256', meta['dataset_sha256'])
            mlflow.log_metrics(scores)
            mlflow.log_dict(meta, 'metadata.json')
            # Log files from a temporary author-owned folder; the tracked copy is the source for restore.
            folder = BASE / 'local_models' / run.info.run_id
            folder.mkdir(parents=True)
            (folder / 'model.pkl').write_bytes(raw)
            mlflow.log_artifact(str(folder / 'model.pkl'))
            run_ids.append(run.info.run_id)
    session = {'tracking_uri': URI, 'experiment_id': experiment_id, 'run_ids': run_ids,
               'data_path': str(data.resolve())}
    (BASE / 'session.json').write_text(json.dumps(session, indent=2))
    print(json.dumps(session, indent=2))
    print('UI nello stesso store: python mlflow_tracking.py ui')


def compare():
    client, experiment_id = client_and_experiment()
    session = json.loads((BASE / 'session.json').read_text())
    expected = set(session['run_ids'])
    runs = client.search_runs([experiment_id], filter_string="attributes.status = 'FINISHED'", max_results=1000)
    rows = [{'run_id': r.info.run_id, 'name': r.data.tags.get('mlflow.runName', ''),
             'accuracy': r.data.metrics['accuracy'], 'f1_macro': r.data.metrics['f1_macro']}
            for r in runs if r.info.run_id in expected]
    if len(rows) != 4:
        raise RuntimeError('Non sono state recuperate le quattro run della sessione')
    # Deterministic rule fixed before viewing the results; validation is not an independent final test.
    rows.sort(key=lambda r: (-r['f1_macro'], -r['accuracy'], r['name']))
    with (BASE / 'comparison.csv').open('w', newline='', encoding='utf-8') as stream:
        writer = csv.DictWriter(stream, fieldnames=list(rows[0]))
        writer.writeheader()
        writer.writerows(rows)
    chosen = {**rows[0], 'rule': 'max f1_macro; tie: accuracy, then name', 'tracking_uri': URI}
    (BASE / 'selected_run.json').write_text(json.dumps(chosen, indent=2))
    print(json.dumps({'comparison': rows, 'selected': chosen}, indent=2))


def restore(data, run_id=None):
    client, _ = client_and_experiment()
    run_id = run_id or json.loads((BASE / 'selected_run.json').read_text())['run_id']
    dest = BASE / 'restored' / run_id
    dest.mkdir(parents=True, exist_ok=True)
    model_path = Path(client.download_artifacts(run_id, 'model.pkl', str(dest)))
    meta_path = Path(client.download_artifacts(run_id, 'metadata.json', str(dest)))
    meta = json.loads(meta_path.read_text())
    raw = model_path.read_bytes()
    if hashlib.sha256(raw).hexdigest() != meta['model_sha256'] or meta['run_id'] != run_id:
        raise RuntimeError('Artefatto e metadati non corrispondono')
    if hashlib.sha256(data.read_bytes()).hexdigest() != meta['dataset_sha256']:
        raise ValueError('Dataset diverso da quello registrato nella run')
    X, y, split, features, ids = read_data(data)
    va = split == 'validation'
    if features != meta['feature_names'] or ids[va].tolist() != meta['validation_ids']:
        raise ValueError('Schema o righe validation diverse dalla run')
    model = pickle.loads(raw)  # only the artifact produced in this local lab
    pred = model.predict(X[va])
    score = float(accuracy_score(y[va], pred))
    rows = [{'record_id': str(i), 'target': int(t), 'prediction': int(p), 'run_id': run_id}
            for i,t,p in zip(ids[va], y[va], pred)]
    with (dest / 'predictions.csv').open('w', newline='', encoding='utf-8') as stream:
        writer = csv.DictWriter(stream, fieldnames=list(rows[0]));writer.writeheader();writer.writerows(rows)
    report = {'run_id': run_id, 'restored_accuracy': score, 'logged_accuracy': meta['metrics']['accuracy'],
              'prediction_rows': len(rows), 'model_source': 'MLflow artifact store'}
    (dest / 'restore_report.json').write_text(json.dumps(report, indent=2))
    print(json.dumps(report, indent=2))


def main():
    parser = argparse.ArgumentParser()
    sub = parser.add_subparsers(dest='command', required=True)
    for name in ['train', 'restore']:
        p = sub.add_parser(name)
        p.add_argument('--data', type=Path, required=True)
        if name == 'restore': p.add_argument('--run-id')
    sub.add_parser('compare')
    sub.add_parser('ui')
    args = parser.parse_args()
    if args.command == 'train': train(args.data)
    elif args.command == 'compare': compare()
    elif args.command == 'ui':
        subprocess.run([sys.executable, '-m', 'mlflow', 'ui', '--backend-store-uri', URI,
                        '--host', '127.0.0.1', '--port', '5000'], check=True)
    else: restore(args.data, args.run_id)


if __name__ == '__main__':
    main()
```

### kfp_pipeline.py

```python
"""Compile and execute a three-step Kubeflow pipeline locally, without a cluster."""
import argparse
import json
import sys
from pathlib import Path

from kfp import compiler, dsl, local
from kfp.dsl import Artifact, Dataset, Input, Model, Output


@dsl.component(base_image='python:3.12-slim', install_kfp_package=False)
def prepare_data(data_path: str, training: Output[Dataset], validation: Output[Dataset], contract: Output[Artifact]):
    import csv
    import hashlib
    import json
    import math
    from pathlib import Path
    path = Path(data_path)
    with path.open(newline='', encoding='utf-8') as stream:
        reader = csv.DictReader(stream)
        names = reader.fieldnames or []
        rows = list(reader)
    if not rows or len(names) != len(set(names)) or not {'record_id', 'split', 'target'} <= set(names):
        raise ValueError('CSV non valido')
    features = [n for n in names if n not in {'record_id', 'split', 'target'}]
    if len(features) != 13 or len({r['record_id'] for r in rows}) != len(rows):
        raise ValueError('Schema o ID non validi')
    if any(not r['record_id'] for r in rows) or {r['split'] for r in rows} != {'train', 'validation'}:
        raise ValueError('ID o split inattesi')
    if any(not math.isfinite(float(r[c])) for r in rows for c in features):
        raise ValueError('Feature non finite')
    if any(r['target'] not in {'0', '1', '2'} for r in rows):
        raise ValueError('Classi inattese')
    for artifact, split in [(training, 'train'), (validation, 'validation')]:
        Path(artifact.path).parent.mkdir(parents=True, exist_ok=True)
        with open(artifact.path, 'w', newline='', encoding='utf-8') as stream:
            writer = csv.DictWriter(stream, fieldnames=names)
            writer.writeheader()
            writer.writerows([r for r in rows if r['split'] == split])
    metadata = {'feature_names': features, 'data_sha256': hashlib.sha256(path.read_bytes()).hexdigest(),
                'train_rows': sum(r['split'] == 'train' for r in rows),
                'validation_rows': sum(r['split'] == 'validation' for r in rows)}
    Path(contract.path).parent.mkdir(parents=True, exist_ok=True)
    Path(contract.path).write_text(json.dumps(metadata, indent=2))


@dsl.component(base_image='python:3.12-slim', install_kfp_package=False)
def train_model(training: Input[Dataset], contract: Input[Artifact], regularization: float, model: Output[Model]):
    import csv
    import json
    import pickle
    from pathlib import Path
    import numpy as np
    from sklearn.linear_model import LogisticRegression
    from sklearn.pipeline import make_pipeline
    from sklearn.preprocessing import StandardScaler
    meta = json.loads(Path(contract.path).read_text())
    with open(training.path, newline='', encoding='utf-8') as stream:
        rows = list(csv.DictReader(stream))
    X = np.asarray([[float(r[c]) for c in meta['feature_names']] for r in rows])
    y = np.asarray([int(r['target']) for r in rows])
    estimator = make_pipeline(StandardScaler(), LogisticRegression(C=regularization, max_iter=1000, random_state=42))
    estimator.fit(X, y)
    bundle = {'model': estimator, 'feature_names': meta['feature_names'], 'data_sha256': meta['data_sha256']}
    Path(model.path).parent.mkdir(parents=True, exist_ok=True)
    Path(model.path).write_bytes(pickle.dumps(bundle))


@dsl.component(base_image='python:3.12-slim', install_kfp_package=False)
def evaluate_model(validation: Input[Dataset], model: Input[Model], contract: Input[Artifact]) -> str:
    import csv
    import hashlib
    import json
    import pickle
    from pathlib import Path
    import numpy as np
    from sklearn.metrics import accuracy_score, f1_score
    meta = json.loads(Path(contract.path).read_text())
    raw = Path(model.path).read_bytes()
    bundle = pickle.loads(raw)  # only the model produced by this local pipeline
    if bundle['feature_names'] != meta['feature_names'] or bundle['data_sha256'] != meta['data_sha256']:
        raise ValueError('Modello e contratto non coerenti')
    with open(validation.path, newline='', encoding='utf-8') as stream:
        rows = list(csv.DictReader(stream))
    X = np.asarray([[float(r[c]) for c in meta['feature_names']] for r in rows])
    y = np.asarray([int(r['target']) for r in rows])
    pred = bundle['model'].predict(X)
    return json.dumps({'accuracy': float(accuracy_score(y, pred)),
                       'f1_macro': float(f1_score(y, pred, average='macro')),
                       'validation_rows': len(rows), 'data_sha256': meta['data_sha256'],
                       'model_sha256': hashlib.sha256(raw).hexdigest(),
                       'execution': 'Kubeflow SDK, local SubprocessRunner'})


@dsl.pipeline(name='lab12-wine-local')
def wine_pipeline(data_path: str, regularization: float = 1.) -> str:
    prepared = prepare_data(data_path=data_path)
    trained = train_model(training=prepared.outputs['training'], contract=prepared.outputs['contract'],
                          regularization=regularization)
    scored = evaluate_model(validation=prepared.outputs['validation'], model=trained.outputs['model'],
                            contract=prepared.outputs['contract'])
    return scored.output


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--data', type=Path, required=True)
    parser.add_argument('--out', type=Path, default=Path('kfp_output'))
    parser.add_argument('--compile-only', action='store_true')
    args = parser.parse_args()
    if not args.compile_only and any(char.isspace() for char in sys.executable):
        raise RuntimeError('KFP: ricrea l’ambiente in un percorso senza spazi come indicato in S1.')
    root = args.out.resolve()
    root.mkdir(parents=True, exist_ok=False)
    compiler.Compiler().compile(wine_pipeline, package_path=str(root / 'wine_pipeline.yaml'))
    if args.compile_only:
        print('YAML compilato; componenti non eseguiti.')
        return
    # All packages are prepared in this virtual environment before the lesson.
    # No cluster, cloud service, Docker image or automatic package install is requested.
    local.init(runner=local.SubprocessRunner(use_venv=False), pipeline_root=str(root / 'runs'))
    task = wine_pipeline(data_path=str(args.data.resolve()), regularization=1.)
    report = json.loads(task.output)
    (root / 'kfp_result.json').write_text(json.dumps(report, indent=2))
    print(json.dumps(report, indent=2))


if __name__ == '__main__':
    main()
```

## Riferimenti

- [MLflow Client](https://mlflow.org/docs/latest/api_reference/python_api/mlflow.client.html)
- [Esecuzione locale Kubeflow Pipelines e limiti](https://www.kubeflow.org/docs/components/pipelines/user-guides/core-functions/execute-kfp-pipelines-locally/)
