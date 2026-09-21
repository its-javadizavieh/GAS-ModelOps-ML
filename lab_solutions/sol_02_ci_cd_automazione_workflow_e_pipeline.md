# Soluzione Lab 02 - CI CD Automazione Workflow e Pipeline

## Obiettivo risolto

- Focus operativo: pipeline a stadi (validate, train, gate) con controlli automatici che bloccano in caso di fallimento.
- Deliverable atteso: log degli stadi eseguiti, codice/pseudo-codice dei 3 stadi, prova del blocco su fallimento simulato.
- Forma consigliata: file Markdown o notebook con log stadi, script pipeline e nota di cleanup.

## Soluzione guidata

1. Scrivere un test `pytest` che verifica colonne attese e assenza di null nel target del dataset.
2. Implementare `pipeline.py --stage validate` che esegue il test e interrompe la pipeline con messaggio chiaro se fallisce.
3. Aggiungere `--stage train` condizionato al successo dello stadio precedente, con stampa della metrica.
4. Aggiungere `--stage gate` che confronta la metrica con una soglia esplicita (es. 0.75) e decide la promozione.
5. Eseguire la sequenza completa, poi simulare un fallimento nello stadio di validazione e verificare che il training non parta.

## Esempio di deliverable compilato

|    # | Elemento                | Evidenza sintetica                                                |
| ---: | ------------------------ | -------------------------------------------------------------------- |
|    1 | Stadio validate (pass)   | `pytest tests/` -> 3 test passati, colonne verificate                 |
|    2 | Stadio train             | accuracy 0.86 sul dataset copiato in Lab 02, `model.pkl` prodotto      |
|    3 | Stadio gate              | soglia 0.75 superata (0.86 >= 0.75) -> artefatto promosso              |
|    4 | Fallimento simulato      | colonna target rimossa -> validate FAIL, train non eseguito           |
|    5 | Log stadi                | sequenza validate -> train -> gate con esito annotato per ciascuno    |

## Codice della soluzione

Struttura minima usata per verificare davvero la sequenza a stadi:

```
lab 02/
  data.csv           # copia del dataset fornito nel Lab 01
  pipeline.py        # stadi validate / train / gate
  tests/
    test_data.py     # test pytest sullo schema dati (Step 1)
```

`tests/test_data.py` - test minimo richiesto dallo Step 1 del lab (colonne attese, nessun null nel target):

```python
"""Lab 02 - test minimo sullo schema del dataset (Step 1 del lab)."""
import csv
from pathlib import Path

DATA_PATH = Path(__file__).parent.parent / "data.csv"
EXPECTED_COLUMNS = {"f1", "f2", "label"}


def _read_rows():
    with DATA_PATH.open() as f:
        return list(csv.DictReader(f))


def test_dataset_exists():
    assert DATA_PATH.exists(), f"dataset non trovato: {DATA_PATH}"


def test_expected_columns():
    rows = _read_rows()
    assert set(rows[0].keys()) == EXPECTED_COLUMNS


def test_no_null_in_target():
    rows = _read_rows()
    assert all(r["label"] not in (None, "", "NA") for r in rows)
```

`pipeline.py` - i tre stadi richiesti dagli Step 2-4, ciascuno un gate esplicito. Lo stadio
`validate` lancia `pytest` come sottoprocesso e usa il suo exit code per decidere se
fermarsi; `train` presuppone che `validate` sia gia passato; `gate` confronta la
metrica prodotta da `train` con una soglia numerica fissa:

