# Lab 11 - Configurare, eseguire e osservare un agente RL

## Obiettivo e lavoro richiesto

Esegui il codice fornito per addestrare un agente Q-learning in FrozenLake, confrontarlo con una politica casuale e osservare il comportamento del modello ricaricato. Poi esegui il confronto tra reward e preferenze sul CSV assegnato.

**Il codice completo è incluso nella sezione «Codice fornito».** Il tuo lavoro consiste nel preparare l'ambiente, salvare gli script indicati, configurare le esecuzioni, lanciarle e documentare passaggi e risultati osservati.

## S1 - Prepara una cartella autonoma e l'ambiente

Crea una cartella personale chiamata `lavoro_lab11` e raccogli questi file:

| File                                 | Come ottenerlo                                                             |
| ------------------------------------ | -------------------------------------------------------------------------- |
| Questo documento                     | Copia del materiale distribuito dal docente                                |
| `11_dataset_preferenze_simulate.csv` | CSV fornito dal docente, da conservare invariato                           |
| `rl_frozenlake.py`                   | Salva una sola volta il primo blocco Python della sezione «Codice fornito» |
| `sol11_reward_ranking.py`            | Salva una sola volta il secondo blocco Python della stessa sezione         |

Tutti i comandi seguenti vanno eseguiti **dalla cartella lavoro_lab11**, dove si trovano script e CSV. Questa disposizione funziona anche scaricando i soli materiali del laboratorio.

Usa Python 3.12. Crea l'ambiente virtuale:

```bash
python -m venv .venv
```

Attivalo con `source .venv/bin/activate` su Linux/macOS oppure `.\.venv\Scripts\Activate.ps1` in PowerShell. Scegli il comando adatto alla tua postazione, poi installa le dipendenze e registra le versioni:

```bash
python -m pip install "gymnasium>=1.0,<2" numpy matplotlib
python -m pip freeze > ambiente.txt
```

Non servono GPU, credenziali o modelli da scaricare. FrozenLake è incluso in Gymnasium. Se la postazione è senza rete, usa un ambiente con i pacchetti già predisposti dal docente.

**Documenta:** sistema operativo, versione Python, comando di attivazione e versione dei pacchetti. Un errore di installazione va risolto prima del training.

## S2 - Controlla la configurazione e fai una previsione

Leggi la funzione `make_env` e la configurazione del training nel codice fornito:

| Impostazione           | Valore iniziale                      | Come configurarla                                             |
| ---------------------- | ------------------------------------ | ------------------------------------------------------------- |
| Ambiente               | FrozenLake-v1, mappa 4×4             | Mantieni la mappa fornita per il confronto di base            |
| Stati / azioni         | 16 / 4                               | Dipendono dall'ambiente                                       |
| Episodi di training    | 6000 per seed                        | Argomento `--episodes`                                        |
| Seed di training       | 42, 43, 44                           | Argomento `--seeds`                                           |
| Cartella dei risultati | rl_output                            | Argomento `--out`; usa un nome nuovo per ripetere il training |
| Scivolamento           | Disattivato                          | Aggiungi `--slippery` per l'esperimento aggiuntivo            |
| Alpha / gamma          | 0,1 / 0,99                           | Valori forniti in `train_one`                                 |
| Esplorazione           | Epsilon da 1 a 0,05                  | Riduzione già implementata nel codice                         |
| Valutazione            | 200 episodi, seed a partire da 10000 | Stesso protocollo per baseline e agente                       |

La mappa ha righe SFFF, FHFH, FFFH, HFFG: S è la partenza, F il ghiaccio, H una buca e G l'obiettivo. Il reward vale 1 al goal e 0 negli altri casi; il limite è 100 passi.

Prima di eseguire, annota quale politica ti aspetti raggiunga il goal più spesso e perché. Individua nel codice la distinzione tra `terminated` e `truncated` e il punto in cui viene aggiornata Q.

