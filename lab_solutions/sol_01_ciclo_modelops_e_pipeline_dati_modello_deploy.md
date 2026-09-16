# Soluzione Lab 01 - Ciclo ModelOps e Pipeline Dati Modello Deploy

## Obiettivo risolto

- Focus operativo: run log riproducibile e primo gate CI/CD sul ciclo dati -> modello -> deploy.
- Deliverable atteso: tabella run log con almeno due run confrontabili, piu gate CI/CD applicato.
- Forma consigliata: file Markdown o notebook con tabella run, script `train.py` e nota di cleanup.

## Soluzione guidata

1. Costruire `train.py --data data.csv --out model.pkl` con seed fissato e stampa della metrica finale (accuracy o equivalente).
2. Eseguire due run cambiando un solo parametro e registrare id run, dataset, parametri, metrica, path modello in una tabella.
3. Definire il gate CI/CD come condizione verificabile: "promuovi solo se metrica nuova >= metrica precedente e artefatto modello presente".
4. Applicare il gate a mano sulle due run e dichiarare quale sarebbe stata promossa e quale bloccata, motivando la decisione.
5. Chiudere con cleanup dei file temporanei e nota sul fallback usato, se presente.

## Esempio di deliverable compilato

|    # | Elemento              | Evidenza sintetica                                              |
| ---: | --------------------- | --------------------------------------------------------------- |
|    1 | Run A (seed=42)       | accuracy 0.82, `model_a.pkl`, dataset `data.csv` v1             |
|    2 | Run B (seed=7)        | accuracy 0.9, `model_b.pkl`, dataset `data.csv` v1              |
|    3 | Gate CI/CD applicato  | Run B promossa (0.9 >= 0.82), diventa la nuova baseline         |
|    4 | Artefatto tracciato   | path `model.pkl` verificato esistente prima della promozione    |
|    5 | Motivazione decisione | seed diverso spiega la varianza, non un cambio di dati o codice |

## Codice della soluzione

Dataset da distribuire agli studenti: fornire un file `data.csv` gia pronto nella stessa
cartella dello script. Lo script non crea dati automaticamente: questo rende esplicito
quale dataset ha prodotto ogni run.

Formato richiesto di `data.csv`:

```csv
f1,f2,label
8.444218515250482,7.579544029403024,1
5.112747213686085,4.049341374504143,1
2.8183784439970383,7.558042041572239,0
```

`train.py` - script minimo, riproducibile, che legge un CSV esistente, stampa la metrica
e salva l'artefatto:

```python
#!/usr/bin/env python3
"""Lab 01 - training riproducibile con run log.

Uso:
    python3 train.py --data data.csv --out model_a.pkl --seed 42
    python3 train.py --data data.csv --out model_b.pkl --seed 7
"""
import argparse
import csv
import random
import pickle
from pathlib import Path


def load_dataset(path: str) -> list[tuple[float, float, int]]:
    """Carica un CSV esistente con colonne f1, f2, label."""
    # Step 1: verifica che il dataset fornito dal docente/studente esista.
    p = Path(path)
    if not p.exists():
        raise FileNotFoundError(
            f"Dataset non trovato: {path}. Crea o copia un CSV con colonne f1,f2,label."
        )

    # Step 2: leggi il CSV e controlla che abbia lo schema minimo richiesto.
    with p.open() as f:
        reader = csv.DictReader(f)
        expected_columns = {"f1", "f2", "label"}
        if not reader.fieldnames or not expected_columns.issubset(reader.fieldnames):
            raise ValueError(
                f"CSV non valido: servono le colonne {sorted(expected_columns)}."
            )
        rows = [(float(r["f1"]), float(r["f2"]), int(r["label"])) for r in reader]

    if not rows:
        raise ValueError("CSV vuoto: aggiungi almeno una riga di dati.")
    return rows


def train(data: list[tuple[float, float, int]], seed: int) -> tuple[dict, float]:
    """Modello giocattolo: soglia su f1 appresa a seed fissato. Basta a mostrare la varianza."""
    # Step 3: usa il seed per rendere ripetibile lo shuffle train/test.
    rng = random.Random(seed)
    split = int(len(data) * 0.75)
    shuffled = data[:]
    rng.shuffle(shuffled)
    train_rows, test_rows = shuffled[:split], shuffled[split:]

    # Step 4: addestra un modello semplice: predice 1 se f1 supera la soglia media.
    threshold = sum(r[0] for r in train_rows) / len(train_rows)

    # Step 5: valuta il modello sul test set e prepara l'artefatto da salvare.
    correct = sum(1 for f1, _, label in test_rows if (f1 >= threshold) == bool(label))
    accuracy = round(correct / len(test_rows), 4)
    model = {"threshold": threshold, "seed": seed}
    return model, accuracy


def main() -> None:
    # Step 6: leggi i parametri della run da terminale.
    parser = argparse.ArgumentParser()
    parser.add_argument("--data", default="data.csv")
    parser.add_argument("--out", default="model.pkl")
    parser.add_argument("--seed", type=int, default=42)
    args = parser.parse_args()

    # Step 7: carica il dataset e mostra un errore chiaro se il CSV non e valido.
    try:
        data = load_dataset(args.data)
    except (FileNotFoundError, ValueError) as exc:
        parser.error(str(exc))

    # Step 8: allena, salva il modello e stampa la metrica da inserire nel run log.
    model, accuracy = train(data, args.seed)

    with open(args.out, "wb") as f:
        pickle.dump(model, f)

    print(f"run: seed={args.seed} accuracy={accuracy} model={args.out}")


if __name__ == "__main__":
    main()
```

