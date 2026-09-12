# SpotifyCares AI Support Agent

An AI support agent for SpotifyCares built on real customer-support conversations from Twitter. The system classifies an incoming customer message by intent, decides whether it can be auto-handled or needs escalation to a human, and drafts a reply grounded in how Spotify has historically resolved similar issues.

Built as a take-home assignment for Hiver.

## Headline result

| Method | Intent Accuracy | Flag Accuracy |
|---|---|---|
| Trivial (most common class) | 40.0% | 45.0% |
| Keyword matching | 67.5% | 42.5% |
| LLM classifier (this system) | **75.0%** | **75.0%** |

Measured on 40 examples from a 200-example hand-labeled golden set.

Read [report.md](report.md) for the full analysis, including what is misleading about that 75 percent figure. Read [decision_log.md](decision_log.md) for the non-obvious choices behind the build.

## What the system does

Given a new customer message, the pipeline runs four steps:

1. **Classify intent.** One of five categories derived from reading the data: Account_Billing, Technical_Troubleshooting, Feature_Request, General_Question, Closure.
2. **Classify resolution flag.** One of four: needs_dm, needs_more_info, answerable_now, already_closed. This is a separate axis from intent because what a customer wants and what Spotify does next vary independently.
3. **Retrieve a grounding example.** Finds the most semantically similar past case from a filtered pool of 75 genuinely informative historical replies, then runs an explicit LLM relevance check before using it.
4. **Decide and draft.** Escalates if the message needs private account verification. Otherwise drafts a reply grounded in the retrieved precedent, or refuses to draft if no relevant precedent exists.

## Reproducing the headline result

Total time: under 15 minutes, most of which is the dataset download.

### Prerequisites

- A Kaggle account and API token (kaggle.com, Settings, API, Create New Token)
- A Groq API key (console.groq.com, free tier is sufficient)
- Google Colab, or any Jupyter environment with internet access

### Steps

**1. Open the notebook.** Upload `spotifycares_pipeline.ipynb` to Google Colab, or run it locally in Jupyter.

**2. Set your Kaggle credentials.** Run the first cell and enter your Kaggle username and API key when prompted. This writes the credentials file the dataset download needs.

**3. Download the dataset.** The notebook fetches it automatically:

```python
import kagglehub
path = kagglehub.dataset_download("thoughtvector/customer-support-on-twitter")
```

The raw dataset is roughly 3 million tweets and is not included in this repo. It downloads in about one minute.

**4. Set your Groq API key.** Replace the placeholder in the classifier cell with your own key.

**5. Upload the golden set.** Upload `golden_set_labeling.xlsx` to the Colab session when the notebook prompts for it. This is the hand-labeled evaluation data everything is measured against.

**6. Run all cells.** The notebook filters the dataset to SpotifyCares, builds customer-reply pairs, runs the classifier against the golden set, runs both baselines, and prints the comparison table above.

Evaluation runs on a 40-example subsample by default. Adjust the row range in the evaluation cell to run against more of the golden set. Note that the free Groq tier has request-rate limits, so larger runs need a longer sleep interval between calls.

## Files

| File | Contents |
|---|---|
| `spotifycares_pipeline.ipynb` | Full pipeline: data loading, filtering, classifier, retrieval, escalation logic, evaluation |
| `golden_set_labeling.xlsx` | 200 hand-labeled examples with intent and resolution_flag, plus labeling criteria on the Instructions tab |
| `report.md` | Problem framing, results against baselines, failure analysis, headline-number caveats, next steps |
| `decision_log.md` | Twelve non-obvious decisions and the reasoning behind each |

## Dataset

Primary source: [Customer Support on Twitter](https://www.kaggle.com/datasets/thoughtvector/customer-support-on-twitter) (Kaggle, thoughtvector). Filtered to SpotifyCares, producing 43,206 customer-message and brand-reply pairs. The golden set samples 200 of these.

## Key limitation worth knowing upfront

Roughly 40 percent of SpotifyCares' public replies redirect the customer to a private DM, where the actual resolution happens. That resolution content is not in this dataset. The system is therefore designed to recognize and correctly escalate those cases rather than to fabricate resolutions it has no basis for. This shapes both the taxonomy and how the results should be read. See the report for the full discussion.

## Tools used

- `pandas` for data filtering and pair construction
- `sentence-transformers` (all-MiniLM-L6-v2) for embeddings
- `openai/gpt-oss-120b` via the Groq API for classification, relevance checking, and reply drafting
- `openpyxl` for the labeled golden set

AI coding assistants were used during development, as permitted by the assignment brief. The design decisions, taxonomy, data findings, and failure analysis are my own and are documented in `decision_log.md`.
