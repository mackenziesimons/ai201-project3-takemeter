# ai201-project3-takemeter

Repository for all Project 3 assets that live outside the notebook.

## Contents

- `planning.md`: Project plan and milestones.
- `data/labeled_dataset.csv`: Labeled dataset used by the notebook workflow.
- `outputs/evaluation_results.json`: Evaluation output exported from Colab.
- `outputs/confusion_matrix.png`: Confusion matrix image exported from Colab.

## Workflow

1. Keep planning notes and dataset updates in this repository.
2. Run training/evaluation in Colab.
3. Download output artifacts from Colab and replace placeholder files in `outputs/`.
4. Commit and push updates.

## Notes

- Placeholder output files are included so folder structure is tracked from the start.
- Replace placeholders with real outputs before final submission.

## Project Context

This project classifies Hacker News comments into four discourse-quality labels: `analysis_evidence`, `practical_experience`, `opinion_speculation`, and `low_substance_reaction`. The point is not topic classification; it is separating comments that add reasoning or firsthand context from comments that are mostly stance, speculation, or terse reaction.

The dataset lives in a single CSV file at `data/labeled_dataset.csv` and contains 220 public comments collected from Hacker News. The current file also includes a `notes` column so difficult cases stay visible during review.

## Failure Analysis

I reviewed three misclassified comments to understand where the label boundaries are still too soft.

1. Heat / insulation comment
- True: `analysis_evidence`
- Predicted: `opinion_speculation`
- Why it failed: the comment mixes a broad stance with a real argument about aircon, insulation, and cost. The model likely overweighted the opinionated tone and missed the concrete causal reasoning.
- What this suggests: the boundary between `analysis_evidence` and `opinion_speculation` needs to privilege explicit mechanism and supporting detail, even when the writing is argumentative.

2. Extreme-heat mortality comment
- True: `analysis_evidence`
- Predicted: `opinion_speculation`
- Why it failed: the text is short, but it still makes a factual claim plus a concrete example about temperature-related deaths. The model seems to struggle when evidence is compressed into a sharp, rhetorical sentence.
- What this suggests: short comments that contain a factual claim and a real-world example should not be treated as `low_substance_reaction` or generic opinion just because they are terse.

3. Portal game comment
- True: `low_substance_reaction`
- Predicted: `opinion_speculation`
- Why it failed: the comment is a simple evaluative statement, but it lacks the reasoning that would make it substantive. The model overgeneralized from the adjective-like wording and treated it as a full opinion.
- What this suggests: `opinion_speculation` should require some explicit stance, prediction, or rationale beyond a bare quality judgment.

Overall pattern: the model is confusing short, rhetorically phrased comments with substantive analysis. The main fix is to sharpen the rule that `analysis_evidence` requires a concrete reason or example, while `opinion_speculation` should be reserved for unsupported stance or prediction, and `low_substance_reaction` should cover terse value judgments without added reasoning.

## Evaluation Report

I used an AI pass first to surface error patterns, then re-read the examples myself to verify which patterns were real. The strongest pattern was a directional confusion from `analysis_evidence` into `opinion_speculation`; the weaker broad claim that the model simply fails on all short posts was too broad and got corrected after rereading the portal example, which is better described as a bare value judgment than as a general short-post failure.

### Overall Accuracy

| Model | Accuracy |
| --- | ---: |
| Baseline | 0.5758 |
| Fine-tuned | 0.6061 |

### Sample Classifications

The three exact row-level confidences below were preserved in the evaluation prompt. The final row is a representative correct prediction from the model’s all-`opinion_speculation` output, but the original export did not preserve its per-example confidence score.

| Example | True label | Predicted label | Confidence | Why the prediction makes sense |
| --- | --- | --- | ---: | --- |
| Something needs to be done with the heat... | `analysis_evidence` | `opinion_speculation` | 0.42 | The model likely focused on the argumentative tone and missed the concrete cost/mechanism reasoning about aircon and insulation. |
| Thousands will die, but one guy went jogging at 40 degrees outside so it’s okay. | `analysis_evidence` | `opinion_speculation` | 0.41 | This is a short rhetorical claim with a factual claim embedded in it, and the model appears to have flattened it into a generic opinion. |
| Portal was pretty good and an originalish product | `low_substance_reaction` | `opinion_speculation` | 0.38 | The model overread a bare evaluative phrase as a full opinion, even though there is no reasoning or prediction. |
| I seem to recall the Economist wholeheartedly supporting the Iraq War. Am I wrong? | `opinion_speculation` | `opinion_speculation` | not exported | This is a reasonable prediction because the comment is a tentative stance/question rather than a reasoned argument or firsthand report. |

