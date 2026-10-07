# Soluzione Lab 07 - Laboratorio Monitoring e Explainability SHAP LIME

## Obiettivo risolto

- Focus operativo: collegare un segnale di monitoring degradato a una spiegazione SHAP/LIME della causa.
- Deliverable atteso: caso spiegato, feature influente individuata, nota di impatto business.
- Forma consigliata: file Markdown o notebook con grafico SHAP/LIME, tabella di collegamento e nota di cleanup.

## Soluzione guidata

1. Recuperare il segnale di monitoring: metrica degradata, timestamp e batch coinvolto (dal lab di monitoring).
2. Selezionare 2-3 predizioni sbagliate o sospette dal batch segnalato.
3. Generare la spiegazione SHAP (waterfall) per almeno un caso, mostrando il contributo delle feature.
4. Generare la spiegazione LIME sullo stesso caso o su un secondo caso e confrontare i risultati.
5. Collegare la feature piu influente al possibile drift gia rilevato dal monitoring.
6. Scrivere la nota di impatto business e chiudere con il cleanup.

## Esempio di deliverable compilato

|    # | Elemento           | Evidenza attesa                                                                 |
| ---: | ------------------ | ------------------------------------------------------------------------------- |
|    1 | metrica degradata  | Accuratezza 0.958 sul batch `batch-2026w29`, sotto la soglia 0.97 -> alert      |
|    2 | feature influente  | SHAP indica "worst area" come feature con contributo maggiore (+0.1435)         |
|    3 | spiegazione locale | SHAP top-5 per il caso #92 (predetto malignant, reale benign, confidenza 0.755) |
|    4 | impatto business   | Da verificare se "worst area" mostra drift nel monitoring del lab 06            |
|    5 | nota limite        | LIME su stesso caso indica "mean area" al primo posto: non coincide con SHAP    |

## Codice della soluzione

`explain_case.py` - collega il segnale di monitoring (accuratezza sotto soglia) alla
spiegazione SHAP/LIME dei due casi piu sospetti. Calcola la soglia di monitoring
e segue gli step 1-5 del lab:

```python
#!/usr/bin/env python3
"""Lab 07 - dal segnale di monitoring alla spiegazione SHAP/LIME.

Scenario: il monitoring (lab 06) segnala che l'accuratezza del modello di
scoring crediti e' scesa sotto soglia sul batch recente. Prima di un
rollback, isoliamo un caso sbagliato del batch e lo spieghiamo con SHAP
(contributo per feature) e LIME (approssimazione locale interpretabile),
poi colleghiamo la feature piu influente al possibile drift.

Nota didattica: usiamo il dataset scikit-learn "breast cancer" (pubblico,
gia' incluso in sklearn, nessun dato reale di credito) come stand-in per il
batch di scoring crediti del lab: le due classi ("malignant"/"benign")
giocano il ruolo di "credito rifiutato"/"credito approvato". Le feature
tecniche del dataset restano quelle originali (misure di un tumore), non
sono feature di credito reali: il punto del lab e' il METODO (monitoring
-> caso sospetto -> SHAP/LIME -> nota business), non il dominio applicativo.

Uso:  python3 explain_case.py
"""
from __future__ import annotations

import numpy as np
import shap
from lime.lime_tabular import LimeTabularExplainer
from sklearn.datasets import load_breast_cancer
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score
from sklearn.model_selection import train_test_split

# Soglia di accuratezza sotto la quale il monitoring del lab 06 genera un alert.
ACCURACY_THRESHOLD = 0.97


def main() -> None:
    data = load_breast_cancer()
    X_train, X_test, y_train, y_test = train_test_split(
        data.data, data.target, test_size=0.25, random_state=42, stratify=data.target
    )

    model = RandomForestClassifier(n_estimators=200, random_state=42)
    model.fit(X_train, y_train)

    proba = model.predict_proba(X_test)
    predictions = model.predict(X_test)
    batch_accuracy = round(float(accuracy_score(y_test, predictions)), 4)

    # --- Step 1: segnale di monitoring (collegato al lab 06) ---
    print("=== Segnale di monitoring (batch settimana corrente) ===")
    print(f"Accuratezza batch: {batch_accuracy}  |  soglia: {ACCURACY_THRESHOLD}  |  "
          f"esito: {'SOTTO SOGLIA - alert' if batch_accuracy < ACCURACY_THRESHOLD else 'ok'}")
    print(f"Timestamp: 2026-07-21  |  batch_id: batch-2026w29  |  n_predizioni: {len(y_test)}\n")

    # --- Step 2: selezione dei casi sbagliati o sospetti dal batch segnalato ---
    wrong_mask = predictions != y_test
    wrong_idx = np.where(wrong_mask)[0]
    confidences = np.max(proba[wrong_idx], axis=1)
    # I 2 casi sbagliati con confidenza piu alta: quelli piu "preoccupanti"
    # (il modello ha sbagliato ma era anche molto sicuro di se').
    ranked = wrong_idx[np.argsort(-confidences)]
    selected_cases = ranked[:2] if len(ranked) >= 2 else ranked

    print("=== Step 2: casi sospetti selezionati dal batch ===")
    for idx in selected_cases:
        print(f"  caso #{idx}: predetto={data.target_names[predictions[idx]]:<10} "
              f"reale={data.target_names[y_test[idx]]:<10} "
              f"confidenza={round(float(np.max(proba[idx])), 4)}")
    print()

    case_idx = int(selected_cases[0])
    predicted_label = data.target_names[predictions[case_idx]]
    true_label = data.target_names[y_test[case_idx]]
    class_idx = int(predictions[case_idx])

    # --- Step 3: spiegazione SHAP per il primo caso ---
    print(f"=== Step 3: spiegazione SHAP per il caso #{case_idx} "
          f"(predetto={predicted_label}, reale={true_label}) ===")
    explainer = shap.TreeExplainer(model)
    shap_values = explainer.shap_values(X_test[case_idx: case_idx + 1])
    # La forma di shap_values varia tra versioni della libreria:
    #  - lista di array (uno per classe), ciascuno (1, n_feature)
    #  - array unico (1, n_feature, n_classi)
    # Normalizziamo sempre a un vettore di lunghezza n_feature per la classe predetta.
    if isinstance(shap_values, list):
        contribs = np.array(shap_values[class_idx])[0]
    else:
        values = np.array(shap_values)
        if values.ndim == 3 and values.shape[-1] > 1:
            contribs = values[0, :, class_idx]
        else:
            contribs = values[0]

    top5_shap = np.argsort(-np.abs(contribs))[:5]
    for rank, feat_i in enumerate(top5_shap, start=1):
        sign = "+" if contribs[feat_i] >= 0 else "-"
        print(f"  {rank}. {data.feature_names[feat_i]:<24} "
              f"contributo {sign}{abs(contribs[feat_i]):.4f}  "
              f"(valore osservato: {X_test[case_idx, feat_i]:.2f})")

    # --- Step 4: spiegazione LIME sullo stesso caso, per confronto ---
    print(f"\n=== Step 4: spiegazione LIME per lo stesso caso #{case_idx} ===")
    lime_explainer = LimeTabularExplainer(
        X_train,
        feature_names=list(data.feature_names),
        class_names=list(data.target_names),
        mode="classification",
        random_state=42,
    )
    lime_exp = lime_explainer.explain_instance(
        X_test[case_idx], model.predict_proba, num_features=5
    )
    lime_top_feature_name = None
    for rank, (feature_rule, weight) in enumerate(lime_exp.as_list(), start=1):
        if rank == 1:
            lime_top_feature_name = feature_rule
        sign = "+" if weight >= 0 else "-"
        print(f"  {rank}. {feature_rule:<40} peso {sign}{abs(weight):.4f}")

    # --- Step 5: collegamento con il monitoring / possibile drift ---
    shap_top_feature = data.feature_names[top5_shap[0]]
    print("\n=== Step 5: collegamento con il segnale di monitoring ===")
    print(f"Feature con contributo maggiore secondo SHAP: {shap_top_feature}")
    print(f"Feature in cima al ranking LIME: {lime_top_feature_name}")
    same_top_feature = shap_top_feature in (lime_top_feature_name or "")
    print(f"SHAP e LIME concordano sulla feature principale: {same_top_feature}")
    print(f"Azione suggerita: verificare se '{shap_top_feature}' mostra drift nel "
          f"monitoring del lab 06 per il batch batch-2026w29; se si', e' la causa "
          f"piu probabile del calo di accuratezza sotto soglia.")


if __name__ == "__main__":
    main()
```

Linux/macOS (dalla cartella dello script):

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install numpy scikit-learn shap lime
python3 explain_case.py
```

Windows PowerShell (dalla cartella dello script):

```powershell
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install numpy scikit-learn shap lime
python explain_case.py
deactivate
```

Output verificato eseguendo davvero lo script (non stimato; shap 0.52.0, lime tabular,
scikit-learn 1.9.0 su Python 3.12 - i pesi possono variare leggermente con altre
versioni delle librerie, ma il ranking delle feature top e la discrepanza SHAP/LIME
qui sotto sono quelli osservati in questa esecuzione):

```text
=== Segnale di monitoring (batch settimana corrente) ===
Accuratezza batch: 0.958  |  soglia: 0.97  |  esito: SOTTO SOGLIA - alert
Timestamp: 2026-07-21  |  batch_id: batch-2026w29  |  n_predizioni: 143

