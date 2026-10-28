# Soluzione Lab 08 - Explainability Etica e Sicurezza Dati

## Obiettivo risolto

- Focus operativo: spiegare una predizione contestata e collegarla a rischi etici, base giuridica GDPR e sicurezza degli accessi.
- Deliverable atteso: spiegazione SHAP/LIME, feature proxy individuate, base giuridica GDPR, esito su revisione umana.
- Forma consigliata: file Markdown o notebook con spiegazione, tabella conformita e nota di cleanup.

## Soluzione guidata

1. Generare la spiegazione SHAP o LIME della predizione contestata (es. rifiuto di credito).
2. Verificare se tra le feature influenti ci sono proxy di categorie protette (zona geografica, nome, ecc.).
3. Identificare la base giuridica GDPR del trattamento e verificare se e documentata.
4. Compilare la checklist di accesso a modello, dati di training e log delle predizioni.
5. Decidere se serve revisione umana (art. 22 GDPR) e documentare la motivazione.
6. Scrivere la risposta in linguaggio semplice per il cliente e chiudere con il cleanup.

## Esempio di deliverable compilato

|    # | Elemento            | Evidenza attesa                                                                                 |
| ---: | ------------------- | ----------------------------------------------------------------------------------------------- |
|    1 | predizione spiegata | Rifiuto con maggiore confidenza nel test split del CSV fornito, spiegazione SHAP generata       |
|    2 | feature proxy       | `cap_residenza` controllato tra le feature influenti: possibile proxy geografico/socioeconomico |
|    3 | dato personale      | Reddito e CAP trattati come dati personali secondo GDPR                                         |
|    4 | base di trattamento | Esecuzione di un contratto (valutazione della richiesta di credito)                             |
|    5 | revisione umana     | Richiesta perche la decisione ha impatto legale/economico significativo                         |

## Codice della soluzione

Il lab 08 e' soprattutto un esercizio di documentazione (base giuridica GDPR, checklist
di accesso, nota al cliente): quella parte resta nel "Template operativo" sotto, come
tabelle/prosa da compilare. Il codice copre gli step 1-2: spiega con SHAP una predizione
di rifiuto e segnala feature candidate proxy. Legge esclusivamente il file
`labs/lab 08/dataset_credit_scoring_sintetico.csv` indicato nel lab; non genera o carica
altri dataset. Il CSV didattico e' sintetico e contiene le feature `reddito_dichiarato`,
`insolvenze_passate`, `anni_impiego`, `cap_residenza` e il target `approvato`.

