# Project 3 Plan: TakeMeter

## Objective
Build and evaluate a model for classifying text using a labeled dataset prepared outside the notebook.

## Scope (Outside Notebook)
- Project planning notes in this file
- Labeled dataset CSV for training/evaluation input
- Repository documentation in README
- Colab-generated output artifacts:
  - `outputs/evaluation_results.json`
  - `outputs/confusion_matrix.png`

## Milestones
1. Define labeling schema and collect examples
2. Create and validate labeled CSV format
3. Train and evaluate model in Colab notebook
4. Export evaluation artifacts from Colab
5. Commit artifacts and summarize outcomes in README

## Dataset Notes
- Keep text and label columns consistent over time
- Track class balance before training
- Avoid leaking test examples into training split

## Artifact Checklist
- [ ] `data/labeled_dataset.csv` populated with full dataset
- [ ] `outputs/evaluation_results.json` exported from Colab
- [ ] `outputs/confusion_matrix.png` exported from Colab
- [ ] README updated with final results summary
