# TakeMeter — Planning Document

**Project:** AI201 Project 3 — TakeMeter
**Community:** r/nba
**Taxonomy:** `analysis` · `hot_take` · `reaction`

---

## 1. Community

I chose **r/nba**, the main NBA discussion subreddit. It's a strong fit for a
discourse-quality classifier for three reasons:

- **High volume, high variance.** On any given thread the comments range from
  detailed tactical breakdowns to one-word reactions. That spread is exactly
  what a classifier needs — if every comment looked the same, there'd be nothing
  to learn.
- **Timely.** I'm collecting right after the Knicks' 2026 championship run, when
  the subreddit is unusually active. Title aftermath produces the full range:
  sober series analysis, overconfident dynasty proclamations, and pure
  celebration. That makes all three of my labels well-represented.
- **The distinction is real to the community.** r/nba users themselves
  constantly police the difference between a "take" and "actual analysis."
  "Hot take vs. analysis" isn't an artificial category I'm imposing — it's how
  people there already talk.

## 2. Labels

> Note: the example comments below are representative of the kind of post each
> label covers. I will replace/confirm them with **real comments pulled from my
> collected CSV** before finalizing, so every example is one I actually labeled.

### `analysis`
A comment that makes a structured argument supported by specific, verifiable
evidence — statistics, tactical observation, lineup/matchup detail, or
historical comparison. The reasoning would still stand if you removed the
opinion framing.

- *"Brunson's halfcourt scoring carried them, but the real swing was their drop
  coverage against the Spurs' bigs — they held SA to 41% in the paint across the
  last three games."*
- *"People forget the Knicks went 3-0 in clutch games this series. Their late-game
  half-court offense ranked top-5 in the playoffs by points per possession."*

### `hot_take`
A bold, confident opinion asserted **without** supporting evidence — or with only
decorative evidence (a single cherry-picked stat used for effect, not as part of
an argument). The claim might be correct, but the comment asserts rather than
argues.

- *"This Knicks team is already a dynasty, mark my words. Three-peat incoming."*
- *"Brunson is a top-3 player in the league now and it's not even close."*

### `reaction`
An immediate emotional response to a specific moment or result, with little or no
argument. The comment expresses a feeling, not a claim.

- *"LETS GOOOO KNICKS IN 5 I'M NOT EVEN CRYING YOU'RE CRYING"*
- *"25 years. I waited 25 years for this. Goodnight everyone."*

## 3. Hard edge cases

**The stat-garnished hot take.** The genuinely ambiguous case is a confident
opinion that drops in *one* stat to sound credible — e.g.
*"Brunson is a lock for Finals MVP, dude averaged 30 a game."* It cites a number,
which looks like `analysis`, but the number is decorative, not load-bearing.

**Decision rule:** If removing the opinion framing leaves a real argument backed
by specific, verifiable evidence → `analysis`. If the stat is a single number
selected for effect and the comment is fundamentally an assertion → `hot_take`.
The test is: *is the evidence doing argumentative work, or is it decoration?*

**Second edge: the reaction with a clause of opinion.** e.g.
*"WE WON omg this team is unstoppable next year."* The dominant register is
emotional celebration with a throwaway claim tacked on.

**Decision rule:** If the comment's center of gravity is the emotional moment and
any claim is incidental → `reaction`. If the claim is the point and the emotion
is just delivery → `hot_take`.

I'll log every borderline case in the `notes` column of my CSV as I annotate, and
the three I found hardest go in section 9 below and in the README.

## 4. Data collection plan

- **Source:** Top-level comments from r/nba's recent `top` posts, pulled via
  Reddit's public JSON endpoints with a collection script (permitted by the
  assignment). Public content only.
- **Volume target:** ~250 collected so I can discard low-quality rows and still
  land comfortably above 200 labeled.
- **Filters at collection:** drop comments under 60 chars (no real take) and over
  600 chars (copypastas), skip AutoModerator and deleted comments.