### Per-Class Metrics

The exported baseline artifact only preserved aggregate accuracy, not a class-wise report, so I cannot reconstruct its per-class metrics without inventing numbers. The fine-tuned model’s class-wise metrics can be read directly from the test-set confusion matrix below.

| Model | Label | Precision | Recall | F1 | Support |
| --- | --- | ---: | ---: | ---: | ---: |
| Fine-tuned | opinion_speculation | 0.6061 | 1.0000 | 0.7547 | 20 |
| Fine-tuned | analysis_evidence | 0.0000 | 0.0000 | 0.0000 | 8 |
| Fine-tuned | low_substance_reaction | 0.0000 | 0.0000 | 0.0000 | 4 |
| Fine-tuned | practical_experience | 0.0000 | 0.0000 | 0.0000 | 1 |

Macro averages for the fine-tuned model: precision 0.1515, recall 0.2500, F1 0.1887.

### Fine-Tuned Confusion Matrix

The fine-tuned model collapsed to a single predicted label on the test set.

| True \ Predicted | opinion_speculation | analysis_evidence | low_substance_reaction | practical_experience |
| --- | ---: | ---: | ---: | ---: |
| opinion_speculation | 20 | 0 | 0 | 0 |
| analysis_evidence | 8 | 0 | 0 | 0 |
| low_substance_reaction | 4 | 0 | 0 | 0 |
| practical_experience | 1 | 0 | 0 | 0 |

### What Failed and Why

1. Heat / insulation comment
- True: `analysis_evidence`
- Predicted: `opinion_speculation`
- Failure mode: the model treated an argument with mechanism and cost reasoning as a general stance because the comment is rhetorically framed.
- What I verified: the factual content is real and explicit, so this is a labeling/model boundary problem, not a mislabeled example.

2. Extreme-heat mortality comment
- True: `analysis_evidence`
- Predicted: `opinion_speculation`
- Failure mode: a short sentence with a factual claim and a concrete example was compressed into the opinion bucket.
- What I verified: the sarcasm/hyperbole is present, but the comment still carries a substantive claim, so the model needs to learn that rhetorical tone does not automatically mean low substance.

3. Portal game comment
- True: `low_substance_reaction`
- Predicted: `opinion_speculation`
- Failure mode: the model overread a simple evaluative phrase as a full opinion.
- What I verified: this is a genuine borderline case, but the better label is `low_substance_reaction` because there is no reasoning or prediction.

### Interpretation

The model is not learning the `analysis_evidence` boundary. It is defaulting to `opinion_speculation` whenever a post is short, sarcastic, or strongly worded, even when the post contains explicit reasoning or an example. The next fix should be to add more boundary examples for `analysis_evidence` versus `opinion_speculation`, especially short argumentative comments and sarcastic factual rebuttals.

## Spec Reflection

The spec helped by forcing a clear community choice, a single labeled CSV, and explicit evaluation criteria before implementation started. That structure kept the project from turning into an open-ended data collection exercise and made the label definitions and success thresholds concrete enough to check later.

My implementation diverged in two ways. First, I chose Hacker News comments instead of one of the example communities because the public Algolia API made it easier to collect 200+ examples quickly and legally. Second, I added an `annotation_audit.csv` file and AI-assisted pre-labeling workflow so I could track how much of the labeling process was helped by the model and which items I reviewed manually.

## AI Usage

I used AI in three specific places and reviewed its output myself each time.

1. Label stress-testing: I asked GPT-5.3-Codex to generate boundary examples for adjacent labels before finishing the schema. It produced examples that showed the `analysis_evidence` vs. `opinion_speculation` boundary was too soft, and I tightened the rules so explicit mechanism and supporting detail count more than tone.

2. Annotation assistance: I used GPT-5.3-Codex to pre-label batches of comments, then manually checked each one before writing the final label into `data/labeled_dataset.csv`. I also logged the AI suggestion versus the final human label in `data/annotation_audit.csv` so the workflow is auditable.

3. Failure analysis: I asked an AI tool to inspect wrong predictions and identify error patterns. It correctly pointed to an `analysis_evidence` -> `opinion_speculation` confusion pattern, but I discarded its broader claim that the model simply fails on all short posts after rereading the examples myself.
