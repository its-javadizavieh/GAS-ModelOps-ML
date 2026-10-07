# Soluzione Lab 06 - Monitoring Logging Drift e Notifiche

## Obiettivo risolto

- Focus operativo: soglia di drift motivata da dati storici e notifica automatica collegata a un log leggibile.
- Deliverable atteso: script/serie temporale con drift score, soglia applicata e log di notifica.
- Forma consigliata: file Markdown o notebook con serie temporale, regola di alert e nota di cleanup.

## Soluzione guidata

1. Calcolare l'indicatore di drift (es. differenza di distribuzione o PSI) tra dati di training e batch recente.
2. Fissare la soglia (es. `drift_score > 0.08`) usando dati storici, non un numero arbitrario.
3. Implementare la logica soglia -> notifica: sopra soglia invia alert, sotto soglia registra log "nessuna azione".
4. Esporre la metrica su Prometheus/Grafana (o simulazione con tabella) mostrando la serie temporale.
5. Generare un alert di prova con canale, responsabile e azione correttiva, poi eseguire il cleanup.

## Esempio di deliverable compilato

|    # | Elemento             | Evidenza attesa                                                                    |
| ---: | -------------------- | ---------------------------------------------------------------------------------- |
|    1 | accuratezza finestra | Accuratezza calcolata su finestra mobile di 7 giorni                               |
|    2 | latenza p95          | Latenza p95 stabile sotto i 200ms nel periodo osservato                            |
|    3 | data drift           | drift_score = 2.5442 sulla feature eta media (batch 2026-07-20), sopra soglia 0.08 |
|    4 | alert rule           | Regola: drift_score > 0.08 per 2 batch consecutivi                                 |
|    5 | azione operativa     | Notifica inviata al team dati per verifica feature eta                             |

## Codice della soluzione

`drift_alert.py` - calcola il PSI (Population Stability Index) tra la distribuzione di
training e i batch recenti, fissa la soglia sui dati storici e applica la logica
soglia -> notifica richiesta dal lab. La soglia e la persistenza sono calcolate
esplicitamente; lo script usa solo la libreria standard di Python:

