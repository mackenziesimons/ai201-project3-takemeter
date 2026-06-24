# Project 3 Planning

## 1) Community
I chose Hacker News comments as the target community. Hacker News is a strong fit for text classification because it has high daily volume, public access to comments, and clear variation in discourse quality, from detailed technical reasoning to short reactions and unsupported takes. This variety makes the task interesting because the classifier is not just separating topics, it is separating comment quality and contribution type in a way regular users and moderators care about.

## 2) Labels
I will use four labels.

### Label A: analysis_evidence
Definition: A comment belongs to analysis_evidence when it makes a claim and supports it with explicit reasoning, mechanism, context, or evidence.
Example 1: "The issue was not just demand growth; supply bottlenecks in transformers and permitting delays are the binding constraints, so price spikes are expected even if consumption is flat."
Example 2: "This benchmark is misleading because it averages first-token latency with throughput; those metrics move in opposite directions as batch size increases."

### Label B: practical_experience
Definition: A comment belongs to practical_experience when it contributes first-hand implementation details, workflow lessons, or operational constraints from direct experience.
Example 1: "We reduced false positives by routing vendor reports through a triage script and only paging the on-call engineer for items above a severity threshold."
Example 2: "On my local setup, a 600W wall draw became the limiting factor, so undervolting the GPU improved stability more than changing the model size."

### Label C: opinion_speculation
Definition: A comment belongs to opinion_speculation when it states a belief, preference, or prediction without enough supporting reasoning to evaluate the claim.
Example 1: "This policy will probably make the problem worse in the long run."
Example 2: "I think this startup is going to dominate the space next year."

### Label D: low_substance_reaction
Definition: A comment belongs to low_substance_reaction when it is primarily a brief reaction, rhetorical jab, or fragment with minimal informational value.
Example 1: "No CLI though."
Example 2: "This is nonsense."

## 3) Hard Edge Cases
The hardest edge cases are mixed comments that include one useful factual sentence but are mostly reactionary tone, for example a sarcastic opener followed by a small correction. To handle this, I will assign labels by primary communicative intent and use this tie-break order when a comment could fit multiple labels: analysis_evidence > practical_experience > opinion_speculation > low_substance_reaction. I will also keep an "ambiguous review" list during annotation; after every 25 annotations, I will revisit those examples and refine boundary notes to improve consistency.

Boundary clarifications used during annotation:
- analysis_evidence vs opinion_speculation: if the comment includes at least one explicit because/therefore-style rationale or concrete supporting fact, use analysis_evidence; otherwise use opinion_speculation.
- practical_experience vs analysis_evidence: if the main value is first-hand workflow or operational detail from direct experience, use practical_experience even if light reasoning is present.
- opinion_speculation vs low_substance_reaction: if it forms a complete stance or prediction, use opinion_speculation; if it is mainly a fragment, jab, or rhetorical line with little standalone meaning, use low_substance_reaction.

Specific difficult cases from annotation:
- Case 1: "Would that be all that's required to make it okay?"
	- Could be: opinion_speculation or low_substance_reaction.
	- Decision: opinion_speculation, because it is a complete normative stance prompt rather than a pure fragment.
- Case 2: "No, he is excluded. So is Alexandr."
	- Could be: low_substance_reaction or analysis_evidence.
	- Decision: low_substance_reaction, because it provides minimal context and no explicit rationale.
- Case 3: "I tried to raise that with my internal security team recently ... malware needs to be dealt even if it's a dev dependency."
	- Could be: practical_experience or analysis_evidence.
	- Decision: practical_experience, because the core contribution is first-hand process experience.
- Case 4: "I fear it'll just move the problem one layer up ... how do you ensure the specification is watertight?"
	- Could be: opinion_speculation or analysis_evidence.
	- Decision: analysis_evidence, because it gives a concrete failure mechanism and argument structure.

## 4) Data Collection Plan
I will collect public comments from Hacker News via the Algolia API endpoint for recent comments. The dataset target is 220 comments total so that at least 200 remain after cleaning and removing malformed rows.

Target distribution by label at annotation time:
- analysis_evidence: 55 comments
- practical_experience: 55 comments
- opinion_speculation: 55 comments
- low_substance_reaction: 55 comments

