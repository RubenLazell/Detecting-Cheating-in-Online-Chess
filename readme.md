# Detecting Cheating in Online Chess

**A data-driven approach to identifying engine assistance in Titled Tuesday tournaments.**

BSc Data Science honours dissertation — Ruben Lazell, Edinburgh Napier University, April 2025.

This project analyses **~880,000 games** from every Chess.com Titled Tuesday tournament played between **28 October 2014 and 24 September 2024**, and applies three independent detection methods to flag games and players whose play is statistically inconsistent with unassisted human chess.

---

## Why Titled Tuesday?

Cheating research in chess tends to sit at one of two extremes: over-the-board scandals, where the evidence is usually circumstantial, or casual online play, where incidents are dismissed because nothing is at stake. Titled Tuesday sits in the middle. It is a weekly, high-stakes, invitation-only event for titled players, with prize money and reputations on the line, played entirely online. Chess.com has banned players from it, but the details of those bans have never been published.

The full archive is also publicly downloadable, game by game, which makes it a rich and completely unexplored dataset. No prior study in the literature had used it.

---

## Research questions

1. To what extent can move time be used as an indicator of engine assistance in competitive online chess?
2. How significant is the issue of cheating in Titled Tuesday events?
3. Which in-game factors are most predictive of engine usage in online chess?

---

## The dataset

| | |
|---|---|
| Source | Chess.com public Titled Tuesday PGN archives |
| Coverage | 28 Oct 2014 – 24 Sep 2024 (every tournament) |
| Games | ~880,000 |
| Raw size | ~2 GB of combined PGN |
| Mean game length | 43.9 moves |
| Mean games per tournament | 1,440.5 |

Every tournament was downloaded individually — a process that took around two months — then merged into a single PGN with a Python script. Two versions were produced: one with clock annotations stripped, for engine analysis, and one with clock annotations retained, for the temporal analysis.

`combined_titled_tuesday.zip` in this repository is that merged dataset, tracked via Git LFS.

---

## Method

### 1. Engine analysis — accuracy and centipawn loss