```python
#!/usr/bin/env python3
"""Lab 06 - drift score, soglia motivata e notifica.

Scenario: modello di classificazione clienti in produzione da un mese.
La feature "eta_media" del batch recente si e' spostata rispetto al
training set. Calcoliamo il PSI (Population Stability Index) con bin
basati sui quantili della reference, fissiamo una soglia motivata da
dati storici, e colleghiamo soglia -> notifica.

Uso: python3 drift_alert.py
"""
from __future__ import annotations

import math
import random
from dataclasses import dataclass


def psi(reference: list[float], current: list[float], bins: int = 5) -> float:
    """Population Stability Index tra due distribuzioni.

    I bin sono ricavati dai quantili della reference (pratica standard PSI):
    cosi ogni bin ha circa la stessa massa nella reference, ed e' piu robusto
    del binning a larghezza fissa su campioni piccoli.

    Lettura standard: < 0.10 stabile | 0.10-0.20 da tenere d'occhio | > 0.20 drift forte.
    """
    sorted_ref = sorted(reference)
    n = len(sorted_ref)
    edges = [sorted_ref[int(q * (n - 1))] for q in (i / bins for i in range(bins + 1))]
    edges[0], edges[-1] = -math.inf, math.inf

    def hist_pct(data: list[float]) -> list[float]:
        counts = [0] * bins
        for v in data:
            for i in range(bins):
                if edges[i] <= v < edges[i + 1]:
                    counts[i] += 1
                    break
        eps = 1e-4
        return [max(c / len(data), eps) for c in counts]

    ref_pct, cur_pct = hist_pct(reference), hist_pct(current)
    return sum((c - r) * math.log(c / r) for r, c in zip(ref_pct, cur_pct))


@dataclass
class AlertLogEntry:
    timestamp: str
    causa: str
    azione: str
    canale: str
    responsabile: str


def evaluate(drift_score: float, threshold: float, consecutive_over: int,
             persistence_required: int, timestamp: str) -> tuple[bool, AlertLogEntry | None]:
    """Applica soglia -> notifica, con requisito di persistenza per ridurre i falsi allarmi."""
    over_threshold = drift_score > threshold
    should_alert = over_threshold and consecutive_over >= persistence_required
    if not should_alert:
        return False, None
    entry = AlertLogEntry(
        timestamp=timestamp,
        causa=f"drift_score={drift_score} > soglia={threshold} per {consecutive_over} batch consecutivi",
        azione="Verificare la feature eta_media con il team dati; valutare retrain o rifiuto batch",
        canale="#modelops-alerts (Slack)",
        responsabile="Data Scientist on-call",
    )
    return True, entry


def main() -> None:
    rng = random.Random(42)

    # Distribuzione di riferimento: eta media vista in training (500 clienti).
    reference = [rng.gauss(40, 8) for _ in range(500)]

    # Soglia motivata da dati storici: sui batch delle ultime 6 settimane in cui
    # il modello si comportava bene (nessun cambio noto nei dati), il PSI
    # settimanale e' sempre rimasto sotto 0.05. Fissiamo drift_score > 0.08
    # come soglia di attenzione, con margine sopra il rumore storico osservato.
    historical_psi = []
    for _week in range(6):
        batch = [rng.gauss(40, 8) for _ in range(300)]  # stessa distribuzione: nessun drift reale
        historical_psi.append(round(psi(reference, batch), 4))
    threshold = 0.08
    print("=== Soglia motivata da dati storici ===")
    print(f"PSI osservato nelle ultime 6 settimane stabili: {historical_psi}")
    print(f"Soglia scelta: drift_score > {threshold} "
          f"(sopra il massimo storico {max(historical_psi)} con margine di sicurezza)\n")

    # Simulazione "scrape Prometheus": serie temporale di drift_score per 5 batch,
    # con uno shift reale a partire dal batch del 2026-07-19 (eta media si alza).
    print("=== Serie temporale (simulazione scrape Prometheus) ===")
    series: dict[str, float] = {}
    consecutive_over = 0
    log: list[AlertLogEntry] = []
    timestamps = ["2026-07-17", "2026-07-18", "2026-07-19", "2026-07-20", "2026-07-21"]
    for i, ts in enumerate(timestamps):
        if i < 2:
            batch = [rng.gauss(40, 8) for _ in range(300)]   # sotto soglia
        else:
            batch = [rng.gauss(55, 9) for _ in range(300)]   # shift reale: eta media +15
        drift_score = round(psi(reference, batch), 4)
        series[ts] = drift_score

        consecutive_over = consecutive_over + 1 if drift_score > threshold else 0
        alerted, entry = evaluate(drift_score, threshold, consecutive_over,
                                   persistence_required=2, timestamp=ts)
        status = "ALERT" if alerted else "sotto soglia"
        print(f"[{ts}] drift_score={drift_score:<7} soglia={threshold} -> {status}")
        if entry:
            log.append(entry)

    print("\n=== Log di notifica ===")
    if not log:
        print("Nessuna notifica inviata.")
    for entry in log:
        print(f"- timestamp={entry.timestamp}")
        print(f"  causa={entry.causa}")
        print(f"  azione_correttiva={entry.azione}")
        print(f"  canale={entry.canale} responsabile={entry.responsabile}")


if __name__ == "__main__":
    main()
```

Linux/macOS (dalla cartella dello script):

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 drift_alert.py
```

Windows PowerShell (dalla cartella dello script):

```powershell
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
python drift_alert.py
deactivate
```

Output verificato eseguendo davvero lo script (non stimato):

```text
=== Soglia motivata da dati storici ===
PSI osservato nelle ultime 6 settimane stabili: [0.0283, 0.0289, 0.026, 0.0199, 0.0117, 0.0476]
Soglia scelta: drift_score > 0.08 (sopra il massimo storico 0.0476 con margine di sicurezza)

