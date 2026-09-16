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

Measured on 40 examples in an initial pass, then re-measured on the full 200-example golden set. The full-set numbers are the ones that should be treated as the real headline result; the 40-example figures overstated flag accuracy.

| Method | Intent Accuracy | Flag Accuracy |
|---|---|---|
| Trivial (most common class) | 40.0% | 45.0% |
| Keyword matching | 67.5% | 42.5% |
| LLM classifier, 40 examples | 75.0% | 75.0% |
| LLM classifier, full 200 examples | 79.5% | 67.0% |

One result deserves attention: the keyword baseline scores *below* the trivial baseline on resolution_flag, at 42.5 percent against 45.0 percent. This is not a bug. It shows that naive keyword rules can perform worse than doing nothing at all, because the clearest signal for needs_dm is the word "DM," and that word appears in Spotify's reply rather than the customer's original message. A keyword classifier operating on customer text alone is structurally blind to it.

The LLM classifier's advantage concentrates on the flag task even at full scale, gaining roughly 22 to 25 points over both baselines.

### Per-class breakdown

Aggregate accuracy hides which categories the classifier actually handles well. Measured on all 200 examples:

| Intent | Precision | Recall | F1 | n |
|---|---|---|---|---|
| Account_Billing | 0.93 | 0.80 | 0.86 | 69 |
| Closure | 0.76 | 1.00 | 0.87 | 13 |
| Technical_Troubleshooting | 0.78 | 0.82 | 0.80 | 49 |
| Feature_Request | 0.82 | 0.73 | 0.77 | 37 |
| General_Question | 0.60 | 0.75 | 0.67 | 32 |

| Resolution flag | Precision | Recall | F1 | n |
|---|---|---|---|---|
| needs_dm | 0.84 | 0.69 | 0.76 | 75 |
| already_closed | 0.79 | 0.85 | 0.81 | 13 |
| answerable_now | 0.62 | 0.80 | 0.70 | 75 |
| needs_more_info | 0.41 | 0.30 | 0.34 | 37 |

Two things stand out. General_Question is the weakest intent class: when the model predicts it, it is wrong 40 percent of the time, consistent with the failure examples in Section 5 where technical bugs and feature requests get defaulted to General_Question when the model is uncertain.

needs_more_info is a clear outlier on the flag side, with F1 less than half of every other class. The model both over-predicts it and misses most real cases of it. Because needs_dm and answerable_now, the two largest classes, perform reasonably well, the aggregate 67 percent flag accuracy conceals a specific, severe weakness on a smaller class. This is direct evidence for the aggregate-accuracy concern raised in Section 6.

## 4. Evaluation harness

Classification accuracy measures whether the system routes a message correctly. It says nothing about whether the replies it drafts are any good. To measure reply quality I built an LLM-as-judge harness and then tested whether that judge can actually be trusted.

### Setup

Thirty messages from the golden set were run through the full pipeline. Twenty four produced a drafted reply and six escalated. Each of the twenty four replies was scored on three binary criteria:

**RELEVANT.** Does the reply address what this specific customer asked about? A closure acknowledgment sent to someone with an open problem fails.

**APPROPRIATE.** Is the tone right for a brand support account, given the customer's mood? Upbeat filler in response to anger or a serious complaint fails.

**SAFE.** Does the reply avoid inventing features, menus, policies, or timelines, avoid overpromising, and avoid mishandling sensitive information posted publicly?

The same twenty four replies were then scored independently by me, using the same written criteria, without reference to the judge's output. One reply was written in Danish and excluded, since I could not assess it reliably. That exclusion is itself worth noting: the pipeline drafts replies in languages the evaluator cannot check, and nothing in the system currently flags this.

### Reply quality as measured by the judge

| Criterion | Judge pass rate | My pass rate |
|---|---|---|
| RELEVANT | 83.3% | 78.3% |
| APPROPRIATE | 87.5% | 87.0% |
| SAFE | 91.7% | 73.9% |

### Judge agreement with a human

Aggregate pass rates can match closely while the two scorers disagree on individual rows, so agreement was measured row by row. Cohen's kappa is reported alongside raw agreement because when both scorers pass most items, a high raw agreement figure is largely produced by chance.

| Criterion | Raw agreement | Cohen's kappa |
|---|---|---|
| RELEVANT | 87.0% (20/23) | 0.59 |
| APPROPRIATE | 82.6% (19/23) | 0.23 |
| SAFE | 73.9% (17/23) | 0.14 |
| Overall | 81.2% (56/69) | |

Under the standard interpretation of kappa, 0.41 to 0.60 is moderate, 0.21 to 0.40 is fair, and below 0.20 is slight. Only RELEVANT reaches moderate agreement. SAFE, at 0.14, is barely distinguishable from chance.

### What this means

The judge is not currently trustworthy as a substitute for human review, and it fails worst on the criterion that matters most.

Of thirteen disagreements, nine are cases where the judge was more lenient than I was, and six of those nine fall on SAFE. The clearest example is a reply that redirected a customer to email in the middle of an active troubleshooting exchange, where the customer had just supplied their device and OS version. I failed it on all three criteria. The judge passed it on all three.

The judge also missed a reply that acknowledged a customer's publicly posted email address without flagging it, which real SpotifyCares agents in this dataset do warn people about, and a cheerful closure sent to a customer who had just accused Spotify of being thieves.

The judge was stricter than me in four cases, so the leniency is not uniform, but the direction is clearly skewed.

The practical consequence is that the reply quality figures in the first table above should not be treated as validated. An automated safety score of 91.7% is produced by an instrument that agrees with human safety judgment at roughly chance levels. Any decision to auto-send replies based on that number would be resting on a measurement that has not earned trust.

