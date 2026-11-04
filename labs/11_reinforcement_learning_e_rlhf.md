# Lab 11 - Eseguire un ambiente RL e addestrare un agente

## Cosa devi eseguire

In questo laboratorio **scrivi ed esegui Python**. La parte principale consiste nell'avviare FrozenLake con Gymnasium, osservare episodi, addestrare un agente con Q-learning, salvarlo e confrontarlo con una politica casuale. Nella seconda parte esegui l'analisi delle preferenze del CSV per capire i limiti di una funzione di reward.

Il report Markdown documenta le esecuzioni. La consegna richiede anche sorgenti, log, tabella Q, grafico di apprendimento e risultati prodotti dal programma.

## Preparazione e mappa S1–S6

Crea lavoro_lab11 direttamente in materiali_didattici, attiva Python 3.12 e salva i due blocchi nei file indicati. L'input CSV si trova in ../labs/. Prepara le dipendenze prima della lezione:

```bash
python -m pip install 'gymnasium>=1.0,<2' numpy matplotlib
```

| Passaggio | Funzione o file             | Evidenza                                               |
| --------- | --------------------------- | ------------------------------------------------------ |
| S1        | make_env, reset e step      | Episodio osservato e rendering testuale                |
| S2        | evaluate con azioni casuali | Baseline su 200 episodi                                |
| S3        | train_one                   | Aggiornamenti Q, log, configurazione e tabella salvata |
| S4        | evaluate, replay e CLI      | Valutazione dopo ricaricamento, curva e comportamento  |
| S5        | sol11_reward_ranking.py     | Calcolo effettivo sulle cinque coppie del CSV          |
| S6        | Report ed evidenze          | Risultati osservati e limiti distinti per attività     |

## Prima del codice: disegna un episodio

Disegna la griglia, scegli una sequenza di azioni e prevedi dove finirà. Distingui stato, azione, reward e condizione di fine episodio. Poi confronta la previsione con un episodio realmente eseguito.

Prima di S5, calcola anche il caso A=(correttezza 1, 80 parole), B=(0,6, 10 parole) con reward = correttezza − 0,01 × parole. Conserverai questo confronto separato dall'addestramento dell'agente.

## Costruzione guidata

### S1 - Avvia FrozenLake e osserva le transizioni

**File da costruire:** `rl_frozenlake.py`, inizialmente con la sola creazione dell'ambiente e la gestione di un episodio.

1. Crea l'ambiente usando Gymnasium, la mappa assegnata e `is_slippery=False`.
2. Individua cosa restituiscono reset e step. Prepara variabili distinte per observation, reward, terminated e truncated.
3. Esegui alcune azioni e stampa stato iniziale, azione, stato successivo e reward. Usa il rendering testuale per vedere la posizione sulla griglia.
4. Chiudi l'episodio quando terminated oppure truncated è vero e chiudi l'ambiente al termine del programma.

**Checkpoint da mostrare:** output di almeno un episodio eseguito, con i passi osservati. Uno schema disegnato da solo non dimostra che l'ambiente sia stato avviato.

**Indizio:** terminated segnala la fine prevista dal compito; truncated segnala un'interruzione come il limite di passi.

### S2 - Misura una baseline casuale

1. Scrivi una funzione che scelga azioni casuali e accumuli reward e numero di passi per episodio.
2. Esegui 200 episodi con seed di valutazione fissati, a partire da 10000.
3. Salva una riga per episodio con politica, seed, ritorno totale, durata, successo e causa di fine.
4. Calcola success rate, ritorno medio e lunghezza media. Definisci successo come raggiungimento dell'obiettivo.

**Checkpoint:** il programma produce risultati e un CSV. Un episodio riuscito non basta a descrivere la qualità della politica.

**Indizio:** usa gli stessi parametri dell'ambiente e lo stesso protocollo per la futura politica addestrata. Il seed del generatore di azioni va controllato separatamente.

### S3 - Implementa e addestra Q-learning

1. Crea una tabella Q con una riga per stato e una colonna per azione, inizialmente a zero.
2. Implementa una scelta epsilon-greedy: con probabilità epsilon esplora, altrimenti scegli un'azione con Q massimo. Gestisci i pareggi senza favorire sempre la stessa azione durante il training.
3. Aggiorna Q a ogni transizione con la regola: valore attuale + alpha × (target − valore attuale). Il target usa reward e gamma × massimo Q dello stato successivo; se la transizione è terminale, usa il solo reward.
4. Parti da alpha 0,1, gamma 0,99, epsilon 1 che scende fino a 0,05. Fissa questi parametri prima del confronto finale. Esegui 6000 episodi per ciascuno dei seed 42, 43 e 44.
5. Registra ritorno, durata, epsilon e successo di ogni episodio. Salva la tabella Q e la configurazione di ciascuna esecuzione in cartelle distinte.

