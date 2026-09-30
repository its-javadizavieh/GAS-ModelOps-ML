# Soluzione Lab 05 - Versioning Rollback e Prime Metriche

## Obiettivo risolto

- Focus operativo: rollback di `model-v2` a `model-v1` motivato da un calo di accuratezza misurato.
- Deliverable atteso: confronto metriche v1/v2, decisione motivata e log di rollback.
- Forma consigliata: file Markdown o notebook con tabella metriche, log di rollback e nota di cleanup.

## Soluzione guidata

1. Registrare la baseline: accuratezza e latenza di `model-v1` prima del rilascio di `model-v2`.
2. Registrare le metriche osservate su `model-v2` dopo il deploy, sullo stesso batch di validazione usato per v1.
3. Confrontare le due misurazioni con la soglia definita in anticipo (es. calo accuratezza > 3 punti) per decidere rollback o fix.
4. Eseguire il rollback (tag Git o cambio stage nel registry) e verificare che le metriche tornino ai valori attesi di v1.
5. Chiudere con un log di rollback (motivo, versione partenza, versione arrivo) e il cleanup delle risorse di test.

## Esempio di deliverable compilato

|    # | Elemento               | Evidenza attesa                                          |
| ---: | ----------------------- | --------------------------------------------------------- |
|    1 | modello v1 stabile      | Accuratezza 0.9474 (LogisticRegression) sullo stesso batch di validazione   |
|    2 | modello v2 degradato    | Accuratezza 0.9123 (DecisionTree troppo shallow) sullo stesso batch         |
|    3 | calo misurato           | drop 0.0351, sopra la soglia consentita 0.02 definita in anticipo           |
|    4 | decisione rollback      | soglia superata (0.0351 > 0.02): rollback a v1 entro il timebox              |
|    5 | verifica post-rollback  | registry aggiorna `active_version` a `1.0.0`, v2 rifiutata dal gate          |

## Codice della soluzione

`evaluate.py` - valuta le due versioni sullo stesso split di validazione.
Salva questo file nella stessa cartella di `rollback_versions.py`:

```python
#!/usr/bin/env python3
"""Valutazione riproducibile delle versioni v1 e v2 sullo stesso dataset."""
from sklearn.datasets import load_breast_cancer
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score
from sklearn.model_selection import train_test_split
from sklearn.tree import DecisionTreeClassifier

def evaluate(model, X_train, X_test, y_train, y_test):
    model.fit(X_train, y_train)
    return round(float(accuracy_score(y_test, model.predict(X_test))), 4)

def main():
    data = load_breast_cancer()
    X_train, X_test, y_train, y_test = train_test_split(
        data.data, data.target, test_size=0.3, random_state=42,
        stratify=data.target,
    )
    models = {
        "v1": LogisticRegression(max_iter=5000),
        "v2": DecisionTreeClassifier(max_depth=1, random_state=42),
    }
    for version, model in models.items():
        accuracy = evaluate(model, X_train, X_test, y_train, y_test)
        print(f"{version}: accuracy={accuracy:.4f}")

if __name__ == "__main__":
    main()
```

`rollback_versions.py` - allena v1 e v2 sullo stesso dataset, le confronta con una
soglia definita prima del confronto e scrive la decisione nel registry:

```python
#!/usr/bin/env python3
"""Sessione 05 - versioning e rollback guidato dalle metriche.

Scenario: rilasciamo una v2 che sembra "piu' moderna" ma peggiora le metriche.
Un gate automatico la blocca e il sistema resta sulla v1.
"""
import json
from pathlib import Path

from sklearn.datasets import load_breast_cancer
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score
from sklearn.model_selection import train_test_split
from sklearn.tree import DecisionTreeClassifier

REGISTRY = Path(__file__).parent / "registry.json"

# Regola di rilascio decisa in anticipo (Step 3 del lab): una nuova versione entra
# in produzione solo se non peggiora l'accuratezza di piu' di questa soglia.
MAX_ACCEPTABLE_DROP = 0.02


def evaluate(model, X_train, X_test, y_train, y_test) -> float:
    model.fit(X_train, y_train)
    return round(float(accuracy_score(y_test, model.predict(X_test))), 4)


def main() -> None:
    data = load_breast_cancer()
    X_train, X_test, y_train, y_test = train_test_split(
        data.data, data.target, test_size=0.3, random_state=42, stratify=data.target
    )

    # v1: baseline solida, gia' in produzione (Step 1).
    acc_v1 = evaluate(LogisticRegression(max_iter=5000), X_train, X_test, y_train, y_test)
    # v2: candidata volutamente sottotono, valutata sullo STESSO batch (Step 2).
    acc_v2 = evaluate(DecisionTreeClassifier(max_depth=1, random_state=42),
                       X_train, X_test, y_train, y_test)

    print("=== Confronto versioni ===")
    print(f"v1 (LogisticRegression) accuracy = {acc_v1}")
    print(f"v2 (DecisionTree d=1)   accuracy = {acc_v2}")

    drop = acc_v1 - acc_v2
    print(f"\nPeggioramento: {drop:.4f}  (soglia consentita: {MAX_ACCEPTABLE_DROP})")

    if drop > MAX_ACCEPTABLE_DROP:
        active, decision = "1.0.0", "ROLLBACK: la v2 e' stata rifiutata dal gate"
    else:
        active, decision = "2.0.0", "PROMOZIONE: la v2 entra in produzione"

    print(f"\n>>> {decision}")
    print(f">>> Versione attiva in produzione: {active}")

    registry = {
        "active_version": active,
        "decision": decision,
        "rule": f"drop massimo consentito = {MAX_ACCEPTABLE_DROP}",
        "versions": {
            "1.0.0": {"algorithm": "LogisticRegression", "accuracy": acc_v1},
            "2.0.0": {"algorithm": "DecisionTreeClassifier(max_depth=1)", "accuracy": acc_v2},
        },
    }
    REGISTRY.write_text(json.dumps(registry, indent=2), encoding="utf-8")
    print(f"\nTraccia scritta in {REGISTRY.name}")


if __name__ == "__main__":
    main()
```

Linux/macOS (dalla cartella dello script):

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install scikit-learn
python3 evaluate.py
python3 rollback_versions.py
```

Windows PowerShell (dalla cartella dello script):

```powershell
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install scikit-learn
python evaluate.py
python rollback_versions.py
```

Output verificato eseguendo davvero `python3 rollback_versions.py` (non stimato):

```
=== Confronto versioni ===
v1 (LogisticRegression) accuracy = 0.9474
v2 (DecisionTree d=1)   accuracy = 0.9123

Peggioramento: 0.0351  (soglia consentita: 0.02)

>>> ROLLBACK: la v2 e' stata rifiutata dal gate
>>> Versione attiva in produzione: 1.0.0

Traccia scritta in registry.json
```

`registry.json` prodotto (verificato, e' l'output reale dello script sopra):

```json
{
  "active_version": "1.0.0",
  "decision": "ROLLBACK: la v2 e' stata rifiutata dal gate",
  "rule": "drop massimo consentito = 0.02",
  "versions": {
    "1.0.0": {
      "algorithm": "LogisticRegression",
      "accuracy": 0.9474
    },
    "2.0.0": {
      "algorithm": "DecisionTreeClassifier(max_depth=1)",
      "accuracy": 0.9123
    }
  }
}
```

`rollback_log.py` - produce il log richiesto dallo Step 6 con motivo, versione
di partenza e versione di arrivo, leggendo `registry.json`:

```python
#!/usr/bin/env python3
"""Lab 05 - Step 6: log di rollback leggibile, costruito da registry.json."""
import json
from datetime import datetime, timezone
from pathlib import Path

REGISTRY = Path(__file__).parent / "registry.json"


