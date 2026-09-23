# Soluzione Lab 03 - API Docker e Strategie di Deploy

## Obiettivo risolto

- Focus operativo: scelta motivata cloud/on-prem/SaaS per dati sensibili + contratto API + Dockerfile minimale coerente.
- Deliverable atteso: tabella comparativa, decisione motivata, contratto `/predict`, Dockerfile.
- Forma consigliata: file Markdown con tabella, contratto API in blocco codice e Dockerfile.

## Soluzione guidata

1. Costruire la tabella comparativa con controllo dati, costo, competenze richieste, tempo di attivazione per cloud/on-prem/SaaS.
2. Applicare la tabella allo scenario (dati sanitari, budget limitato, **nessun team infra interno**) e scegliere una sola opzione, motivando in 2-3 frasi. Il vincolo "nessun team infra" pesa quanto quello sui dati sensibili: non va ignorato a favore del solo controllo dati.
3. Scrivere il contratto `/predict`: payload di input tipizzato, risposta di successo, risposta di errore su input non valido.
4. Scrivere un Dockerfile minimale con base image, dipendenze, comando di avvio coerente con la scelta.
5. Verificare la coerenza tra Dockerfile e contratto API (porta esposta, comando di avvio corrispondente al servizio descritto).

## Esempio di deliverable compilato

|    # | Elemento               | Evidenza sintetica                                                                                                                                                                                                                                   |
| ---: | ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|    1 | Tabella comparativa    | cloud/on-prem/SaaS confrontati su controllo dati, costo, competenze, tempo                                                                                                                                                                           |
|    2 | Decisione motivata     | SaaS gestito scelto: dati sensibili richiedono controllo, ma senza team infra interno l'on-prem introduce un rischio operativo piu grave del rischio di controllo (verificare che il fornitore SaaS offra garanzie contrattuali/compliance adeguate) |
|    3 | Contratto `/predict`   | payload `{feature_1: float, feature_2: float}`, risposta `{class, score}`, errore 422                                                                                                                                                                |
|    4 | Dockerfile minimale    | base `python:3.11-slim`, `pip install -r requirements.txt`, `CMD uvicorn app:app`                                                                                                                                                                    |
|    5 | Coerenza porta/comando | porta 8000 esposta nel Dockerfile, stessa porta usata nel contratto API                                                                                                                                                                              |

## Codice della soluzione

Punteggio della decisione (Step 1-2) verificato eseguendo davvero
`demo/demo_03_api_docker/deploy_decision.py` sullo scenario `sanita` (dati sanitari
sensibili, budget limitato, nessun team infra interno) — script riusato cosi com'e,
nessun adattamento necessario perche calcola esattamente il confronto pesato
richiesto dagli step 1-2 del lab:

```bash
python3 deploy_decision.py --scenario sanita
```

Output verificato (non stimato):

```
=== Scenario: sanita ===
Dati sanitari sensibili, budget limitato, nessun team infra interno

  SaaS       punteggio pesato = 3.57 <-- scelta
  cloud      punteggio pesato = 3.21
  on-prem    punteggio pesato = 2.43

>>> Decisione: SaaS
>>> Motivazione: punteggio piu alto sui criteri pesati per QUESTO scenario.
>>> Nota: nessuna opzione vince sempre, cambia lo scenario e cambia la scelta.
```

Il punteggio pesato conferma la decisione della tabella comparativa: SaaS (3.57)
batte cloud (3.21) e on-prem (2.43) su questo scenario specifico.