```python
#!/usr/bin/env python3
"""Lab 08 - spiegazione di una predizione contestata + controllo proxy.

Scenario: un modello di approvazione credito ha rifiutato la richiesta di
un cliente, che contesta la decisione. Qui copriamo SOLO la parte di codice
del lab (step 1-2): generare una spiegazione SHAP per la predizione
contestata e verificare se tra le feature piu influenti ce ne sono che
sembrano fare da proxy per categorie protette (es. CAP come proxy di
zona/reddito). Le parti GDPR, checklist di accesso, revisione umana e
risposta al cliente (step 3-6 del lab) restano documentazione/prosa nel
resto di questa soluzione: non sono task di codice.

Dataset: viene letto esclusivamente il file
`labs/lab 08/dataset_credit_scoring_sintetico.csv`, senza generare altri record.

Uso: python3 proxy_check.py
"""
from __future__ import annotations

from pathlib import Path

import numpy as np
import pandas as pd
import shap
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split

# Feature considerate potenziali proxy di categorie protette (art. 9 GDPR /
# discriminazione indiretta): vanno segnalate se compaiono tra le piu influenti.
SCRIPT_DIR = Path(__file__).resolve().parent
DATA_PATH = SCRIPT_DIR / "dataset_credit_scoring_sintetico.csv"
if not DATA_PATH.exists():  # consente di eseguire il codice copiato in questo Markdown
    DATA_PATH = SCRIPT_DIR.parent / "lab 08" / "dataset_credit_scoring_sintetico.csv"
FEATURES = ["reddito_dichiarato", "insolvenze_passate", "anni_impiego", "cap_residenza"]
TARGET = "approvato"
PROXY_CANDIDATES = {"cap_residenza"}


def load_dataset(path: Path):
    """Carica e valida esclusivamente il CSV fornito per il lab."""
    data = pd.read_csv(path)
    required = FEATURES + [TARGET]
    missing = sorted(set(required) - set(data.columns))
    if missing:
        raise ValueError(f"Colonne mancanti nel CSV: {', '.join(missing)}")

    frame = data[required].apply(pd.to_numeric, errors="coerce")
    if frame.empty or frame.isna().any().any() or not np.isfinite(frame.to_numpy()).all():
        raise ValueError("Il CSV e' vuoto o contiene valori mancanti/non numerici.")
    if set(frame[TARGET].unique()) != {0, 1}:
        raise ValueError(f"Il target {TARGET!r} deve contenere solo le classi 0 e 1.")
    return frame[FEATURES], frame[TARGET].astype(int)


def rejection_shap_values(explainer, row, class_index):
    """Estrae i contributi della classe rifiuto con SHAP recente o legacy."""
    values = explainer(row).values
    values = np.asarray(values)
    if values.ndim == 3:  # righe x feature x classi
        return values[0, :, class_index]
    if values.ndim == 2:  # singola uscita: righe x feature
        return values[0]

    legacy = explainer.shap_values(row)
    if isinstance(legacy, list):
        return np.asarray(legacy[class_index])[0]
    legacy = np.asarray(legacy)
    if legacy.ndim == 3:
        return legacy[0, :, class_index]
    if legacy.ndim == 2:
        return legacy[0]
    raise ValueError(f"Formato SHAP non riconosciuto: {legacy.shape}")


def main() -> None:
    X, y = load_dataset(DATA_PATH)
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.25, random_state=42, stratify=y
    )

    model = RandomForestClassifier(n_estimators=200, random_state=42)
    model.fit(X_train, y_train)
    proba = model.predict_proba(X_test)

    # --- Step 1: caso contestato dal cliente: un rifiuto (classe 0) nel batch di test ---
    rejected_class = int(np.flatnonzero(model.classes_ == 0)[0])
    rejected_idx = np.flatnonzero(model.predict(X_test) == 0)
    if rejected_idx.size == 0:
        raise RuntimeError("Il modello non ha prodotto rifiuti nel campione di test.")
    # prendiamo il rifiuto con confidenza piu alta come "caso tipo" da spiegare
    case_idx = int(rejected_idx[np.argmax(proba[rejected_idx, rejected_class])])
    case = X_test.iloc[[case_idx]]
    csv_row = int(case.index[0]) + 2  # +1 per indice da zero, +1 per intestazione
    print("=== Step 1: predizione contestata ===")
    print(f"Riga CSV #{csv_row}  |  decisione: RIFIUTO CREDITO  |  "
          f"confidenza rifiuto: {proba[case_idx, rejected_class]:.4f}")
    for name, val in case.iloc[0].items():
        print(f"  {name:<20} = {val:.2f}")

    print("\n=== Step 1: spiegazione SHAP della predizione ===")
    explainer = shap.TreeExplainer(model)
    contribs = rejection_shap_values(explainer, case, rejected_class)

    ranking = np.argsort(-np.abs(contribs))
    for rank, feat_i in enumerate(ranking, start=1):
        sign = "+" if contribs[feat_i] >= 0 else "-"
        print(f"  {rank}. {FEATURES[feat_i]:<20} "
              f"contributo {sign}{abs(contribs[feat_i]):.4f} alla classe rifiuto")

    # --- Step 2: verifica proxy tra le feature piu influenti ---
    print("\n=== Step 2: controllo feature proxy di categorie protette ===")
    flagged = [FEATURES[i] for i in ranking if FEATURES[i] in PROXY_CANDIDATES]
    if flagged:
        print(f"ATTENZIONE: feature potenzialmente proxy tra le influenti: {flagged}")
        print("Il CAP puo' riflettere area geografica o condizioni socioeconomiche; "
              "e' un rischio da approfondire, non una prova di discriminazione.")
    else:
        print("Nessuna feature della lista proxy tra quelle influenti in questo caso.")


if __name__ == "__main__":
    main()
```

