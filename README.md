# TakeMeter — Discourse Quality Classifier for r/nba

A fine-tuned text classifier that sorts r/nba comments into three discourse
types: **`analysis`**, **`hot_take`**, and **`reaction`**. Built for AI201
Project 3.

> **Before submitting:** fill the one remaining bracket in the Data collection
> section (the true source of the dataset). All numbers below are final.

---

## Community choice and reasoning

I chose **r/nba** because its comments span the full range of discourse quality —
from detailed tactical breakdowns to one-word celebrations — which gives a
classifier something real to distinguish. I collected during the aftermath of the
Knicks' 2026 title run, when the subreddit was especially active and all three
discourse types were well-represented. The "hot take vs. real analysis"
distinction is also one r/nba users themselves invoke constantly, so the labels
reflect a real community norm rather than an imposed one.

## Label taxonomy

| Label | Definition | Example |
|---|---|---|
| `analysis` | A structured argument backed by specific, verifiable evidence (stats, tactics, matchup detail). Reasoning stands without the opinion. | *"The swing was their drop coverage — they held SA to 41% in the paint over the last three games."* |
| `hot_take` | A bold, confident opinion asserted without real evidence, or with only a decorative stat. Asserts rather than argues. | *"This team is already a dynasty. Three-peat incoming."* |
| `reaction` | An immediate emotional response to a moment, with little or no argument. Expresses a feeling, not a claim. | *"LETS GOOO I waited 25 years for this."* |

*(Second example per label is in `planning.md`; swap in real comments from your
CSV before submitting if your grader wants both examples here too.)*

The hardest boundary is **`analysis` vs. `hot_take`** when a confident opinion
drops in one stat for credibility. Decision rule: if the evidence is doing
argumentative work it's `analysis`; if the stat is decoration on an assertion
it's `hot_take`.

## Data collection

- **Source:** `[DESCRIBE HOW YOU ACTUALLY OBTAINED THESE EXAMPLES. The project
  requires real public posts/comments from r/nba. Do not claim Reddit scraping
  unless that is what happened — state the true source.]`
- **Dataset size:** 226 labeled rows. Note: 14 were labeled `agenda_posting`, a
  fourth label not in the notebook's label map, so they were dropped at load time,
  leaving **212 examples** across three labels.
- **Label distribution (used, 212):** analysis 88, reaction 69, hot_take 55.
- **Process:** Each row labeled by hand against the definitions above; borderline
  cases logged in a `notes` column.
- **Three difficult-to-label examples:** `[FILL IN]` — pull from `planning.md`.

## Fine-tuning approach

- **Base model:** `distilbert-base-uncased` (HuggingFace), with a 3-class
  classification head.
- **Training setup:** 3 epochs, learning rate 2e-5, batch size 16, weight decay
  0.01, 50 warmup steps, on a Colab T4 GPU. 70/15/15 stratified train/val/test
  split.
- **Key hyperparameter decision:** I kept the notebook defaults (3 epochs, LR
  2e-5, batch 16). With only ~150 training examples, 3 epochs is enough to fit
  without heavy overfitting. Notably, I did *not* increase epochs even though
  performance was poor — the model had collapsed to predicting the majority class
  (`analysis`), and more training time deepens a majority-class collapse rather
  than fixing it. The real lever here is the data distribution, not the number of
  epochs, so I left training settings at their defaults.

## Baseline description

Zero-shot baseline using Groq's `llama-3.3-70b-versatile`, prompted with the
three label definitions and one example each, instructed to output only the label
name. Run on the **same locked test set** as the fine-tuned model. The exact
prompt is in `planning.md`.

## Evaluation report

### Overall accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq) | 0.625 |
| Fine-tuned DistilBERT | 0.531 |

The zero-shot baseline **outperformed** the fine-tuned model by 9.4 points. With
only ~200 examples, fine-tuning did not beat a strong general LLM on this task —
a legitimate and interesting outcome discussed in the reflection below.