```python
#!/usr/bin/env python3
"""Lab 02 - pipeline a stadi con controlli automatici (validate, train, gate).

Uso:
    python3 pipeline.py --stage validate
    python3 pipeline.py --stage train
    python3 pipeline.py --stage gate
"""
import argparse
import csv
import pickle
import subprocess
import sys
from pathlib import Path

BASE_DIR = Path(__file__).parent
DATA_PATH = BASE_DIR / "data.csv"
MODEL_PATH = BASE_DIR / "model.pkl"
METRIC_PATH = BASE_DIR / "metric.txt"
GATE_THRESHOLD = 0.75


def stage_validate() -> int:
    """Esegue i test pytest sullo schema dati (Step 1). Blocca se falliscono."""
    result = subprocess.run(
        [sys.executable, "-m", "pytest", str(BASE_DIR / "tests"), "-q"],
        cwd=BASE_DIR,
    )
    if result.returncode != 0:
        print("[VALIDATE] FAIL: dataset non valido, la pipeline si ferma qui.")
        return 1
    print("[VALIDATE] PASS: schema dati verificato.")
    return 0


def stage_train() -> int:
    """Allena un modello a soglia e stampa/salva la metrica. Presuppone validate passato."""
    with DATA_PATH.open() as f:
        rows = [(float(r["f1"]), float(r["f2"]), int(r["label"]))
                for r in csv.DictReader(f)]

    split = int(len(rows) * 0.75)
    train_rows, test_rows = rows[:split], rows[split:]

    mean_f1 = sum(r[0] for r in train_rows) / len(train_rows)
    mean_f2 = sum(r[1] for r in train_rows) / len(train_rows)

    def predict(f1: float, f2: float) -> int:
        return 1 if (f1 - mean_f1) - 0.3 * (f2 - mean_f2) > 0 else 0

    correct = sum(1 for f1, f2, label in test_rows if predict(f1, f2) == label)
    accuracy = round(correct / len(test_rows), 4)

    with MODEL_PATH.open("wb") as f:
        pickle.dump({"mean_f1": mean_f1, "mean_f2": mean_f2}, f)
    METRIC_PATH.write_text(str(accuracy), encoding="utf-8")

    print(f"[TRAIN] accuracy={accuracy} model={MODEL_PATH.name}")
    return 0


def stage_gate() -> int:
    """Confronta la metrica con la soglia fissa (Step 4). Decide la promozione."""
    if not METRIC_PATH.exists():
        print("[GATE] FAIL: nessuna metrica trovata, esegui prima --stage train.")
        return 1

    accuracy = float(METRIC_PATH.read_text(encoding="utf-8"))
    promoted = accuracy >= GATE_THRESHOLD

    print(f"[GATE] accuracy={accuracy} soglia={GATE_THRESHOLD} -> "
          f"{'PROMOSSO' if promoted else 'BLOCCATO'}")
    return 0 if promoted else 1


STAGES = {"validate": stage_validate, "train": stage_train, "gate": stage_gate}


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--stage", choices=sorted(STAGES), required=True)
    args = parser.parse_args()
    return STAGES[args.stage]()


if __name__ == "__main__":
    sys.exit(main())
```

Sequenza completa richiesta dallo Step 5 (output verificato eseguendo davvero i comandi):

```bash
python3 pipeline.py --stage validate
# ...
# 3 passed in 0.00s
# [VALIDATE] PASS: schema dati verificato.

python3 pipeline.py --stage train
# [TRAIN] accuracy=0.86 model=model.pkl

python3 pipeline.py --stage gate
# [GATE] accuracy=0.86 soglia=0.75 -> PROMOSSO
```

Fallimento simulato richiesto dallo Step 6 (colonna `label` rimossa da `data.csv`),
eseguito con la sequenza incatenata da `&&` cosi il secondo stadio non parte se il
primo fallisce:

```bash
python3 pipeline.py --stage validate && python3 pipeline.py --stage train
# ...
# FAILED tests/test_data.py::test_expected_columns - AssertionError: assert {'f1', 'f2'} == {'f1', 'f2', 'label'}
# FAILED tests/test_data.py::test_no_null_in_target - KeyError: 'label'
# 2 failed, 1 passed in 0.01s
# [VALIDATE] FAIL: dataset non valido, la pipeline si ferma qui.
# (exit code 1: "&&" impedisce che "--stage train" venga anche solo lanciato)
```