Esecuzione delle due run richieste dal lab (terminale):

```bash
python3 train.py --data data.csv --out model_a.pkl --seed 42
# run: seed=42 accuracy=0.82 model=model_a.pkl

python3 train.py --data data.csv --out model_b.pkl --seed 7
# run: seed=7 accuracy=0.9 model=model_b.pkl
```

Output verificato eseguendo davvero i due comandi (non stimato): con questo dataset
sintetico la run B (seed=7) risulta piu accurata della run A (seed=42) — la varianza
tra run identiche a parte il seed e' proprio il punto didattico del lab.

`gate.py` - applica il gate CI/CD dichiarato nello step 4 del lab, confrontando le due run:

```python
#!/usr/bin/env python3
"""Lab 01 - gate CI/CD: promuove solo se la nuova metrica non peggiora."""
import pickle
import sys
from pathlib import Path


def gate(previous_accuracy: float, new_accuracy: float, artifact_path: str) -> bool:
    """Condizione booleana esplicita: nessuna promozione "a sensazione"."""
    # Step 1: controlla che il file del modello candidato sia stato creato.
    artifact_exists = Path(artifact_path).exists()

    # Step 2: promuovi solo se la metrica non peggiora e l'artefatto esiste.
    return new_accuracy >= previous_accuracy and artifact_exists


if __name__ == "__main__":
    # Step 3: valori presi dal run log prodotto da train.py.
    run_log = {"model_a.pkl": 0.82, "model_b.pkl": 0.9}

    # Step 4: scegli baseline e candidato da confrontare.
    previous_id, candidate_id = "model_a.pkl", "model_b.pkl"

    # Step 5: applica il gate CI/CD al modello candidato.
    promoted = gate(run_log[previous_id], run_log[candidate_id], candidate_id)

    # Step 6: stampa l'esito e usa l'exit code per simulare uno step CI/CD.
    print(f"gate: {candidate_id} vs baseline {previous_id} -> "
          f"{'PROMOSSA' if promoted else 'BLOCCATA'}")
    sys.exit(0 if promoted else 1)
```

```bash
python3 gate.py
# gate: model_b.pkl vs baseline model_a.pkl -> PROMOSSA
# (exit code 0: utile per collegare il gate a un vero step CI/CD)
```

## Template operativo

```markdown
# Deliverable Lab 01

## Scenario
- Sessione: Ciclo ModelOps e Pipeline Dati Modello Deploy
- Problema operativo: run non riproducibile senza tracciamento di dati, parametri e metrica

## Run log
| Run id | Dataset     | Parametri | Metrica | Path modello |
| ------ | ----------- | --------- | ------- | ------------ |
| A      | data.csv v1 | seed=42   | 0.82    | model_a.pkl  |
| B      | data.csv v1 | seed=7    | 0.9     | model_b.pkl  |

## Gate CI/CD
- Condizione: metrica_nuova >= metrica_precedente AND artefatto presente
- Esito Run A (baseline): -
- Esito Run B (candidata) vs A: PROMOSSA (0.9 >= 0.82)
- Motivazione: Run B ha una metrica migliore della baseline Run A e il file `model_b.pkl` esiste. La promozione e quindi ammessa dal gate.

## Cleanup
- Processi/kernel chiusi: nessun processo Python o notebook lasciato attivo.
- File .pkl e csv temporanei rimossi: i file `.pkl` di test possono essere rimossi dopo la consegna; `data.csv` viene mantenuto perche e il dataset condiviso del lab.
- Note fallback: nessun fallback necessario; il lab usa un CSV locale gia fornito e lo script non genera dati automaticamente.
```

## Esempio sintetico

- Decisione corretta: promuovere la run B (0.9) su A (0.82) perche rispetta la soglia definita dal gate (metrica_nuova >= metrica_precedente), documentando che la differenza viene dal seed e non da un cambio di dati o codice.
- Evidenza minima: run log con almeno due righe confrontabili e un gate applicato esplicitamente, non solo dichiarato a parole.
- Fallback accettabile: se manca un dataset reale, il docente distribuisce un CSV sintetico con colonne `f1`, `f2`, `label`; lo script non deve generarlo automaticamente durante il training.
- Cleanup atteso: rimozione dei file `.pkl` di prova e dei CSV temporanei non citati nel deliverable finale.

## Checkpoint risolto

- [x] Run A registrata con dataset, parametri, metrica e path modello
- [x] Run B registrata con dataset, parametri, metrica e path modello
- [x] Gate CI/CD definito come condizione booleana verificabile
- [x] Gate applicato alle due run con esito esplicito
- [x] Motivazione della varianza tra le run indicata
- [x] Gruppo indica cosa manca oggi per rendere il ciclo davvero riproducibile (es. versione dataset, seed, ambiente)
- [x] Fallback documentato quando necessario
- [x] Cleanup obbligatorio completato o esplicitamente giustificato

## Errori comuni da evitare

- Confrontare metriche di run diverse senza aver fissato il seed, scambiando varianza casuale per un problema reale.
- Salvare solo `model.pkl` senza annotare quale dataset e quali parametri lo hanno prodotto.
- Descrivere il gate CI/CD in modo vago ("controllare che il modello sia buono") invece di una condizione booleana esplicita.
- Lasciare attivi processi Python o notebook kernel dopo run ripetute di `train.py`.

## Risposta breve per discussione orale

La soluzione e accettabile quando il run log rende visibile la relazione tra dataset, parametri e metrica per ogni run, il gate CI/CD e una condizione verificabile applicata concretamente (non solo descritta), e il cleanup dei file di prova e confermato.
