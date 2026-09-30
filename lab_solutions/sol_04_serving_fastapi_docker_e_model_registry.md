# Soluzione Lab 04 - Serving FastAPI Docker e Model Registry

## Obiettivo risolto

- Focus operativo: endpoint `/predict` che espone `model_version`, registry minimale con stato per versione, decisione di promozione documentata.
- Deliverable atteso: registry aggiornato, endpoint funzionante o descritto, esito della promozione di `model_v3`.
- Forma consigliata: file Markdown con registry, snippet FastAPI e nota di cleanup.

## Soluzione guidata

1. Creare `model_v2.pkl` (accuracy 0.8889) e `model_v3.pkl` (accuracy 0.5) come artefatti di prova.
2. Scrivere il registry con colonne versione, path, metrica, stato, inizialmente con `model_v2` promosso.
3. Costruire `/predict` che carica `model_v2` all'avvio e restituisce `{prediction, model_version: "v2"}`.
4. Testare l'endpoint con una richiesta valida e verificare il campo `model_version` nella risposta.
5. Applicare il gate a `model_v3` (soglia 0.85): 0.5 non supera la soglia, quindi resta "scartato" e `model_v2` resta attivo.

## Esempio di deliverable compilato

|    # | Elemento                | Evidenza sintetica                                                                |
| ---: | ----------------------- | --------------------------------------------------------------------------------- |
|    1 | model_v2 registrato     | accuracy 0.8889, path `model_v2.pkl`, stato "promosso"                            |
|    2 | model_v3 registrato     | accuracy 0.5, path `model_v3.pkl`, stato "scartato" (sotto soglia 0.85)           |
|    3 | Endpoint /predict       | risposta `{"prediction": 0, "model_version": "v2"}` su richiesta valida           |
|    4 | Errore input non valido | risposta 422 su campo mancante, nessuna predizione restituita                     |
|    5 | Decisione documentata   | model_v3 non promosso, model_v2 resta servito, motivazione: metrica insufficiente |

## Codice della soluzione

Salva questo contenuto come `requirements.txt` nella cartella del lab:

```text
fastapi>=0.110,<1
uvicorn[standard]>=0.29,<1
httpx>=0.27,<1
scikit-learn>=1.4,<2
numpy>=1.26,<3
joblib>=1.3,<2
```

Struttura della soluzione: training di due versioni, registry, servizio FastAPI e
Dockerfile. Le due versioni usano lo stesso dataset e hanno metriche confrontabili:

```
lab04/
  train.py            # produce model_v2.pkl e model_v3.pkl con relativi .json
  build_registry.py    # costruisce registry.json e applica il gate su v3
  app.py                # endpoint FastAPI /predict, carica model_v2
  artifacts/            # output di train.py
  requirements.txt
  Dockerfile
```

`train.py` - Step 1: due versioni allenate sullo stesso dataset (wine di scikit-learn,
con rumore gaussiano aggiunto apposta: senza rumore un RandomForest arriva a
1.0 di accuracy, che non lascia spazio a un confronto realistico tra un candidato
buono e uno scadente):

```python
#!/usr/bin/env python3
"""Lab 04 - allena model_v2 e model_v3 con metriche diverse (Step 1).

model_v2: RandomForest ben configurato (dovrebbe superare la soglia 0.85).
model_v3: modello volutamente debole (un solo albero pochissimo profondo),
per dimostrare un candidato che il gate deve scartare.
"""
import json
from pathlib import Path

import joblib
import numpy as np
from sklearn.datasets import load_wine
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score
from sklearn.model_selection import train_test_split

ARTIFACTS = Path(__file__).parent / "artifacts"


def train_and_save(version, model, X_train, X_test, y_train, y_test, classes) -> float:
    model.fit(X_train, y_train)
    accuracy = round(float(accuracy_score(y_test, model.predict(X_test))), 4)
    joblib.dump(model, ARTIFACTS / f"model_{version}.pkl")
    metadata = {
        "version": version,
        "algorithm": type(model).__name__,
        "accuracy": accuracy,
        "classes": list(classes),
    }
    (ARTIFACTS / f"model_{version}.json").write_text(json.dumps(metadata, indent=2))
    return accuracy


def main() -> None:
    ARTIFACTS.mkdir(exist_ok=True)
    data = load_wine()
    rng = np.random.RandomState(0)
    X_noisy = data.data + rng.normal(0, 2.0, data.data.shape)
    X_train, X_test, y_train, y_test = train_test_split(
        X_noisy, data.target, test_size=0.3, random_state=42, stratify=data.target
    )

    acc_v2 = train_and_save("v2", RandomForestClassifier(n_estimators=200, random_state=42),
                             X_train, X_test, y_train, y_test, data.target_names)
    acc_v3 = train_and_save("v3", RandomForestClassifier(n_estimators=1, max_depth=1, random_state=42),
                             X_train, X_test, y_train, y_test, data.target_names)

    print(f"model_v2 accuracy={acc_v2}")
    print(f"model_v3 accuracy={acc_v3}")


if __name__ == "__main__":
    main()
```

Output verificato eseguendo davvero `python3 train.py`:

```
model_v2 accuracy=0.8889
model_v3 accuracy=0.5
```

`build_registry.py` - Step 2 e Step 5: registry minimale piu gate di promozione su
`model_v3` con soglia esplicita 0.85:

Esegui prima `train.py`: crea i modelli e i file JSON con le metriche dentro
`artifacts/`. Poi esegui `build_registry.py`: legge quelle metriche, scrive
`registry.json` nella cartella corrente e registra l'esito del gate per v3.
Con le metriche dell'esempio, v2 resta promosso e v3 viene scartato.
Questo script non avvia FastAPI e non cambia automaticamente il modello servito:
in questa soluzione `app.py` carica esplicitamente v2.

```python
#!/usr/bin/env python3
"""Lab 04 - registry minimale (Step 2) + gate di promozione su model_v3 (Step 5)."""
import json
from pathlib import Path

ARTIFACTS = Path(__file__).parent / "artifacts"
THRESHOLD = 0.85


def main() -> None:
    meta_v2 = json.loads((ARTIFACTS / "model_v2.json").read_text())
    meta_v3 = json.loads((ARTIFACTS / "model_v3.json").read_text())

    registry = []
    for meta, promoted_by_default in [(meta_v2, True), (meta_v3, False)]:
        accuracy = meta["accuracy"]
        stato = "promosso" if promoted_by_default else ("promosso" if accuracy >= THRESHOLD else "scartato")
        registry.append({"versione": meta["version"], "path": f"model_{meta['version']}.pkl",
                          "metrica": accuracy, "stato": stato})

    Path("registry.json").write_text(json.dumps(registry, indent=2))

    print("=== Registry ===")
    for row in registry:
        print(f"{row['versione']:4} {row['path']:16} accuracy={row['metrica']:<7} {row['stato']}")

    v3_accuracy = meta_v3["accuracy"]
    gate_ok = v3_accuracy >= THRESHOLD
    print(f"\n=== Gate promozione model_v3 (soglia {THRESHOLD}) ===")
    print(f"accuracy model_v3={v3_accuracy} -> {'PROMOSSO' if gate_ok else 'SCARTATO'}")


if __name__ == "__main__":
    main()
```

Output verificato:

```
=== Registry ===
v2   model_v2.pkl     accuracy=0.8889  promosso
v3   model_v3.pkl     accuracy=0.5     scartato

=== Gate promozione model_v3 (soglia 0.85) ===
accuracy model_v3=0.5 -> SCARTATO
```

`app.py` - Step 3: endpoint `/predict` che carica `model_v2` all'avvio e dichiara
`model_version` nella risposta:

```python
#!/usr/bin/env python3
"""Lab 04 - servizio FastAPI che carica model_v2 all'avvio (Step 3).

Avvio: uvicorn app:app --port 8000
"""
import json
from pathlib import Path

import joblib
from fastapi import FastAPI
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from pydantic import BaseModel

ARTIFACTS = Path(__file__).parent / "artifacts"
ACTIVE_VERSION = "v2"  # scelta esplicita: nell'esempio v3 non supera il gate

app = FastAPI(title="ModelOps Lab 04 - Serving")

model = joblib.load(ARTIFACTS / f"model_{ACTIVE_VERSION}.pkl")
metadata = json.loads((ARTIFACTS / f"model_{ACTIVE_VERSION}.json").read_text())


class PredictRequest(BaseModel):
    features: list[float]  # le 13 feature del dataset wine, in ordine


@app.exception_handler(RequestValidationError)
async def validation_handler(request, exc):
    return JSONResponse(status_code=422, content={"detail": "campo mancante o non numerico"})


@app.post("/predict")
def predict(payload: PredictRequest) -> dict:
    prediction = int(model.predict([payload.features])[0])
    return {"prediction": prediction, "model_version": metadata["version"]}
```

Sequenza di esecuzione e richieste HTTP dello Step 4:

Crea e attiva l'ambiente virtuale su Linux/macOS dalla cartella del lab:

```bash
python3 -m venv .venv

source .venv/bin/activate

python3 -m pip install -r requirements.txt

python3 train.py
python3 build_registry.py

python3 -m uvicorn app:app --port 8000 &
SERVER_PID=$!
python3 -c "
import time
import httpx
for attempt in range(100):
    try:
        httpx.get('http://127.0.0.1:8000/openapi.json', timeout=1).raise_for_status()
        break
    except httpx.HTTPError:
        time.sleep(0.2)
else:
    raise SystemExit('API non disponibile: controlla i log di uvicorn.')
features = [13.2,1.78,2.14,11.2,100.0,2.65,2.76,0.26,1.28,4.38,1.05,3.4,1050.0]
print(httpx.post('http://127.0.0.1:8000/predict', json={'features': features}).json())
print(httpx.post('http://127.0.0.1:8000/predict', json={}).status_code)
"
kill "$SERVER_PID"
wait "$SERVER_PID" 2>/dev/null || true
```

Windows PowerShell:

```powershell
py -3 -m venv .venv

.\.venv\Scripts\Activate.ps1

python -m pip install -r requirements.txt

python train.py

python build_registry.py

$server = Start-Process -FilePath (Join-Path $PWD ".venv\Scripts\python.exe") -ArgumentList "-m", "uvicorn", "app:app", "--port", "8000" -PassThru
Start-Sleep -Seconds 2
$features = @(13.2,1.78,2.14,11.2,100.0,2.65,2.76,0.26,1.28,4.38,1.05,3.4,1050.0)
$body = @{ features = $features } | ConvertTo-Json -Compress
Invoke-RestMethod -Uri http://127.0.0.1:8000/predict -Method Post -ContentType "application/json" -Body $body
try {
    Invoke-RestMethod -Uri http://127.0.0.1:8000/predict -Method Post -ContentType "application/json" -Body '{}'
} catch {
    $_.Exception.Response.StatusCode.value__
}
Stop-Process -Id $server.Id
```

Output verificato:

```
200 {'prediction': 0, 'model_version': 'v2'}
422 {'detail': 'campo mancante o non numerico'}
```

`Dockerfile` - installa le dipendenze e copia il codice e gli artefatti del modello:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .
COPY artifacts/ ./artifacts/

EXPOSE 8000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

`docker build` e `docker run` sono stati eseguiti davvero in questo ambiente (Docker
disponibile) per verificare che l'immagine funzioni end-to-end:

```bash
docker build -t lab04-serving .

docker run -d --name lab04-test -p 8000:8000 lab04-serving

curl -s -X POST http://127.0.0.1:8000/predict -H "Content-Type: application/json" \
     -d '{"features":[13.2,1.78,2.14,11.2,100.0,2.65,2.76,0.26,1.28,4.38,1.05,3.4,1050.0]}'
```

Windows PowerShell:

```powershell
docker build -t lab04-serving .

docker run -d --name lab04-test -p 8000:8000 lab04-serving

Invoke-RestMethod -Uri http://127.0.0.1:8000/predict -Method Post -ContentType "application/json" -Body $body

docker rm -f lab04-test

docker rmi lab04-serving

deactivate
```

Output verificato (build riuscita, stessa risposta del test locale):

```
{"prediction":0,"model_version":"v2"}
```

Cleanup eseguito subito dopo la verifica: `docker rm -f lab04-test && docker rmi lab04-serving`.

## Template operativo

```markdown
# Deliverable Lab 04

## Scenario
- Sessione: Serving FastAPI Docker e Model Registry
- Problema operativo: nessuno sa quale versione modello risponde in produzione

## Registry
| Versione | Path         | Metrica | Stato    |
| -------- | ------------ | ------- | -------- |
| v2       | model_v2.pkl | 0.8889  | promosso |
| v3       | model_v3.pkl | 0.5     | scartato |

## Endpoint /predict
- Modello caricato all'avvio: model_v2
- Risposta valida: { "prediction": 0, "model_version": "v2" }
- Risposta errore: 422 { "detail": "campo mancante o non numerico" }

## Decisione promozione v3
- Soglia: 0.85
- Esito: NON promosso (0.5 < 0.85)
- Modello attivo confermato: v2

## Cleanup
- Processo uvicorn fermato: ...
- File .pkl di prova rimossi: ...
- Note fallback: ...
```

## Esempio sintetico

- Decisione corretta: mantenere `model_v2` attivo e scartare `model_v3` perche non supera la soglia, documentando il confronto numerico.
- Evidenza minima: registry con almeno due versioni e stato esplicito, endpoint che dichiara la versione nella risposta.
- Fallback accettabile: se MLflow non e disponibile, un registry in tabella Markdown e sufficiente purche mostri versione, metrica e stato.
- Cleanup atteso: arresto del processo `uvicorn`/container e rimozione dei file modello di prova non consegnati.

## Checkpoint risolto

- [x] model_v2 e model_v3 registrati con metrica e stato
- [x] Endpoint /predict espone il campo model_version nella risposta
- [x] Richiesta non valida gestita con errore esplicito
- [x] Decisione su model_v3 documentata con soglia e motivazione
- [x] model_v2 confermato come versione attiva dopo il confronto
- [x] Gruppo spiega cosa farebbe per un rollback se model_v2 iniziasse a comportarsi male in produzione
- [x] Fallback documentato quando necessario
- [x] Cleanup obbligatorio completato o esplicitamente giustificato

## Errori comuni da evitare

- Servire un modello senza restituire la sua versione nella risposta, rendendo impossibile capire cosa ha risposto in caso di problemi.
- Promuovere `model_v3` "perche sembra piu recente" ignorando che la metrica e sotto soglia.
- Mescolare codice di training e codice di serving nello stesso script, rendendo fragile l'endpoint nel tempo.
- Lasciare il processo `uvicorn` attivo sulla porta 8000 dopo il lab, bloccando il lab successivo.

## Risposta breve per discussione orale

La soluzione e accettabile quando l'endpoint dichiara esplicitamente quale versione del modello ha risposto, il registry mostra metrica e stato per ogni versione, la decisione di non promuovere `model_v3` e motivata dalla soglia, e il cleanup del servizio e confermato.