If one label is underrepresented after the first 200 comments, I will do focused collection by querying additional time windows and stories likely to contain that style, then continue annotation until each label has at least 40 examples. During training, I will also use class-weighted loss to reduce bias from any remaining imbalance.

Collection outcome (current file):
- Total labeled examples: 220 (single CSV file)
- Label counts:
	- opinion_speculation: 135
	- analysis_evidence: 52
	- low_substance_reaction: 26
	- practical_experience: 7
- Largest class share: 61.36%, so no label exceeds the 70% imbalance threshold.

## 5) Evaluation Metrics
Accuracy alone is not enough because class imbalance can hide poor performance on rare but important labels. I will report:
- Macro F1: treats each label equally and shows whether minority labels are being learned.
- Per-label Precision and Recall: reveals whether the model is over-predicting or missing specific classes.
- Confusion Matrix: shows where boundary mistakes happen, especially between opinion_speculation and low_substance_reaction.
- Weighted F1: gives an overall quality score that still reflects label frequency.

These metrics match this task because the goal is reliable label discrimination across nuanced discourse types, not just maximizing a single aggregate number.

## 6) Definition of Success
For this classifier to be genuinely useful in a real community tool, it must separate high-value comments from low-value comments with stable behavior across labels, not just perform well on the majority class.

Objective evaluation protocol:
- Data split: 70/15/15 train/validation/test with stratification by label.
- Model selection: choose the checkpoint with best validation macro F1.
- Final decision: compute metrics once on the held-out test set and compare against thresholds below.

Deployment-ready threshold (pass/fail):
- Test macro F1 >= 0.72
- Test weighted F1 >= 0.78
- Per-label recall >= 0.60 for all four labels
- No single confusion pair accounts for more than 35% of all errors

Good-enough threshold for assisted triage (not full automation):
- Test macro F1 >= 0.65
- Test weighted F1 >= 0.72
- Per-label recall >= 0.50 for all four labels
- Manual review required for predictions with confidence below 0.60

If any deployment-ready criterion fails, the model is not considered production-ready and will only be used as a human-in-the-loop sorting aid.

## 7) AI Tool Plan

### A) Label Stress-Testing (before full annotation)
- Tool: GPT-5.3-Codex in Copilot Chat.
- Procedure: provide label definitions plus edge-case rules and ask for 10 synthetic boundary comments (for each adjacent pair: analysis_evidence vs practical_experience, practical_experience vs opinion_speculation, opinion_speculation vs low_substance_reaction).
- Decision rule: if I cannot confidently assign at least 8 of 10 without changing rules, I will tighten definitions and rerun one more stress-test round before annotating 200 examples.
- Output to store: a short stress-test table in planning notes with comment text, intended boundary pair, chosen label, and rationale.

### B) Annotation Assistance
- Decision: Yes, use LLM pre-labeling for speed, but always human-verified before final label lock.
- Tool: GPT-5.3-Codex via Copilot Chat on batches of 25 comments.
- Workflow:
	- Step 1: AI proposes labels and 1-line rationale per comment.
	- Step 2: I review and either accept or override.
	- Step 3: only reviewed labels are written as final labels.
- Disclosure tracking:
	- In data/labeled_dataset.csv, keep final agreed label in label.
	- In a sidecar file annotation_audit.csv, record id, ai_suggested_label, human_final_label, changed_flag, and review_notes.
	- This gives explicit traceability for the AI usage section.

### C) Failure Analysis
- Procedure: after evaluation, export all wrong predictions from the test set and provide them to GPT-5.3-Codex for pattern mining.
- What I will ask the AI to find:
	- recurring confusion themes between specific label pairs,
	- lexical or structural cues correlated with errors,
	- potential guideline gaps causing systematic mislabeling.
- Verification plan:
	- manually inspect at least 20 sampled errors,
	- confirm each claimed pattern has multiple real examples,
	- reject patterns that do not survive manual inspection,
	- convert confirmed patterns into concrete guideline updates.

Explicit non-use decision:
- I will not use AI to auto-approve final metrics interpretation without manual cross-check against confusion matrix and per-label scores.

## Implementation Notes
- Annotation file: data/labeled_dataset.csv
- Planned outputs from Colab: outputs/evaluation_results.json and outputs/confusion_matrix.png