Every game was replayed through **Stockfish** and scored move by move. The scoring pipeline is adapted from the open-source [`chess_accuracy`](https://github.com/Asavis) project by GitHub user Asavis, with the core evaluation functions retained and the surrounding tooling rewritten for batch use.

- **Win probability** — centipawn evaluations are mapped to a win percentage through a logistic function, so that a blunder is measured by how much winning chance it throws away rather than by raw centipawns.
- **Move accuracy** — the drop in win probability (ΔW) is converted to a 0–100 accuracy score via exponential decay. Chess error is non-linear (missing mate in 1 is not the same as a slightly worse pawn structure), so a linear penalty would misrepresent severity.
- **Harmonic mean** — averages a player's move accuracies in a way that punishes low scores more heavily than it rewards high ones, emphasising *consistency* over occasional brilliance.
- **Volatility-weighted mean** — weights each move by how volatile the position was, measured as the standard deviation of win chances across the branching lines. Playing well in a quiet position is easy; repeatedly finding the only good move in a sharp one is not.
- **Final accuracy** = mean of the harmonic and volatility-weighted means.

Custom additions on top of the original script: full PGN metadata extraction, automatic CSV output (`combined_stockfish_results.csv`), and multi-game batch looping so the whole 2 GB archive runs in one pass.

**Compute.** Locally, six games took 30.76 seconds — roughly **48.6 days** for the full dataset. The job was moved to the Edinburgh Napier HPC cluster and batched with `sbatch`, and the engine depth was set to **13**, which Ferreira (2013) estimates at ~2459 Elo. That is sufficient to approximate strong titled play in a 3+1 blitz format, where play is measurably weaker than at classical time controls. Total runtime: **~3 days**.

### 2. Temporal analysis — how long each move took

Normal players think: they burn time in critical positions and move instantly in obvious ones, producing a **high standard deviation** of move times. An engine user tends to take a similar amount of time every move — glancing at another tab, copying the recommendation — producing a **low, flat** distribution.

Two metrics were computed per player, per tournament:

- **Standard deviation of move times.**
- **ANTSC (Average Normalised Time Suspicion Coefficient)** — the standard deviation z-score normalised against the player's own history and then inverted, so that *high* ANTSC means suspicious and *low* means human. Normalising against the player's own baseline controls for individual playing style; a value above 1.96 is significant at the 95% level.

Aggregating over whole tournaments rather than single games is deliberate — one fast game is noise, a whole fast tournament is a signal.

### 3. Unsupervised anomaly detection — Isolation Forest

No trustworthy labelled dataset of confirmed cheaters exists publicly, and supervised models trained on noisy labels tend to learn the noise. So the third method is unsupervised: an **Isolation Forest** (Liu et al., 2008), chosen for its linear time complexity and low memory footprint at ~880,000 rows.

**Filters applied first:**
- Move count between 25 and 100 — very short games are opening theory or timeouts, very long ones are dominated by trivial moves in a decided position.
- Combined accuracy of both players ≥ 170, so games where one side simply collapsed are excluded.
- Both players rated ≥ 1900, which removes the untitled players who slipped into early tournaments.
- Drawn games excluded — if one side has engine help, the game should usually be decisive.

**Features** (standardised with `StandardScaler`, since raw accuracy/Elo ≈ 0.03 while move count ranges 25–100 and would otherwise dominate):
- accuracy relative to Elo
- average centipawn loss relative to Elo
- move count

Normalising performance against rating is the key idea here: 99% accuracy from a 2000-rated player is far more remarkable than the same score from a 3000-rated player. **Contamination is set to 0.01**, flagging the most extreme 1% of games — deliberately generous, so that repeat offenders can be spotted across the resulting set.

The flagged games are then aggregated into a **suspicion table** per player: games flagged, total games played, mean Elo across flagged games, and a suspicion rate.

---

## Findings

**No player was identified as a definite cheater, and none is named as one here.** That is by design — the study is not a prosecution tool.

What the pipeline does do is reduce a dataset of ~880,000 games to a shortlist one-hundredth of the size, in which every entry trips at least one of the three detectors.

Selected results:

- **Filtering matters more than it looks.** The first pass at "highest accuracy games" returned 2-move timeouts, pre-agreed draws, a 280-move game where a winning player deliberately refused to mate his opponent (every stalling move preserved the evaluation, so the engine scored it 100%), and a Carlsen game that reached a dead-drawn king-and-pawn endgame on move 25 and shuffled to a threefold repetition. 65 of the first 100 hits were draws. Each of these forced a refinement of the criteria.
- **Case study — an outlier tournament.** One flagged player averaged ~98% accuracy across a December 2020 tournament, against a personal baseline of 85–89%. Their ANTSC for that same event was ≈ −0.3, i.e. entirely normal timing. Cross-referencing the two methods suggests a very good day rather than assistance.
- **Statistically significant timing anomalies had mundane explanations.** Two ANTSC scores above 2.0 turned out to be single games that ran past midnight GMT, which the parser treated as separate one-game "tournaments". An ANTSC of 3.75 turned out to be a player who made three instant moves and resigned. Both are genuine bugs-or-artefacts worth documenting, and both are caught by the fact that the metrics are designed to be read in context.
- **The suspicion table's top 30 entries are all players with 18 or fewer Titled Tuesday games** — small-sample noise, or possibly accounts that no longer play there.

**On the research questions:** move time is a usable signal, and works best as an adjudicator for players already flagged by accuracy rather than as a primary detector (RQ1). Cheating in Titled Tuesday is undeniably present, though its prevalence cannot be quantified without ground truth (RQ2). No single feature is decisive — confidence requires high accuracy, low centipawn loss, anomalous timing, and performance above one's rating band appearing *together* (RQ3).

---

## Limitations

- **Engine depth 13.** The strongest Titled Tuesday players exceed 3000 Elo, above depth 13's estimated strength. Depth 20 across the full dataset would have taken months even on the cluster.
- **No confidence scores.** Isolation Forest is unsupervised, so flagged games carry no probability of cheating and cannot be validated against ground truth.
- **Clock annotation inconsistency** across a decade of archives required substantial cleaning, and date handling around midnight GMT remains imperfect.
- **Chess literacy assumed.** Interpreting the outputs meaningfully needs at least a basic understanding of the game.

---

## Ethics

All data is public, downloadable by anyone from Chess.com, and identifies players by username rather than legal name. No players were contacted, no personal data was processed, and — per Edinburgh Napier SCEBE ethics guidance — **no player is named as a cheater anywhere in this work**, including those flagged by the models. Flagging is an invitation to investigate, not an accusation.

---

## Repository contents

| File | Description |
|---|---|
| `Dissertation_final_draft_40679914.pdf` | Full dissertation (76 pages), including literature review, methodology, results and appendices with all code listings. |
| `Dissertation Poster - 40679914.pdf` | One-page poster summary of the project. |
| `Supplementary_Files_40679914.zip` | Analysis scripts and supporting materials. |
| `combined_titled_tuesday.zip` | Merged Titled Tuesday PGN dataset (Git LFS). |

### Pipeline scripts

| Stage | What it does | Output |
|---|---|---|
| PGN combiner | Merges every downloaded tournament PGN into one file; strips clock annotations for the engine pass. | `combined_titled_tuesday.pgn` |
| Stockfish analysis | Runs every game through Stockfish; computes per-player accuracy, ACPL and estimated Elo. Built for `sbatch` batch execution. | `combined_stockfish_results.csv` |
| Accuracy filter | Extracts the highest-accuracy games subject to the move-count, combined-accuracy and result filters. | Top-100 shortlist |
| `time_analysis.py` | Takes a username, extracts per-move times from the clock-annotated PGN, and plots standard deviation and ANTSC by tournament. | Per-player timing charts |
| Isolation Forest | Elo-normalised unsupervised anomaly detection over the results CSV. | Flagged games + suspicion table |

---

## Running it

```bash
git clone https://github.com/RubenLazell/Detecting-Cheating-in-Online-Chess.git
cd Detecting-Cheating-in-Online-Chess
git lfs pull
```

**Requirements:** Python 3, a local Stockfish binary, and `python-chess`, `pandas`, `scikit-learn` and `matplotlib`.

The engine analysis expects a PGN file in the same directory and is invoked from the command line with the target file as an argument. Be aware of the runtime: on a single machine, the full archive is measured in weeks, not hours. Run it on a subset, or on a cluster.

---

## References

- Liu, F.T., Ting, K.M. & Zhou, Z.-H. (2008). *Isolation Forest.* IEEE ICDM.
- Regan, K. & Haworth, G. (2011). *Intrinsic Chess Ratings.* AAAI.
- Guid, M. & Bratko, I. (2006). *Computer Analysis of World Chess Champions.* ICGA Journal.
- Ferreira, D. (2013). *The Impact of the Search Depth on Chess Playing Strength.* ICGA Journal.
- Pedregosa, F. et al. (2011). *Scikit-learn: Machine Learning in Python.* JMLR.
- Asavis (2024). *chess_accuracy* — the basis for the engine evaluation functions.

Full bibliography in the dissertation PDF.

---

## Licence and use

Academic work, shared for reference and reuse. If you build on the methods or the dataset, a citation of the dissertation is appreciated.