- **Per-label target:** aim for at least 20% per label. `reaction` is likely the
  easiest to over-collect post-championship, so if it exceeds ~50% I'll
  specifically pull from analysis-heavy threads (post-game film threads, the
  daily discussion) to top up `analysis` and `hot_take`.
- **Underrepresentation plan:** if any label is under 20% after the first 200,
  I'll collect targeted batches from threads biased toward that label rather than
  re-sampling randomly.

## 5. Evaluation metrics

Accuracy alone is misleading here because the classes are likely imbalanced
(celebration reactions will dominate raw volume). A model that just predicts
`reaction` could look "good" on accuracy while being useless.

I'll use:
- **Macro-averaged F1** as the headline metric — it weights all three classes
  equally, so the model can't coast by nailing the majority class.
- **Per-class precision, recall, and F1** — because the classes mean different
  things. I care most about `analysis` precision: if the eventual use is
  "surface the good takes," a prediction of `analysis` needs to be trustworthy.
- **Confusion matrix** — to see the *direction* of errors (e.g., is it calling
  `analysis` → `hot_take`, the boundary I expect to be hardest?).

## 6. Definition of success

"Good enough to be useful in a real community tool" means:
- Fine-tuned **macro-F1 ≥ 0.70**, with **no single class below 0.60 F1**.
- Fine-tuned model **beats the zero-shot Groq baseline on macro-F1** by a
  meaningful margin (not within noise) — otherwise fine-tuning didn't earn its
  keep.
- `analysis` **precision ≥ 0.75** specifically, since a "highlight the good
  takes" feature is only useful if its `analysis` calls are mostly right.

If I hit macro-F1 ≥ 0.70 but `analysis` precision is low, I'd call it
"promising but not deployable" and say so honestly.

---

## AI Tool Plan

**Label stress-testing.** Before annotating, I gave Claude my three definitions
and the edge-case rules and asked it to generate boundary-straddling comments
(stat-garnished hot takes, reactions with a tacked-on claim). Where I couldn't
classify its output cleanly, I tightened the definitions — that's what produced
the "is the evidence load-bearing or decorative?" test in section 3.

**Annotation assistance.** I will optionally use an LLM to **pre-label** a batch
of collected comments using my definitions, then review and correct **every**
row myself — pre-labeling speeds review but I'm not trusting it blind. Any
pre-labeled rows are flagged in the `notes` column and disclosed in the README's
AI usage section.

**Failure analysis.** After evaluation, I'll paste my list of wrong predictions
to an LLM and ask it to spot systematic patterns (a recurring confused label
pair, sarcasm, short comments) — then verify each pattern by re-reading the
examples myself before writing it up.

---

## Groq baseline prompt (for Section 5 of the notebook)

```
You are classifying comments from r/nba, the NBA discussion subreddit.
Assign each comment to exactly one of these three categories.

analysis: A structured argument backed by specific, verifiable evidence —
statistics, tactical detail, matchup observation, or historical comparison.
The reasoning stands on its own even without the opinion.
Example: "The real swing was their drop coverage — they held the opponent to 41% in the paint over the last three games."

hot_take: A bold, confident opinion stated without real supporting evidence,
or with only a single decorative stat used for effect. It asserts rather than argues.
Example: "This team is already a dynasty, three-peat incoming."

reaction: An immediate emotional response to a moment or result, with little or
no argument. It expresses a feeling, not a claim.
Example: "LETS GOOO I waited 25 years for this."

Respond with ONLY the label name: analysis, hot_take, or reaction.
Do not explain your reasoning.

Valid labels:
analysis
hot_take
reaction
```

---

## 9. Hardest annotation decisions (fill in during labeling)

*(Log the 3 hardest real comments here as you annotate — text, the labels it sat
between, and your final call. These carry over into the README.)*

1. _TODO after labeling_
2. _TODO after labeling_
3. _TODO after labeling_