**Checkpoint:** esistono un log di training e una tabella Q salvata. Controlla se i valori sono cambiati; se restano tutti a zero, indaga esplorazione e raggiungimento del goal.

**Indizio:** il limite di tempo interrompe l'episodio, ma non rende automaticamente terminale lo stato nel target Q-learning. Tratta separatamente terminated e truncated.

### S4 - Ricarica l'agente, valuta e mostra il comportamento

1. Ricarica la tabella Q da disco in una nuova esecuzione del programma.
2. Valuta la politica greedy, senza esplorazione e senza aggiornare Q, sui 200 episodi del protocollo S2.
3. Confronta agente e baseline casuale; riporta i risultati dei tre seed di training senza scegliere soltanto il migliore.
4. Disegna il success rate di training su finestra mobile di 100 episodi. Mostra anche un replay testuale dell'agente caricato e salvalo in un file.
5. Se resta tempo, ripeti da una cartella nuova con `is_slippery=True`, allenando e valutando nello stesso ambiente modificato. Conserva il confronto tra configurazioni.

**Checkpoint:** puoi mostrare il grafico e l'episodio dell'agente salvato, oltre a una tabella random/Q-learning. Non sono richiesti valori numerici prestabiliti: se il training non migliora, documenta la causa osservata.

**Indizio:** gli episodi di valutazione non aggiornano la politica. Valutare sulla stessa mappa con altri seed non dimostra generalizzazione a mappe nuove.

## rl_frozenlake.py - S1–S4

Il codice usa una mappa locale fissa. Non scarica dati e non usa GPU. La tabella Q viene aggiornata nell'ambiente, salvata e poi ricaricata per eseguire la politica. I risultati variano con seed e configurazione: non sono garantiti da questo documento.

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

### Esecuzione del percorso RL

```bash
python rl_frozenlake.py train --out rl_output --episodes 6000 --seeds 42 43 44
python rl_frozenlake.py evaluate --run rl_output/seed_42
python rl_frozenlake.py demo --run rl_output/seed_42
```

Ripeti evaluate e demo per gli altri seed quando vuoi osservare ciascuna esecuzione. Le valutazioni usano una politica congelata e gli stessi episodi della baseline, senza aggiornare Q. Training.csv, evaluation.csv, q_table.npy, config.json, learning_curve.png ed episode.txt sono evidenze da interpretare, non file da sostituire con valori attesi.

Il grafico mostra il successo durante il training con esplorazione; la tabella finale mostra la valutazione greedy. Sono protocolli distinti. Il confronto riguarda la stessa mappa e non misura generalizzazione su mappe nuove.

Per l'estensione con scivolamento usa una cartella diversa:

```bash
python rl_frozenlake.py train --out rl_slippery_output --episodes 6000 --seeds 42 43 44 --slippery
```

Questo nuovo comando allena e valuta nell'ambiente modificato. Non confrontare numeri senza dichiarare is_slippery, seed e budget di training.

### S5 - Esegui il confronto tra reward e preferenze

**File da costruire:** `sol11_reward_ranking.py`. La scrittura e l'esecuzione dello script fanno parte della consegna.

1. Leggi le cinque coppie del CSV distribuito. Controlla che ogni pair_id abbia A e B per la stessa domanda.
2. Prima di vedere punteggi e preferenze di riferimento, fate scegliere al gruppo A o B leggendo solo domanda e testi; conservate scelta e motivo.
3. Conta le parole dai testi e applica la rubrica: correttezza − 0,01 × parole − 0,5 × tono inadeguato.
4. Calcola vincitore, pareggi e accordo con la preferenza simulata di riferimento; salva il confronto prodotto dal programma. Confrontalo anche con i giudizi raccolti dal gruppo.
5. Spiega il caso della risposta breve ma scorretta e proponi una revisione della rubrica, distinguendo ciò che hai eseguito da ciò che proponi.

**Checkpoint:** la consegna contiene l'output dello script e i giudizi indipendenti del gruppo. Correttezza e tono sono annotazioni, non misure semantiche scoperte dal programma.