**Documenta:** configurazione scelta e previsione iniziale. Questa lettura serve a interpretare il programma che eseguirai.

## S3 - Avvia il training e osserva gli artefatti

Esegui:

```bash
python rl_frozenlake.py train --out rl_output --episodes 6000 --seeds 42 43 44
```

Il programma esegue i tre training e, per ciascuno, valuta sia azioni casuali sia la politica Q-learning. Alla fine trovi:

| Percorso                               | Contenuto da osservare                                  |
| -------------------------------------- | ------------------------------------------------------- |
| `rl_output/comparison.json`            | Confronto random/Q-learning dei tre seed                |
| `rl_output/seed_42/training.csv`       | Ritorno, durata, epsilon e successo durante il training |
| `rl_output/seed_42/evaluation.csv`     | Episodi di valutazione delle due politiche              |
| `rl_output/seed_42/q_table.npy`        | Tabella Q appresa e salvata                             |
| `rl_output/seed_42/config.json`        | Configurazione, versioni e sintesi della valutazione    |
| `rl_output/seed_42/learning_curve.png` | Success rate di training su finestra mobile             |

Le cartelle seed_43 e seed_44 hanno la stessa struttura. Apri i grafici e controlla nei metadati se i valori Q sono cambiati. Se una cartella output esiste già, il programma la rifiuta per conservare la prova precedente: configura un nuovo nome per la ripetizione.

**Documenta:** comando usato, percorso dei risultati, success rate delle due politiche per tutti i seed e andamento della curva. Non sono richiesti numeri prestabiliti; registra anche un eventuale mancato miglioramento.

## S4 - Ricarica il modello e osserva un episodio

Esegui in processi nuovi:

```bash
python rl_frozenlake.py evaluate --run rl_output/seed_42
python rl_frozenlake.py demo --run rl_output/seed_42
```

Il primo comando carica Q da disco, valuta senza aggiornarla e salva `evaluation_reloaded.csv` e `evaluation_reloaded.json`. Il secondo mostra il percorso testuale e salva `episode.txt` nella cartella del seed. Puoi osservare gli altri seed sostituendo il percorso passato a `--run`.

Confronta il risultato ricaricato con quello registrato dopo il training. Spiega la differenza tra curva di training con esplorazione e valutazione greedy. Una valutazione sulla stessa mappa non dimostra generalizzazione a mappe nuove.

**Esperimento di configurazione:** ripeti il comando di training di S3 con una nuova cartella `--out` e con il flag `--slippery`, poi usa evaluate e demo sulla nuova cartella. Confronta i risultati dichiarando il cambiamento dell'ambiente e mantenendo uguali gli altri parametri.

**Documenta:** file ricaricato, risultato osservato, un episodio e confronto tra configurazioni. Il rendering è testuale e funziona senza finestra grafica.

## S5 - Esegui il confronto tra reward e preferenze

Il CSV contiene cinque coppie A/B, annotazioni di correttezza e tono e preferenze simulate di riferimento. Non è il dataset di transizioni FrozenLake.

Prima del comando, leggete soltanto domanda e testi e annotate per ogni coppia la scelta del gruppo e il motivo. Consultate punteggi e preferenze di riferimento dopo questa scelta.

```bash
python sol11_reward_ranking.py --data 11_dataset_preferenze_simulate.csv --out preference_comparison.json
```

Apri il JSON prodotto. Lo script conta le parole, calcola correttezza − 0,01 × parole − 0,5 × tono inadeguato, identifica vincitore o pareggio e misura l'accordo con il riferimento simulato.

**Documenta:** scelte del gruppo, vincitori del reward e almeno un disaccordo da spiegare, se osservato. Correttezza e tono sono annotazioni: il programma non ne scopre automaticamente il significato.

