# Multilingual Support Ticket Triage Pipeline

A Claude-powered pipeline that classifies multilingual customer support tickets (queue routing + priority) and drafts reply responses, with accuracy evaluated against ground-truth labels from a public support ticket dataset.

## Overview

Customer support and IT operations teams that serve global markets often face a routing bottleneck: incoming tickets need to be read, classified into the right department queue, assigned a priority, and answered — across multiple languages. This project builds and evaluates a Claude-based classification and response-drafting pipeline for that workflow, using a public multilingual support ticket dataset as a proxy for real-world traffic.

The goal was not just to build a working pipeline, but to rigorously evaluate where it performs well, where it breaks down, and why — the kind of diagnostic work that matters when deciding whether (and how) to deploy an LLM-based automation in production.

## Dataset

- **Source:** [Multilingual Customer Support Tickets](https://www.kaggle.com/datasets/tobiasbueck/multilingual-customer-support-tickets) (public Kaggle dataset, `dataset-tickets-multi-lang3-4k.csv` version)
- **Size:** 4,000 raw tickets → 3,983 after cleaning
- **Languages:** English, German, Spanish, French, Portuguese
- **Fields used:** subject, body, ground-truth `queue` (10 categories), ground-truth `priority` (low/medium/high), human agent `answer`, `business_type`, `tags`

## Pipeline

1. **Data cleaning** — dropped empty-body rows, backfilled missing subjects from ticket body text, removed noisy/duplicate `business_type` labels, consolidated 9 sparse tag columns into a single list field.
2. **Stratified sampling** — drew a 250-ticket test set stratified by `language × queue` so the evaluation sample preserves the real (imbalanced) distribution of both dimensions.
3. **Claude classification + reply drafting** — for each ticket, Claude (`claude-sonnet-4-6`) was prompted to return a structured JSON response with:
   - `queue`: predicted routing department, constrained to the dataset's 10 fixed labels
   - `priority`: predicted urgency (low/medium/high)
   - `reply_draft`: a short reply written in the ticket's own language
4. **Evaluation** — predictions were compared against ground-truth labels to compute accuracy overall, by language, and by queue category, plus a confusion analysis of misclassification patterns.

The pipeline includes retry logic for malformed JSON / rate limits, and incremental checkpointing so an interrupted run can resume without reprocessing (or re-billing) completed tickets.

## Results

| Metric | Accuracy |
|---|---|
| Overall queue classification | 47.2% |
| Overall priority classification | 58.8% |

**By language** (queue accuracy): differences across languages were modest (40–58%), indicating that multilingual understanding was *not* the primary bottleneck — Claude handled ticket content roughly as well in German, Spanish, French, and Portuguese as in English.

**By queue category**, accuracy varied enormously:

| Queue | Accuracy |
|---|---|
| Returns and Exchanges | 100% |
| Billing and Payments | 85.7% |
| Service Outages and Maintenance | 77.8% |
| Sales and Pre-Sales | 66.7% |
| Technical Support | 61.2% |
| IT Support | 48.1% |
| Product Support | 29.5% |
| Customer Service | 2.6% |
| General Inquiry | 0% |
| Human Resources | 0% |

### Root-cause analysis

Categories with narrow, concrete definitions (Returns and Exchanges, Billing and Payments) were classified almost perfectly. Categories with overlapping or "catch-all" definitions performed poorly. A confusion-matrix analysis showed the dominant error pattern was mutual confusion between **Technical Support**, **IT Support**, and **Product Support** — three labels whose names describe closely related concepts with no explicit boundary. In addition, **Customer Service**, the dataset's general-purpose "catch-all" queue, was almost never predicted correctly: tickets that a human labeler filed under this default category were instead routed by Claude to whichever *specific* queue best matched the ticket's surface content, since the model has no equivalent of a human labeler's "none of the above" instinct.

This points to a data/label-design issue rather than a model-capability or language-understanding limitation: **the accuracy ceiling here is bounded by how clearly the queue taxonomy is defined**, not by Claude's classification ability. A follow-up experiment adding explicit per-label definitions to the system prompt was tested on a 30-ticket subset; results were inconclusive at that sample size, and would need a larger controlled test (ideally the full 250-ticket set, evaluated with an unambiguous, versioned prompt) to properly validate.

## Limitations

- Ground truth comes from a public/synthetic dataset, not real production tickets — label noise in the source data (queue/priority assigned by different annotators) puts a ceiling on achievable accuracy that isn't purely a model-quality issue.
- Evaluation used a 250-ticket stratified sample rather than the full 3,983-ticket cleaned dataset, to control API cost during iteration.
- Priority accuracy (58.8%) should be read with the same caveat — urgency labeling is inherently subjective, and inter-annotator agreement on priority is typically far from 100% even among humans.

## Tech Stack

- **Claude API** (`claude-sonnet-4-6`) — structured classification and reply generation
- **Python / pandas** — data cleaning, stratified sampling, metrics computation
- **Jupyter Notebook** — end-to-end pipeline (`ticket_triage_pipeline.ipynb`)
- **matplotlib** — accuracy visualization

## Repository Structure

```
.
├── ticket_triage_pipeline.ipynb   # Full pipeline: cleaning → sampling → Claude API → metrics
├── dataset-tickets-multi-lang3-4k.csv   # Raw source data
├── cleaned_tickets_full.csv       # Cleaned dataset (3,983 rows)
├── sampled_tickets_test.csv       # Stratified 250-ticket test sample
├── results_partial.csv            # Claude predictions vs. ground truth per ticket
├── metrics_summary.csv            # Aggregate accuracy metrics
└── README.md
```

## Planned Extension: Workflow Routing Automation

To extend this from a classification pipeline into a full triage *workflow*, the next step is a trigger-based automation that acts on Claude's output — for example, automatically emailing the relevant team when a ticket is classified as `priority: high`. This maps directly onto the trigger → condition → action model used by tools like Microsoft Power Automate / Copilot Studio and Zapier. *(Implementation status: designed, not yet built — see [Zapier flow build] for the in-progress version.)*

## Key Takeaway

A raw accuracy number without diagnosis is not very informative. The more useful output of this project is the root-cause analysis: the classification bottleneck traces back to ambiguous label definitions in the queue taxonomy (specifically the Technical Support / IT Support / Product Support overlap, and the Customer Service catch-all problem) rather than to Claude's language understanding, which held up consistently across five languages.

## screenshot
<img width="1169" height="612" alt="image" src="https://github.com/user-attachments/assets/4e959773-48e6-4b15-9650-ccaf34823a6d" />
<img width="957" height="329" alt="image" src="https://github.com/user-attachments/assets/707e0f76-d478-49da-93e1-314653e7f0ff" />
<img width="1037" height="506" alt="image" src="https://github.com/user-attachments/assets/957b3747-0975-4edc-be10-f80def518e6d" />