=== Step 2: casi sospetti selezionati dal batch ===
  caso #92: predetto=malignant  reale=benign     confidenza=0.755
  caso #4: predetto=benign     reale=malignant  confidenza=0.7

=== Step 3: spiegazione SHAP per il caso #92 (predetto=malignant, reale=benign) ===
  1. worst area               contributo +0.1435  (valore osservato: 1009.00)
  2. worst perimeter          contributo +0.1385  (valore osservato: 117.20)
  3. worst radius             contributo +0.0759  (valore osservato: 18.13)
  4. mean area                contributo +0.0712  (valore osservato: 838.10)
  5. mean perimeter           contributo +0.0495  (valore osservato: 106.60)

=== Step 4: spiegazione LIME per lo stesso caso #92 ===
  1. mean area > 765.38                       peso -0.0554
  2. 0.06 < worst concave points <= 0.10      peso +0.0529
  3. mean perimeter > 103.67                  peso -0.0410
  4. 97.62 < worst perimeter <= 124.97        peso -0.0298
  5. mean radius > 15.75                      peso -0.0240

=== Step 5: collegamento con il segnale di monitoring ===
Feature con contributo maggiore secondo SHAP: worst area
Feature in cima al ranking LIME: mean area > 765.38
SHAP e LIME concordano sulla feature principale: False
Azione suggerita: verificare se 'worst area' mostra drift nel monitoring del lab 06
per il batch batch-2026w29; se si', e' la causa piu probabile del calo di accuratezza
sotto soglia.
```

Nota didattica confermata dall'esecuzione reale: SHAP e LIME NON concordano sulla feature
piu influente per questo caso (`worst area` per SHAP, `mean area` per LIME) — esattamente
la discrepanza che il lab chiede di documentare invece di ignorare (vedi "Troubleshooting
rapido" nel lab e riga "nota limite" nella tabella sopra), non un errore dello script.

## Template operativo

```markdown
# Deliverable Lab 07

## Scenario
- Sessione: Laboratorio Monitoring e Explainability SHAP LIME
- Problema operativo: accuratezza degradata sul batch corrente, causa da individuare con SHAP/LIME

## Collegamento monitoring -> explainability
| Segnale monitoring       | Caso selezionato | Feature influente (SHAP) | Feature influente (LIME) |
| ------------------------ | ---------------- | ------------------------ | ------------------------ |
| accuratezza sotto soglia | predizione #...  | ...                      | ...                      |

## Decisione
- Scelta: SHAP / LIME / entrambi
- Motivazione: coerenza matematica vs velocita/leggibilita locale
- Metrica/versione/log usato: ...
- Fallback: locale / simulato / console / non necessario

## Impatto business
- Destinatario: ...
- Azione proposta: ...

## Cleanup
- Kernel/processi SHAP-LIME chiusi: ...
- File .html/grafici temporanei rimossi: ...
- Dashboard di supporto chiusa o non creata: ...
```

## Esempio sintetico

- Decisione corretta: usare SHAP per il ranking coerente delle feature e LIME come verifica rapida locale sullo stesso caso.
- Evidenza minima: grafico SHAP/LIME per almeno un caso, piu la tabella che collega segnale di monitoring e feature influente.
- Fallback accettabile: dataset ridotto e dashboard simulata con tabella Markdown se mancano risorse cloud.
- Cleanup atteso: chiusura kernel e rimozione di grafici, cache degli explainer e regole di alert di test.

## Checkpoint risolto

- [x] metrica degradata: presente e motivato
- [x] feature influente: presente e motivato
- [x] spiegazione locale: presente e motivato
- [x] impatto business: presente e motivato
- [x] nota limite: presente e motivato
- [x] Spiegazione collegata esplicitamente al segnale di monitoring
- [x] Caso e input della predizione spiegata chiaramente identificati
- [x] La nota di impatto business e comprensibile anche a chi non conosce SHAP/LIME
- [x] Cleanup obbligatorio completato o esplicitamente giustificato

## Errori comuni da evitare

- Generare un grafico SHAP/LIME senza collegarlo al segnale di monitoring che lo ha reso necessario.
- Usare SHAP su tutto il dataset invece che su un campione mirato, rendendo il calcolo troppo lento per il timebox.
- Ignorare una discrepanza tra SHAP e LIME invece di documentarla come limite.
- Presentare il grafico senza tradurlo in una nota comprensibile per un interlocutore business.

## Risposta breve per discussione orale

La soluzione e accettabile quando la spiegazione **SHAP/LIME** e collegata in modo esplicito al segnale di **monitoring** che l'ha generata, il caso spiegato e identificabile con il suo input, e la nota di impatto business e comprensibile anche senza conoscere SHAP/LIME.