Mantieni distinte le due attività: il programma RL aggiorna una politica tabulare; il confronto sulle risposte applica una rubrica e non addestra un reward model o un assistente conversazionale. È a questa seconda attività che si riferiscono i mini-esercizi sulle preferenze.

## S6 - Documenta e consegna

Prepara un report breve con sei sezioni: ambiente, configurazione, comandi eseguiti, risultati RL, confronto delle preferenze, problemi e limiti.

Allega `ambiente.txt`, configurazioni e output significativi: confronto dei seed, curve, valutazione dopo ricaricamento, replay, JSON delle preferenze e giudizi del gruppo. Includi gli script effettivamente usati se hai modificato parametri nel codice, indicando ogni modifica.

Riporta risultati reali e indica le parti rimaste non eseguite. Chiudi eventuali visualizzazioni; gli ambienti sono chiusi dal programma. Conserva gli artefatti della consegna e rimuovi soltanto gli intermedi personali non necessari.

## Codice fornito

Copia ciascun blocco completo nel file indicato, una sola volta. Mantieni i due script separati: ciascuno ha il proprio punto di avvio.

### rl_frozenlake.py

```python
"""Train, save, reload and observe a tabular Q-learning agent in Gymnasium."""
import argparse
import csv
import json
from pathlib import Path

import gymnasium as gym
import numpy as np

MAP = ['SFFF', 'FHFH', 'FFFH', 'HFFG']


def make_env(slippery=False, render=False):
    return gym.make('FrozenLake-v1', desc=MAP, is_slippery=slippery,
                    max_episode_steps=100, render_mode='ansi' if render else None)


def save_csv(path, rows):
    with path.open('w', newline='', encoding='utf-8') as stream:
        writer = csv.DictWriter(stream, fieldnames=list(rows[0]))
        writer.writeheader()
        writer.writerows(rows)


def evaluate(q, slippery, episodes=200, seed=10000):
    """Same evaluation protocol for random actions and a frozen greedy policy."""
    env = make_env(slippery)
    rows, summary = [], {}
    try:
        for policy in ['random', 'q_learning']:
            rng = np.random.default_rng(seed)
            scores, lengths = [], []
            for episode in range(episodes):
                state, _ = env.reset(seed=seed + episode)
                total, steps = 0., 0
                while True:
                    action = int(rng.integers(4)) if policy == 'random' else int(np.argmax(q[state]))
                    state, reward, terminated, truncated, _ = env.step(action)
                    total += float(reward)
                    steps += 1
                    if terminated or truncated:
                        break
                scores.append(total)
                lengths.append(steps)
                rows.append({'policy': policy, 'episode': episode, 'seed': seed + episode,
                             'return': total, 'length': steps, 'success': int(total > 0),
                             'terminated': terminated, 'truncated': truncated})
            summary[policy] = {'success_rate': float(np.mean(np.asarray(scores) > 0)),
                               'mean_return': float(np.mean(scores)),
                               'mean_episode_length': float(np.mean(lengths)), 'episodes': episodes}
    finally:
        env.close()
    return rows, summary


def train_one(folder, seed, episodes, slippery):
    folder.mkdir(parents=True, exist_ok=False)
    rng = np.random.default_rng(seed)
    env = make_env(slippery)
    q = np.zeros((env.observation_space.n, env.action_space.n), dtype=float)
    alpha, gamma = .1, .99
    rows = []
    try:
        for episode in range(episodes):
            epsilon = max(.05, 1 - .95 * episode / max(1, .7 * episodes))
            state, _ = env.reset(seed=seed + episode)
            total, steps = 0., 0
            while True:
                if rng.random() < epsilon:
                    action = int(rng.integers(env.action_space.n))
                else:
                    # Break ties randomly during training; zero values otherwise favor action 0.
                    candidates = np.flatnonzero(q[state] == q[state].max())
                    action = int(rng.choice(candidates))
                next_state, reward, terminated, truncated, _ = env.step(action)
                # A time-limit truncation stops the episode; only true termination cancels bootstrap.
                target = float(reward) + (0. if terminated else gamma * float(q[next_state].max()))
                q[state, action] += alpha * (target - q[state, action])
                state = next_state
                total += float(reward)
                steps += 1
                if terminated or truncated:
                    break
            rows.append({'episode': episode, 'return': total, 'length': steps,
                         'epsilon': epsilon, 'success': int(total > 0),
                         'terminated': terminated, 'truncated': truncated})
    finally:
        env.close()
    np.save(folder / 'q_table.npy', q, allow_pickle=False)
    save_csv(folder / 'training.csv', rows)
    evaluation, summary = evaluate(q, slippery)
    save_csv(folder / 'evaluation.csv', evaluation)
    config = {'environment': 'FrozenLake-v1', 'map': MAP, 'is_slippery': slippery,
              'max_episode_steps': 100, 'seed': seed, 'episodes': episodes,
              'alpha': alpha, 'gamma': gamma, 'epsilon_start': 1., 'epsilon_min': .05,
              'epsilon_decay_episodes': .7 * episodes, 'gymnasium': gym.__version__,
              'numpy': np.__version__, 'nonzero_q_values': int(np.count_nonzero(q)),
              'evaluation_seed_start': 10000, 'evaluation': summary}
    (folder / 'config.json').write_text(json.dumps(config, indent=2))
    # Render a standalone learning curve for the student's report.
    import matplotlib
    matplotlib.use('Agg')
    import matplotlib.pyplot as plt
    window = min(100, episodes)
    moving = np.convolve([r['success'] for r in rows], np.ones(window) / window, mode='valid')
    fig, ax = plt.subplots(figsize=(8, 4))
    ax.plot(np.arange(window - 1, episodes), moving)
    ax.set(xlabel='Episodio di training', ylabel=f'Success rate, finestra {window}', ylim=(0, 1.05))
    ax.set_title(f'Q-learning, seed={seed}, slippery={slippery}')
    fig.tight_layout()
    fig.savefig(folder / 'learning_curve.png', dpi=150)
    plt.close(fig)
    return {'seed': seed, 'nonzero_q_values': config['nonzero_q_values'], **summary}


def replay(folder):
    config = json.loads((folder / 'config.json').read_text())
    q = np.load(folder / 'q_table.npy', allow_pickle=False)
    if q.shape != (16, 4) or not np.isfinite(q).all():
        raise ValueError('Tabella Q non valida')
    env = make_env(config['is_slippery'], render=True)
    text = []
    try:
        state, _ = env.reset(seed=10000)
        text.append(env.render())
        while True:
            action = int(np.argmax(q[state]))
            state, reward, terminated, truncated, _ = env.step(action)
            text.append(f'azione={action}, stato={state}, reward={reward}, terminated={terminated}, truncated={truncated}')
            text.append(env.render())
            if terminated or truncated:
                break
    finally:
        env.close()
    result = '\n'.join(text)
    (folder / 'episode.txt').write_text(result, encoding='utf-8')
    print(result)


def main():
    parser = argparse.ArgumentParser()
    sub = parser.add_subparsers(dest='command', required=True)
    p = sub.add_parser('train')
    p.add_argument('--out', type=Path, default=Path('rl_output'))
    p.add_argument('--episodes', type=int, default=6000)
    p.add_argument('--seeds', type=int, nargs='+', default=[42, 43, 44])
    p.add_argument('--slippery', action='store_true')
    p = sub.add_parser('evaluate')
    p.add_argument('--run', type=Path, required=True)
    p = sub.add_parser('demo')
    p.add_argument('--run', type=Path, required=True)
    args = parser.parse_args()
    if args.command == 'train':
        if args.episodes < 1 or len(set(args.seeds)) != len(args.seeds):
            raise ValueError('Servono episodi positivi e seed distinti')
        args.out.mkdir(parents=True, exist_ok=False)
        results = [train_one(args.out / f'seed_{s}', s, args.episodes, args.slippery) for s in args.seeds]
        (args.out / 'comparison.json').write_text(json.dumps(results, indent=2))
        print(json.dumps(results, indent=2))
    elif args.command == 'evaluate':
        config = json.loads((args.run / 'config.json').read_text())
        q = np.load(args.run / 'q_table.npy', allow_pickle=False)
        rows, summary = evaluate(q, config['is_slippery'])
        save_csv(args.run / 'evaluation_reloaded.csv', rows)
        (args.run / 'evaluation_reloaded.json').write_text(json.dumps(summary, indent=2))
        print(json.dumps(summary, indent=2))
    else:
        replay(args.run)


if __name__ == '__main__':
    main()
```