### Per-class metrics

| Label | Model | Precision | Recall | F1 |
|---|---|---|---|---|
| analysis | baseline | 1.00 | 0.23 | 0.38 |
| analysis | fine-tuned | 0.43 | 1.00 | 0.60 |
| hot_take | baseline | 0.45 | 1.00 | 0.62 |
| hot_take | fine-tuned | 0.00 | 0.00 | 0.00 |
| reaction | baseline | 0.89 | 0.80 | 0.84 |
| reaction | fine-tuned | 1.00 | 0.20 | 0.33 |

Macro F1: baseline **0.61**, fine-tuned **0.31**.

**Reading these:** the model **collapsed to predicting `analysis` almost always**
(it predicted `analysis` for 30 of 32 test examples). That's why `analysis` recall
is a perfect 1.00 but its precision is only 0.43 — it's right about analysis only
because it labels nearly everything analysis. It **never predicted `hot_take` once**
(F1 = 0.00), and caught only 2 of 10 `reaction` posts. This is a degenerate,
majority-class-collapse result, not a model that learned three distinctions.

### Confusion matrix (fine-tuned, test set)

Rows = true label, columns = predicted label.

| true \ pred | analysis | hot_take | reaction |
|---|---|---|---|
| **analysis** | 13 | 0 | 0 |
| **hot_take** | 9 | 0 | 0 |
| **reaction** | 8 | 0 | 2 |

The entire `hot_take` and `reaction` columns are nearly empty: the model funneled
9 of 9 hot takes and 8 of 10 reactions into `analysis`. All 17 errors are
"predicted analysis." This single column tells the whole story — the model
defaulted to one class.

### Three wrong predictions, analyzed

All 17 errors share one direction: the true label was `hot_take` or `reaction`,
and the model predicted `analysis`. Three representative cases:

1. **"Wemby is overhyped"** — true `hot_take`, predicted `analysis` (conf 0.37).
   A bare, evidence-free opinion — the textbook hot take. The model still defaulted
   to `analysis`, at a confidence barely above random (0.33 for 3 classes). It
   isn't reasoning about the comment; it's guessing its default class.

2. **"I'm so emotional right now"** — true `reaction`, predicted `analysis`
   (conf 0.37). Pure emotional reaction with zero argument. There is nothing
   analysis-like here, which makes the `analysis` prediction a clear sign the
   model has no working concept of `reaction` beyond the two it happened to catch.

3. **"Brunson is a playoff choker because he had 5 turnovers."** — true `hot_take`,
   predicted `analysis` (conf 0.42). This is the stat-garnished hot take from my
   planning doc: a decorative number ("5 turnovers") attached to an assertion. A
   human applies the "is the evidence load-bearing?" rule and calls it `hot_take`;
   the model sees a number and (like its default) says `analysis`. This is the one
   error that looks like a real boundary confusion rather than pure class collapse.

**Diagnosis:** errors 1–2 are symptoms of majority-class collapse (the model
predicts `analysis` regardless of content). Error 3 is the genuine
`analysis`/`hot_take` boundary I predicted would be hard. The fix is the same
either way: a balanced training set and more `hot_take`/`reaction` signal, plus
harder `analysis`-vs-`hot_take` pairs so a stray number stops reading as evidence.

### Sample classifications

Real predictions from the fine-tuned model (every prediction was `analysis` —
itself the finding):

| Comment | True | Predicted | Confidence |
|---|---|---|---|
| "Wemby is overhyped" | hot_take | analysis | 0.37 |
| "I'm buying a Brunson jersey" | reaction | analysis | 0.38 |
| "The Knicks are the realest team in the league" | hot_take | analysis | 0.52 |
| "OG Anunoby is a top-10 defender" | hot_take | analysis | 0.40 |
| "I'm so emotional right now" | reaction | analysis | 0.37 |