## 5. Failure analysis

**Failure 1: "No dice. Same problems."** Misclassified because the message is genuinely unintelligible without the prior turn it responds to. Hypothesis: a share of misclassifications are not model failures at all but a structural ceiling imposed by the single-turn dataset.

**Failure 2: "LISTEN I DON'T USE SPOTIFY THAT MUCH AND I DID SOMETHING THAT SKIPPED IT."** The same structural issue. Spotify's own historical reply ("the reply was for your first tweet to us") confirms this message only makes sense inside a longer thread.

**Failure 3: "Do you require any of my account details to do this?"** This is a meta-question about the DM process itself rather than a new issue. It is genuinely ambiguous even for a human labeler, and the taxonomy currently has no category for a process question about a prior interaction.

**Failure 4: Ad played during ad-free time, misclassified as Account_Billing.** A real model error rather than data ambiguity. Hypothesis: the message structure, combining frustration with a specific unexpected event, may pattern-match more closely to billing complaints than to technical bugs in the model's training distribution.

**Failure 5: Retrieval and grounding failure on a playback query.** For the message "My playlist keeps skipping songs randomly on my Samsung phone," retrieval returned a topically unrelated past case about downloading the app on a Roku device. The match was driven by surface phrasing similarity, since both replies share the template shape "Sure thing... Keep us posted." The reply-drafting step did not copy the irrelevant source. Instead it generated plausible troubleshooting advice that was not actually grounded in any retrieved evidence, and presented that advice with exactly the same confidence as a properly grounded reply. This confident hallucination under weak retrieval is arguably more dangerous than an obviously wrong answer, because nothing about the output distinguishes it from a correct one.

*Fix attempt 1, similarity threshold.* I added a minimum cosine-similarity cutoff of 0.35 before allowing grounding. This proved insufficient. The same irrelevant Roku match scored 0.537, comfortably clearing the threshold, and the system still produced an ungrounded reply. Embedding similarity on these short, template-heavy tweets tracks surface phrasing more than topical content, which makes a bare similarity cutoff an unreliable filter.

*Fix attempt 2, explicit relevance check.* Before drafting, a separate LLM call now asks directly whether the retrieved example concerns the same underlying issue as the new message. This worked. On the identical test case, with the same 0.54-scoring Roku match, the relevance check caught the mismatch and the system refused to draft a reply, escalating instead.

The progression from a cheap similarity heuristic to a more expensive semantic check is a concrete demonstration of why embedding similarity alone is not sufficient as a safety mechanism for grounded generation on short, templated support text.

## 6. What is misleading about my headline number

**Accuracy alone rewards majority-class prediction.** The needs_dm flag dominates this dataset at roughly 40 percent of examples, so a classifier that over-predicts it looks deceptively strong on aggregate accuracy. Section 3 already reports per-class precision and recall rather than relying on the single 67.0 percent flag accuracy figure, and that breakdown shows exactly this problem in practice: needs_more_info scores an F1 of 0.34 while the aggregate number stays respectable because the two largest classes perform well.

**Ground truth reflects what an agent did, not the only correct action.** The resolution_flag labels record what a human agent historically chose. Some labeled examples appear to reflect agent inconsistency rather than a clean learnable signal, such as a general status question answered with a DM request. Some apparent model errors may therefore be disagreements with noisy ground truth.

**The initial 40-example evaluation overstated flag accuracy.** Re-running on the full 200-example golden set dropped flag accuracy from 75.0 percent to 67.0 percent, an 8-point difference that a smaller sample simply could not reveal. The reply-quality judge evaluation in Section 4 still runs on a 24-example subsample of the golden set, chosen because human scoring does not scale the way automated classification does; that figure carries a correspondingly wider margin.

**Grounding quality was not measured separately from reply quality.** A system can produce a fluent, well-formed reply that is ungrounded or misleading, and aggregate reply-quality metrics would not necessarily catch it. Failure 5 is exactly this case.

**The reply quality numbers rest on an unvalidated instrument.** The LLM-as-judge scores of 83.3%, 87.5% and 91.7% look reassuring in isolation. Section 4 shows the judge agrees with human safety judgment at a Cohen's kappa of 0.14, which is close to chance. Quoting the safety figure without that context would be the single most misleading thing in this report.

**Non-English replies were excluded from human evaluation.** The pipeline drafted a Danish reply that I could not assess. It was excluded rather than scored, which means the reported quality figures describe only the subset of output the evaluator could read, and the system gives no signal when it produces output in a language nobody has checked.

## 7. What I would do with one more week

Report per-class precision, recall, and F1 for both the classifier and the escalation decision rather than aggregate accuracy alone.

Improve the judge rather than just measure it. The current judge is too lenient on safety, so the next step is a stricter rubric with explicit failure examples in the prompt, scored on a scale rather than binary PASS/FAIL, and re-measured against human judgment to see whether kappa improves. A second scorer would also help establish whether my own labels are the reliable half of that comparison.

Reconstruct multi-turn threads using the response_tweet_id and in_response_to_tweet_id chains, recovering the context that Failures 1 and 2 depend on instead of treating every exchange as standalone.

Measure the relevance-check fix across the full evaluation set, specifically how often it correctly refuses against how often it over-refuses on genuinely answerable messages.

Extend the golden set toward the upper end of the requested range through continued careful labeling.

Handle non-English messages explicitly, either by detecting them and escalating, or by bringing in an evaluator who can assess them. Right now they pass through the pipeline unflagged and drop out of evaluation.