`app.py` - contratto `/predict` richiesto dallo Step 3, adattato dal pattern FastAPI
del demo di Sessione 04 (payload tipizzato + gestione esplicita dell'errore 422):

```python
#!/usr/bin/env python3
"""Lab 03 - contratto API /predict per il servizio scelto (SaaS gestito).

Avvio locale: uvicorn app:app --port 8000
"""
from fastapi import FastAPI
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from pydantic import BaseModel

app = FastAPI(title="Scoring service - Lab 03")


class PredictRequest(BaseModel):
    feature_1: float
    feature_2: float


@app.exception_handler(RequestValidationError)
async def validation_handler(request, exc):
    # Risposta di errore esplicita per input non valido (Step 3 del lab).
    return JSONResponse(
        status_code=422,
        content={"detail": "campo mancante o non numerico"},
    )


@app.post("/predict")
def predict(payload: PredictRequest) -> dict:
    # Modello giocattolo: soglia sulla somma delle due feature.
    score = round((payload.feature_1 + payload.feature_2) / 2, 4)
    predicted_class = "positive" if score >= 5 else "negative"
    return {"class": predicted_class, "score": score}
```

Test locale eseguito davvero con `uvicorn` + client HTTP (Step 3, prima ancora di
passare a Docker):

```bash
uvicorn app:app --port 8000 &
python3 -c "
import httpx
r = httpx.post('http://127.0.0.1:8000/predict', json={'feature_1': 6.0, 'feature_2': 7.0})
print(r.status_code, r.json())
r = httpx.post('http://127.0.0.1:8000/predict', json={'feature_1': 6.0})
print(r.status_code, r.json())
"
```

Output verificato:

```
200 {'class': 'positive', 'score': 6.5}
422 {'detail': 'campo mancante o non numerico'}
```

Kill server:

```bash
pkill -f "uvicorn app:app"
```


`Dockerfile` - minimale, coerente con la scelta SaaS (porta 8000, stesso comando di
avvio testato sopra):

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

`requirements.txt`:

```
fastapi>=0.110
uvicorn[standard]>=0.29
```

Verifica richiesta dallo Step 5: `docker build` e' stato eseguito davvero in questo
ambiente (Docker disponibile), non solo simulato. Build e run confermano che il
Dockerfile e' sintatticamente corretto e che il contratto risponde identico dentro
al container:

```bash
docker build -t lab03-scoring-service .

docker run -d --name lab03-test -p 8000:8000 lab03-scoring-service

docker ps

curl -s -X POST http://127.0.0.1:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"feature_1": 6.0, "feature_2": 7.0}'

curl -s -o /dev/null -w "%{http_code}\n" \
  -X POST http://127.0.0.1:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"feature_1": 6.0}'
```

Output verificato (build riuscita, container avviato, stesse risposte del test locale):

```
{"class":"positive","score":6.5}
422
```

Cleanup eseguito subito dopo la verifica: `docker rm -f lab03-test && docker rmi lab03-scoring-service`.

## Template operativo

```markdown
# Deliverable Lab 03

## Scenario
- Sessione: API Docker e Strategie di Deploy
- Problema operativo: scelta deploy per dati sanitari sensibili con budget limitato

## Comparazione
| Opzione | Controllo dati | Costo iniziale | Competenze richieste | Tempo attivazione |
| ------- | -------------- | -------------- | -------------------- | ----------------- |
| Cloud   | medio          | basso          | medie                | rapido            |
| On-prem | alto           | alto           | alte                 | lento             |
| SaaS    | basso          | basso          | basse                | immediato         |

## Decisione
- Scelta: SaaS gestito (con verifica di garanzie contrattuali/compliance sui dati sanitari)
- Motivazione: il controllo diretto sui dati conterebbe di piu se il team avesse le competenze per gestirlo; senza un team infra interno, un on-prem mal gestito e' un rischio piu concreto del rischio di affidarsi a un fornitore SaaS con garanzie contrattuali adeguate. Il budget limitato rende inoltre insostenibili i costi iniziali dell'on-prem.

## Contratto API /predict
- Input: { "feature_1": float, "feature_2": float }
- Output valido: { "class": string, "score": float }
- Output errore: 422 { "detail": "campo mancante o non numerico" }

## Dockerfile
- Base image: ...
- Dipendenze: ...
- Comando avvio: ...

## Cleanup
- Immagini/container rimossi: ...
- Note fallback: ...
```

## Esempio sintetico

- Decisione corretta: scegliere l'opzione che bilancia TUTTI i vincoli dello scenario, non solo quello piu vistoso. Qui i vincoli sono tre (dati sensibili, budget limitato, nessun team infra) e nessuno da solo decide: e' la combinazione a spostare la scelta verso SaaS invece che on-prem. Uno script di supporto alla decisione con criteri pesati (`demo/demo_03_api_docker/deploy_decision.py`) arriva alla stessa conclusione per questo identico scenario.
- Evidenza minima: tabella comparativa applicata al caso specifico, non generica, piu contratto API con caso di errore esplicito.
- Fallback accettabile: se Docker non e installato, validare il Dockerfile per confronto con un esempio ufficiale, senza eseguire la build.
- Cleanup atteso: rimozione di immagini e container di prova costruiti durante la verifica del Dockerfile.

## Checkpoint risolto

- [x] Tabella comparativa con almeno 4 criteri per le tre opzioni
- [x] Decisione motivata rispetto ai vincoli specifici dello scenario
- [x] Contratto API con payload, risposta valida e risposta di errore
- [x] Dockerfile minimale con base image, dipendenze, comando di avvio
- [x] Coerenza verificata tra porta/comando del Dockerfile e contratto API
- [x] Fallback documentato quando necessario
- [x] Cleanup obbligatorio completato o esplicitamente giustificato

## Errori comuni da evitare

- Scegliere cloud/on-prem/SaaS senza collegare la scelta ai vincoli specifici dello scenario (dati sensibili, budget, competenze).
- Farsi guidare solo dal vincolo piu "allarmante" (dati sanitari sensibili -> on-prem "di default") ignorando gli altri vincoli dello scenario: qui la mancanza di un team infra interno rende l'on-prem rischioso quanto (o piu) di un SaaS con garanzie contrattuali.
- Definire un contratto API senza il caso di errore, lasciando ambiguo cosa succede con input non valido.
- Scrivere un Dockerfile con comando di avvio che non corrisponde alla porta o al file descritti nel contratto API.
- Lasciare immagini Docker di test accumulate senza rimuoverle (`docker images` che cresce senza controllo).

## Risposta breve per discussione orale

La soluzione e accettabile quando la scelta tra cloud, on-prem e SaaS e motivata sui vincoli reali dello scenario, il contratto API copre sia il caso valido sia l'errore, il Dockerfile e coerente con quel contratto, e il cleanup delle risorse Docker di prova e confermato.