### sol11_reward_ranking.py

```python
"""Testi concreti, giudizi dichiarati, reward e preferenze: nessun training RL."""
import argparse
import csv
import json
from dataclasses import dataclass
from pathlib import Path

@dataclass
class Answer:
    text: str
    correctness: float  # annotazione manuale del valutatore, non misura automatica
    bad_tone: bool = False


def reward(answer):
    return answer.correctness - .01*len(answer.text.split()) - .5*answer.bad_tone


def main():
    parser=argparse.ArgumentParser()
    parser.add_argument('--data',type=Path,required=True)
    parser.add_argument('--out',type=Path,default=Path('preference_comparison.json'))
    args=parser.parse_args()
    with args.data.open(newline='',encoding='utf-8') as stream:
        records=list(csv.DictReader(stream))
    if not records:
        raise ValueError('CSV preferenze vuoto')
    pairs={}
    for record in records:
        pair=pairs.setdefault(record['pair_id'],{})
        if record['response_id'] in pair:
            raise ValueError('Risposta duplicata nella coppia')
        if not 0 <= float(record['correctness']) <= 1 or record['bad_tone'] not in ('0','1'):
            raise ValueError('Annotazioni fuori scala')
        pair[record['response_id']]=record
    rows = []
    for i,pair in pairs.items():
        if set(pair) != {'A','B'}:
            raise ValueError('Ogni coppia deve avere A e B')
        record_a,record_b=pair['A'],pair['B']
        if record_a['question'] != record_b['question']:
            raise ValueError('Domande diverse nella stessa coppia')
        a=Answer(record_a['text'],float(record_a['correctness']),record_a['bad_tone']=='1')
        b=Answer(record_b['text'],float(record_b['correctness']),record_b['bad_tone']=='1')
        human,reason=record_a['preferred_reference'],record_a['reason_reference']
        ra, rb = reward(a), reward(b)
        chosen = 'A' if ra>rb else ('B' if rb>ra else 'pari')
        rows.append({'pair': i, 'question': record_a['question'], 'A': a.text, 'B': b.text,
                     'words_A': len(a.text.split()), 'words_B': len(b.text.split()),
                     'reward_A': ra, 'reward_B': rb, 'reward_choice': chosen,
                     'human_choice_simulated': human, 'reason': reason,
                     'agreement': chosen == human})
    args.out.write_text(json.dumps(rows, indent=2, ensure_ascii=False),encoding='utf-8')
    print(json.dumps(rows, indent=2, ensure_ascii=False))
    print('Accordo:', sum(row['agreement'] for row in rows), '/', len(rows))
    print('Le preferenze sono una fixture didattica. Raccolta reale, reward model e politica non sono stati addestrati.')


if __name__ == '__main__':
    main()
```

## Riferimenti

- [FrozenLake](https://gymnasium.farama.org/environments/toy_text/frozen_lake/)
- [API Gymnasium](https://gymnasium.farama.org/api/env/)
- [Terminated, truncated e limiti temporali](https://gymnasium.farama.org/tutorials/gymnasium_basics/handling_time_limits/)
