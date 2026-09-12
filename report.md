# SpotifyCares AI Support Agent: Report

## 1. Problem framing

**What "good" means for this brand.** SpotifyCares' Twitter support handles a high volume of repeatable issues. Reading a random sample of the data showed the support workflow splits cleanly into two behaviors. The first covers issues that need private account access, such as login problems, billing disputes, and hacked accounts. These get resolved by redirecting the customer to DM, so no real resolution content appears in the public dataset. The second covers issues answerable with public information, such as feature questions, known product limitations, and bugs that need a diagnostic follow-up. These do get real, gradeable responses.

Good performance for this system means correctly telling those two situations apart. Account-specific issues should be escalated for safety, since the system has no authority to access private account data and should never pretend otherwise. Public-answerable issues should get a real, grounded response rather than a generic template.

**What I chose not to build.**

Full multi-turn conversation modeling. The public dataset only exposes single-exchange pairs, and roughly 40 percent of these are DM redirects where the actual resolution is invisible. Building conversational memory on top of this would create false confidence in data the system fundamentally cannot see.

A model that attempts to resolve account or billing issues with generated content. The correct behavior for these is recognition followed by escalation, not a drafted answer. Attempting one would be actively unsafe, since it would mean fabricating account actions the system has no information or authority to take.

Production-scale retrieval with approximate nearest neighbor indexing or reranking. At a pool size of 75, brute-force cosine similarity is fast enough and considerably easier to reason about and debug.

## 2. Taxonomy

The taxonomy uses two dimensions, both discovered by reading the data rather than assumed in advance.

**Intent** covers what the customer wants: Account_Billing, Technical_Troubleshooting, Feature_Request, General_Question, Closure.

**Resolution flag** covers what Spotify does next: needs_dm, needs_more_info, answerable_now, already_closed.

These are separate axes because they answer different questions, and the data shows they vary independently. Both billing and technical issues can require DM verification. Both technical and feature-request messages sometimes get a clarifying question in reply. Collapsing these into a single axis would either inflate the category count or hide a genuinely useful signal.

## 3. Results versus baselines

Measured on 40 examples from the hand-labeled golden set.

| Method | Intent Accuracy | Flag Accuracy |
|---|---|---|
| Trivial (most common class) | 40.0% | 45.0% |
| Keyword matching | 67.5% | 42.5% |
| LLM classifier (this system) | 75.0% | 75.0% |

One result deserves attention: the keyword baseline scores *below* the trivial baseline on resolution_flag, at 42.5 percent against 45.0 percent. This is not a bug. It shows that naive keyword rules can perform worse than doing nothing at all, because the clearest signal for needs_dm is the word "DM," and that word appears in Spotify's reply rather than the customer's original message. A keyword classifier operating on customer text alone is structurally blind to it.

The LLM classifier's advantage concentrates almost entirely on this flag task, gaining 30 to 33 points over both baselines, because it can infer likely resolution behavior from context instead of requiring an explicit keyword match.

## 4. Failure analysis

**Failure 1: "No dice. Same problems."** Misclassified because the message is genuinely unintelligible without the prior turn it responds to. Hypothesis: a share of misclassifications are not model failures at all but a structural ceiling imposed by the single-turn dataset.

**Failure 2: "LISTEN I DON'T USE SPOTIFY THAT MUCH AND I DID SOMETHING THAT SKIPPED IT."** The same structural issue. Spotify's own historical reply ("the reply was for your first tweet to us") confirms this message only makes sense inside a longer thread.

**Failure 3: "Do you require any of my account details to do this?"** This is a meta-question about the DM process itself rather than a new issue. It is genuinely ambiguous even for a human labeler, and the taxonomy currently has no category for a process question about a prior interaction.

**Failure 4: Ad played during ad-free time, misclassified as Account_Billing.** A real model error rather than data ambiguity. Hypothesis: the message structure, combining frustration with a specific unexpected event, may pattern-match more closely to billing complaints than to technical bugs in the model's training distribution.

**Failure 5: Retrieval and grounding failure on a playback query.** For the message "My playlist keeps skipping songs randomly on my Samsung phone," retrieval returned a topically unrelated past case about downloading the app on a Roku device. The match was driven by surface phrasing similarity, since both replies share the template shape "Sure thing... Keep us posted." The reply-drafting step did not copy the irrelevant source. Instead it generated plausible troubleshooting advice that was not actually grounded in any retrieved evidence, and presented that advice with exactly the same confidence as a properly grounded reply. This confident hallucination under weak retrieval is arguably more dangerous than an obviously wrong answer, because nothing about the output distinguishes it from a correct one.

*Fix attempt 1, similarity threshold.* I added a minimum cosine-similarity cutoff of 0.35 before allowing grounding. This proved insufficient. The same irrelevant Roku match scored 0.537, comfortably clearing the threshold, and the system still produced an ungrounded reply. Embedding similarity on these short, template-heavy tweets tracks surface phrasing more than topical content, which makes a bare similarity cutoff an unreliable filter.

*Fix attempt 2, explicit relevance check.* Before drafting, a separate LLM call now asks directly whether the retrieved example concerns the same underlying issue as the new message. This worked. On the identical test case, with the same 0.54-scoring Roku match, the relevance check caught the mismatch and the system refused to draft a reply, escalating instead.

The progression from a cheap similarity heuristic to a more expensive semantic check is a concrete demonstration of why embedding similarity alone is not sufficient as a safety mechanism for grounded generation on short, templated support text.

## 5. What is misleading about my headline number

**Accuracy alone rewards majority-class prediction.** The needs_dm flag dominates this dataset at roughly 40 percent of examples, so a classifier that over-predicts it looks deceptively strong on aggregate accuracy. Per-class precision and recall would be a more honest measure than the single 75 percent figure.

**Ground truth reflects what an agent did, not the only correct action.** The resolution_flag labels record what a human agent historically chose. Some labeled examples appear to reflect agent inconsistency rather than a clean learnable signal, such as a general status question answered with a DM request. Some apparent model errors may therefore be disagreements with noisy ground truth.

**Evaluation ran on a subsample.** The 40 examples used are a subset of the full golden set, chosen under time constraints. The reported accuracy carries a wider confidence interval than the point estimate suggests.

**Grounding quality was not measured separately from reply quality.** A system can produce a fluent, well-formed reply that is ungrounded or misleading, and aggregate reply-quality metrics would not necessarily catch it. Failure 5 is exactly this case.

## 6. What I would do with one more week

Report per-class precision, recall, and F1 for both the classifier and the escalation decision rather than aggregate accuracy alone.

Build a proper LLM-as-judge harness for reply quality and validate it against my own manual judgment on a held-out sample, so the judge's agreement rate with a human is measured rather than assumed.

Reconstruct multi-turn threads using the response_tweet_id and in_response_to_tweet_id chains, recovering the context that Failures 1 and 2 depend on instead of treating every exchange as standalone.

Measure the relevance-check fix across the full evaluation set, specifically how often it correctly refuses against how often it over-refuses on genuinely answerable messages.

Extend the golden set toward the upper end of the requested range through continued careful labeling.