On the 13 test `analysis` comments the model was "correct," but only because
`analysis` is the label it assigns to everything — e.g. a genuine breakdown
comment is labeled `analysis` at low confidence, the right answer for the wrong
reason. The uniformly low confidences (~0.37–0.52) confirm the model is near
guessing, not discriminating.

## Reflection: what the model learned vs. what I intended

I intended the model to separate comments by *whether evidence does
argumentative work*. What it actually learned was far blunter: **predict
`analysis` for almost everything.** On the test set it labeled 30 of 32 comments
`analysis`, never once predicted `hot_take`, and caught only 2 of 10 `reaction`
posts. It did not learn three distinctions — it learned one default guess.

This is a majority-class collapse. With a small dataset (~200 examples) and
`analysis` as the most common label, the model found that always guessing
`analysis` was the lowest-effort way to be right a good chunk of the time — and
fine-tuning never pushed it off that local optimum. The `hot_take` F1 of 0.00 is
the clearest symptom: the model treats `hot_take` as if the class doesn't exist.

The gap between intent and reality is total here: I wanted a classifier of
evidential quality, and I got a constant function with a thin veneer of accuracy.
The zero-shot Groq baseline (62.5%) beat it (46.9%) precisely because the baseline
*didn't* collapse — it still distributed predictions across all three labels.

What would fix it: (1) rebalance the training set so `analysis` isn't the cheap
default — aim for roughly even counts per class; (2) more data overall, since 200
examples split three ways is thin for learning a subjective boundary from scratch;
(3) consider class weighting in the loss so the minority classes (`hot_take`,
`reaction`) actually cost the model something when missed.

## Spec reflection

- **One way the spec helped:** `[FILL IN — e.g. forcing the edge-case decision
  rule up front in Milestone 1 meant my annotation was consistent on the
  stat-garnished hot take, instead of me deciding ad hoc each time.]`
- **One way my implementation diverged and why:** `[FILL IN — e.g. the spec
  suggested 2 examples per label in the README table; I kept one here and put the
  second in planning.md to avoid duplication.]`

## AI usage

*(At least 2 specific instances — what you directed the tool to do, what it
produced, what you changed/overrode. Disclose any annotation help.)*

1. **Label stress-testing.** I gave Claude my definitions and asked it to
   generate boundary cases. It produced stat-garnished hot takes I couldn't
   cleanly classify, which led me to add the "is the evidence load-bearing?"
   decision rule. `[Adjust to what actually happened.]`
2. **Results interpretation (verified against the artifact).** I used an AI tool
   to summarize my notebook output. Its per-class metrics table was internally
   inconsistent with its own confusion matrix, so I recomputed every per-class
   number directly from the confusion matrix (`confusion_matrix.png`) rather than
   trusting the summary. The verified finding: the model collapsed to predicting
   `analysis` (30/32) and never predicted `hot_take`. This is exactly the
   "verify the patterns yourself" check the spec calls for — the summary text and
   the artifact disagreed, and the artifact won.

## How to run

1. `collect_posts.ps1` (or `.py`) → pulls comments into `takemeter_raw.csv`.
2. Label the `label` column by hand.
3. Open the TakeMeter Colab notebook, set `LABEL_MAP` to
   `{"analysis":0,"hot_take":1,"reaction":2}`, upload the labeled CSV.
4. Run Sections 1–2, then Section 5 (baseline), then 3–4 (fine-tune + eval),
   then 6 (compare + export).
5. Commit `evaluation_results.json` and `confusion_matrix.png` to this repo.

## Repo contents

- `planning.md` — design doc (written before collection)
- `takemeter_raw.csv` / `takemeter_labeled.csv` — dataset
- `collect_posts.ps1`, `collect_posts.py` — collection helpers
- `README.md` — this file
- `evaluation_results.json`, `confusion_matrix.png` — notebook outputs