Output verificato eseguendo davvero i comandi (non stimato): con schema valido la
pipeline arriva fino al gate e lo supera (accuracy 0.86 su soglia 0.75); con la
colonna `label` rimossa, `pytest` fallisce 2 test su 3 e lo stadio `validate` ritorna
exit code 1, che blocca l'esecuzione dello stadio `train` grazie a `&&`.

## Template operativo

```markdown
# Deliverable Lab 02

## Scenario

- Sessione: CI CD Automazione Workflow e Pipeline
- Problema operativo: lancio manuale senza controlli, rischio di promuovere modelli non validati

## Log stadi

| Stadio   | Comando                          | Esito | Note                        |
| -------- | -------------------------------- | ----- | --------------------------- |
| validate | pytest tests/                    | PASS  | colonne e null verificati   |
| train    | python pipeline.py --stage train | PASS  | accuracy 0.86               |
| gate     | python pipeline.py --stage gate  | PASS  | soglia 0.75 superata (0.86) |

## Fallimento simulato

- Modifica applicata: colonna target rimossa temporaneamente da `labs/lab 02/data.csv`
- Stadio bloccato: validate (`2 failed, 1 passed`, exit code 1)
- Comportamento verificato: train e gate non sono stati eseguiti

## Cleanup

- File test/dataset temporanei rimossi: rimossi `model.pkl`, `metric.txt` e cache; `labs/lab 02/data.csv` ripristinato
- Processi pipeline fermati: nessun processo Python o watcher rimasto attivo
- Note fallback: nessun fallback usato; esecuzione locale completata con la virtual environment richiesta

## Comandi provati

~~~bash
cd "/Desktop/ModelOps e Machine Learning/labs/lab 02"
VENV_PYTHON="/Desktop/ModelOps e Machine Learning/labs/.venv/bin/python"

"$VENV_PYTHON" --version
"$VENV_PYTHON" -m pytest tests/ -q
"$VENV_PYTHON" pipeline.py --stage validate
"$VENV_PYTHON" pipeline.py --stage train
"$VENV_PYTHON" pipeline.py --stage gate
~~~

Output osservato: Python 3.12.3, 3 test passati, accuracy 0.86 e gate superato
con soglia 0.75. Il dataset usato contiene 200 righe e le colonne `f1`, `f2`,
`label`, senza valori nulli nel target.
```

## Esempio sintetico

- Decisione corretta: bloccare la pipeline allo stadio di validazione quando il dataset non rispetta lo schema atteso, senza proseguire al training.
- Evidenza minima: log con tre stadi e relativo esito, piu una prova concreta che il blocco funziona davvero (non solo dichiarato).
- Fallback accettabile: se manca un ambiente CI reale, eseguire gli stadi manualmente in sequenza e documentare l'ordine e l'esito.
- Cleanup atteso: rimozione di dataset modificati per il test di fallimento e dei modelli generati durante le prove ripetute.

## Checkpoint risolto

- [x] Test pytest su schema dati definito e funzionante
- [x] Stadio validate blocca correttamente in caso di dati non validi
- [x] Stadio train esegue solo dopo validate passato
- [x] Stadio gate applica una soglia numerica esplicita
- [x] Fallimento simulato documentato con esito coerente
- [x] Gruppo indica quale stadio aggiungerebbe per primo in un progetto reale (es. controllo drift, test su modello serializzato)
- [x] Fallback documentato quando necessario
- [x] Cleanup obbligatorio completato o esplicitamente giustificato

## Errori comuni da evitare

- Scrivere stadi che non si bloccano davvero a vicenda (es. train parte anche se validate fallisce).
- Usare una soglia del gate non giustificata da un baseline o da un criterio esplicito.
- Confondere l'esecuzione riuscita di un singolo comando con una pipeline funzionante end-to-end.
- Lasciare dataset modificati per il test di fallimento mescolati con quelli reali nel repository.

## Risposta breve per discussione orale

La soluzione e accettabile quando la pipeline e composta da stadi con dipendenza esplicita (validate -> train -> gate), il blocco su fallimento e dimostrato con un caso concreto, e il cleanup dei file di prova e confermato.