**Indizio:** in S3 hai addestrato un agente tabulare. In S5 esegui un confronto aritmetico su risposte: questa seconda attività non addestra un reward model né un assistente conversazionale. È a questo confronto che si riferiscono i mini-esercizi sulle preferenze.

## sol11_reward_ranking.py - S5

Prima dell'esecuzione raccogli le preferenze del gruppo leggendo solo domande e testi. Il CSV è una fixture con cinque coppie A/B e giudizi simulati; correctness e bad_tone non sono stimati automaticamente. Il numero di parole è contato dai testi.

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

```bash
python sol11_reward_ranking.py --data ../labs/11_dataset_preferenze_simulate.csv --out preference_comparison.json
```

### Interpretazione

Nel caso su carta A vale 0,20 e B 0,50: la penalità può favorire una risposta breve ma incompleta. Nella seconda coppia del CSV emerge un disaccordo tra reward e preferenza simulata. La raccolta delle preferenze del gruppo resta un'evidenza separata dal giudizio di riferimento distribuito.

S1–S4 addestrano una politica tabulare Q-learning. S5 esegue un confronto aritmetico: questo script non addestra un reward model né una politica linguistica RLHF. I mini-esercizi sul calcolo e sul confine del confronto si riferiscono a S5.

## S6 - Consegna e conservazione

Conserva sorgenti, versioni dei pacchetti, log dei tre seed, Q-table, curve, valutazione dopo ricaricamento, replay e confronto delle preferenze. Spiega quali dati dimostrano un aggiornamento della politica e quali dimostrano soltanto l'applicazione di una regola di reward.

Gli ambienti sono chiusi con env.close; rimuovi soltanto output intermedi dopo avere conservato quelli della consegna. La presenza del codice in questa soluzione non certifica che la tua esecuzione sia stata completata: annota risultati ed errori effettivi.

| Da consegnare | Evidenza richiesta |
| --- | --- |
| Sorgenti | Programma RL e programma reward/preferenze scritti dallo studente |
| Ambiente | Versioni, mappa, parametri, seed e comandi realmente usati |
| Training | CSV episodi, Q-table e configurazione per i tre seed |
| Valutazione | Tabella random/Q-learning e risultati dopo ricaricamento |
| Comportamento | Grafico di apprendimento e replay testuale |
| Preferenze | Scelte motivate del gruppo e output del confronto sul CSV |
| Report breve | Interpretazione dei risultati, problemi incontrati e limiti |

Chiudi gli ambienti Gymnasium e conserva gli artefatti finali. Un report senza sorgenti e risultati di esecuzione non completa il laboratorio. Se una postazione non dispone delle dipendenze, lavora in coppia su una postazione predisposta e documenta l'ambiente usato; indica le parti rimaste non eseguite.

## Sequenza di avvio dei programmi che costruisci

Prevedi queste modalità nell'interfaccia dei tuoi script. Dopo avere scritto il codice, esegui dalla cartella di lavoro:

1. `python rl_frozenlake.py train --out rl_output --episodes 6000 --seeds 42 43 44` per allenare e salvare le tre esecuzioni.
2. `python rl_frozenlake.py evaluate --run rl_output/seed_42` per valutare Q ricaricata; ripeti per gli altri seed.
3. `python rl_frozenlake.py demo --run rl_output/seed_42` per osservare e salvare un episodio.
4. `python sol11_reward_ranking.py --data ../labs/11_dataset_preferenze_simulate.csv --out preference_comparison.json` per produrre il confronto delle preferenze.

Per ripetere il training scegli una cartella output nuova e conserva quella della prova precedente. La soluzione mostra una possibile implementazione dell'interfaccia; questi comandi di avvio non contengono la logica da costruire.

## Quando consultare la soluzione

Tenta prima S1–S3 e conserva il tuo codice. Usa la soluzione per confrontare il passaggio in cui sei bloccato, poi riprendi il tuo programma. Il codice completo resta nel file soluzione, separato da questa guida.

Riferimenti: [FrozenLake](https://gymnasium.farama.org/environments/toy_text/frozen_lake/), [API Gymnasium](https://gymnasium.farama.org/api/env/), [gestione dei limiti temporali](https://gymnasium.farama.org/tutorials/gymnasium_basics/handling_time_limits/).
[API Env](https://gymnasium.farama.org/api/env/), [terminated e truncated](https://gymnasium.farama.org/tutorials/gymnasium_basics/handling_time_limits/).