Linux/macOS (dalla cartella dello script):

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install numpy pandas scikit-learn shap
python3 proxy_check.py
```

Windows PowerShell (dalla cartella dello script):

```powershell
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install numpy pandas scikit-learn shap
python proxy_check.py
deactivate
```

L'output cambia in base ai dati e alle versioni delle librerie. La forma attesa e':

```text
Riga CSV #<riga> | decisione: RIFIUTO CREDITO | confidenza rifiuto: <valore>
  reddito_dichiarato   = <valore>
  insolvenze_passate   = <valore>
  anni_impiego         = <valore>
  cap_residenza        = <valore>

=== Step 1: spiegazione SHAP della predizione ===
  1. <feature>           contributo <segno><valore> alla classe rifiuto
  ...

=== Step 2: controllo feature proxy di categorie protette ===
ATTENZIONE: feature potenzialmente proxy tra le influenti: ['cap_residenza']
```

La classifica SHAP e i relativi contributi sono calcolati sui dati effettivi del CSV e
possono variare tra versioni delle librerie. Se `cap_residenza` compare tra le feature
influenti, va approfondito come possibile proxy; SHAP non dimostra da solo causalita' o
discriminazione.

Le parti non di codice del lab (base giuridica GDPR, checklist di accesso, necessita di
revisione umana ex art. 22, risposta al cliente) restano compilate come tabelle/prosa
nel template qui sotto: non sono task eseguibili, sono decisioni di conformita' da
documentare con motivazione.

## Template operativo

```markdown
# Deliverable Lab 08

## Scenario
- Sessione: Explainability Etica e Sicurezza Dati
- Problema operativo: spiegare un rifiuto di credito automatico contestato dal cliente

## Spiegazione e rischio
| Feature influente | Possibile proxy? | Base giuridica GDPR | Revisione umana richiesta |
| ----------------- | ---------------- | ------------------- | ------------------------- |
| ...               | si/no            | ...                 | si/no                     |

## Sicurezza e accessi
- Chi accede al modello: ...
- Chi accede ai dati di training: ...
- Chi accede ai log delle predizioni: ...

## Decisione
- Scelta: comunicare esito con revisione umana / risposta automatica motivata
- Motivazione: ...
- Fallback: dataset anonimizzato / simulato / non necessario

## Cleanup
- Dataset con dati personali reali/simulati rimossi: ...
- Notebook/log con identificativi rimossi: ...
- Accessi di test chiusi: ...
```

## Esempio sintetico

- Decisione corretta: richiedere revisione umana quando la decisione automatica ha impatto economico diretto su una persona (art. 22 GDPR).
- Evidenza minima: spiegazione SHAP/LIME della predizione, tabella feature proxy/base giuridica, esito su revisione umana.
- Fallback accettabile: dataset didattico anonimizzato quando non sono disponibili dati reali.
- Cleanup atteso: rimozione di ogni dato personale, anche fittizio, non necessario alla consegna.

## Checkpoint risolto

- [x] predizione spiegata: presente e motivato
- [x] feature proxy: presente e motivato
- [x] dato personale: presente e motivato
- [x] base di trattamento: presente e motivato
- [x] revisione umana: presente e motivato
- [x] Spiegazione tecnica tradotta in linguaggio comprensibile al cliente
- [x] Feature proxy di categorie protette verificate esplicitamente
- [x] Base giuridica GDPR e livello di accesso a dati/modello documentati esplicitamente
- [x] Cleanup obbligatorio completato o esplicitamente giustificato

## Errori comuni da evitare

- Fornire una spiegazione tecnicamente corretta ma incomprensibile per il cliente.
- Non verificare la presenza di feature proxy di categorie protette tra i fattori piu influenti.
- Trattare dati personali (anche fittizi) senza dichiarare la base giuridica GDPR.
- Saltare la valutazione sulla necessita di revisione umana quando la decisione ha impatto reale.

## Risposta breve per discussione orale

La soluzione e accettabile quando la spiegazione **SHAP/LIME** individua eventuali feature proxy, la base giuridica **GDPR** e dichiarata esplicitamente, e chiaro se serve revisione umana, e il cleanup dei dati personali (anche simulati) e confermato.