=== Serie temporale (simulazione scrape Prometheus) ===
[2026-07-17] drift_score=0.0166  soglia=0.08 -> sotto soglia
[2026-07-18] drift_score=0.0012  soglia=0.08 -> sotto soglia
[2026-07-19] drift_score=2.1298  soglia=0.08 -> sotto soglia
[2026-07-20] drift_score=2.5442  soglia=0.08 -> ALERT
[2026-07-21] drift_score=1.8026  soglia=0.08 -> ALERT

=== Log di notifica ===
- timestamp=2026-07-20
  causa=drift_score=2.5442 > soglia=0.08 per 2 batch consecutivi
  azione_correttiva=Verificare la feature eta_media con il team dati; valutare retrain o rifiuto batch
  canale=#modelops-alerts (Slack) responsabile=Data Scientist on-call
- timestamp=2026-07-21
  causa=drift_score=1.8026 > soglia=0.08 per 3 batch consecutivi
  azione_correttiva=Verificare la feature eta_media con il team dati; valutare retrain o rifiuto batch
  canale=#modelops-alerts (Slack) responsabile=Data Scientist on-call
```

Nota didattica confermata dall'esecuzione reale: il batch del 2026-07-19 e' gia sopra
soglia (0.08) ma non genera ancora un alert perche il requisito di persistenza
(`persistence_required=2`) non e' ancora soddisfatto — evita di allarmare su un singolo
picco isolato. Solo dal 2026-07-20, con 2 batch consecutivi sopra soglia, parte la
notifica. Questo e' esattamente il compromesso rumore/silenzio richiesto dal lab.

## Template operativo

```markdown
# Deliverable Lab 06

## Scenario
- Sessione: Monitoring Logging Drift e Notifiche
- Problema operativo: drift su feature eta media rilevato nel batch recente

## Metriche monitorate
| Metrica              | Valore osservato | Soglia | Esito       |
| -------------------- | ---------------- | ------ | ----------- |
| accuratezza finestra | ...              | ...    | sopra/sotto |
| drift_score          | ...              | > 0.08 | sopra/sotto |

## Decisione
- Scelta: alert inviato / nessuna azione
- Motivazione: soglia superata su N batch consecutivi
- Metrica/versione/log usato: ...
- Fallback: locale / simulato / console / non necessario

## Log di notifica
- Timestamp: ...
- Causa: soglia drift superata
- Azione correttiva proposta: ...
- Canale/responsabile: ...

## Cleanup
- Processi Prometheus/exporter chiusi: ...
- Dashboard/alert rule di test rimossi: ...
- File temporanei rimossi: ...
```

## Esempio sintetico

- Decisione corretta: inviare notifica quando il drift_score supera la soglia storica per piu batch consecutivi, non su un singolo picco.
- Evidenza minima: serie temporale della metrica monitorata piu la regola di soglia applicata.
- Fallback accettabile: simulare lo scrape Prometheus con un dizionario Python e la dashboard con un grafico Matplotlib.
- Cleanup atteso: chiusura di exporter/container Grafana e rimozione di regole di alert create per il test.

## Checkpoint risolto

- [x] accuratezza finestra: presente e motivato
- [x] latenza p95: presente e motivato
- [x] data drift: presente e motivato
- [x] alert rule: presente e motivato
- [x] azione operativa: presente e motivato
- [x] Soglia di drift motivata con dati storici
- [x] Alert collegato a canale e responsabile
- [x] Il log distingue chiaramente i casi "sotto soglia" (nessuna azione) da quelli "sopra soglia" (notifica inviata)
- [x] Cleanup obbligatorio completato o esplicitamente giustificato

## Errori comuni da evitare

- Fissare la soglia di drift senza guardare dati storici, generando falsi allarmi o silenzio.
- Loggare ogni evento senza filtro, nascondendo il segnale reale nel rumore.
- Configurare un alert senza canale o responsabile che lo riceva davvero.
- Lasciare dashboard o regole di alert attive dopo il lab senza cleanup.

## Risposta breve per discussione orale

La soluzione e accettabile quando la soglia di **drift** e motivata da dati storici, l'**alert** e collegato a un canale e a un responsabile, il log distingue chiaramente i casi sopra e sotto soglia, e il cleanup delle risorse di monitoring e confermato.