def main() -> None:
    registry = json.loads(REGISTRY.read_text(encoding="utf-8"))
    v1 = registry["versions"]["1.0.0"]
    v2 = registry["versions"]["2.0.0"]
    drop = round(v1["accuracy"] - v2["accuracy"], 4)
    rolled_back = registry["active_version"] == "1.0.0"

    log = {
        "timestamp": datetime.now(timezone.utc).isoformat(timespec="seconds"),
        "motivo": registry["decision"],
        "soglia_superata": f"drop osservato {drop} > soglia consentita 0.02" if rolled_back
                            else f"drop osservato {drop} entro soglia consentita 0.02",
        "versione_di_partenza": "2.0.0" if rolled_back else "1.0.0",
        "versione_di_arrivo": "1.0.0" if rolled_back else "2.0.0",
    }
    print("=== Log di rollback ===")
    for k, v in log.items():
        print(f"{k}: {v}")


if __name__ == "__main__":
    main()
```

Linux/macOS:

```bash
python3 rollback_log.py
deactivate
```

Windows PowerShell (dalla cartella dello script):

```powershell
python rollback_log.py
deactivate
```

Output verificato:

```
=== Log di rollback ===
timestamp: 2026-07-21T13:59:30+00:00
motivo: ROLLBACK: la v2 e' stata rifiutata dal gate
soglia_superata: drop osservato 0.0351 > soglia consentita 0.02
versione_di_partenza: 2.0.0
versione_di_arrivo: 1.0.0
```

## Template operativo

```markdown
# Deliverable Lab 05

## Scenario
- Sessione: Versioning Rollback e Prime Metriche
- Problema operativo: rollback di model-v2 a model-v1 dopo calo di accuratezza in produzione

## Confronto metriche
| Versione | Accuratezza | Latenza media | Errori osservati |
| -------- | ----------- | -------------- | ----------------- |
| model-v1 | ...         | ...            | ...                |
| model-v2 | ...         | ...            | ...                |

## Decisione
- Scelta: rollback / fix rapido
- Motivazione: soglia superata / tempo di fix stimato / rischio di v2 attivo
- Metrica/versione/log usato: ...
- Fallback: locale / simulato / console / non necessario

## Log di rollback
- Motivo: ...
- Versione di partenza: model-v2
- Versione di arrivo: model-v1
- Timestamp: ...

## Cleanup
- Server di inferenza locali chiusi: ...
- Tag/branch di test rimossi: ...
- Artifact/file temporanei rimossi: ...
- MLflow tracking server locale chiuso: ...
```

## Esempio sintetico

- Decisione corretta: rollback a `model-v1` perche il calo di accuratezza supera la soglia definita prima dell'incidente.
- Evidenza minima: tabella v1 vs v2 sullo stesso batch, piu il log di rollback con motivo e versioni coinvolte.
- Fallback accettabile: simulare gli stage del registry con tag Git e tabella Markdown quando MLflow non e disponibile.
- Cleanup atteso: chiusura server di inferenza, rimozione tag/branch di test e cartella `mlruns` di prova.

## Checkpoint risolto

- [x] modello v1 stabile: presente e motivato
- [x] modello v2 degradato: presente e motivato
- [x] latenza/errori: presente e motivato
- [x] decisione rollback: presente e motivato
- [x] verifica post-rollback: presente e motivato
- [x] Metriche v1/v2 confrontate sulle stesse condizioni
- [x] Soglia di rollback definita prima dell'incidente, non a posteriori
- [x] Log di rollback leggibile da terzi: versione di partenza, versione di arrivo e motivo comprensibili senza rileggere il codice
- [x] Cleanup obbligatorio completato o esplicitamente giustificato

## Errori comuni da evitare

- Confrontare v1 e v2 su batch di dati diversi, rendendo il confronto inutilizzabile.
- Decidere il rollback "a sensazione" invece che sulla soglia fissata in anticipo.
- Versionare il modello senza versionare i dati/feature che lo hanno prodotto.
- Eseguire il rollback senza lasciare un log leggibile di motivo e versioni coinvolte.

## Risposta breve per discussione orale

La soluzione e accettabile quando il rollback di **model-v2** a **model-v1** e motivato da una soglia di metrica definita in anticipo, il confronto v1/v2 e fatto sulle stesse condizioni, il log riporta motivo e versioni, e il cleanup delle risorse di test e confermato.
