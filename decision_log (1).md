# Decision Log

Non-obvious decisions made while building this system, and the reasoning behind each.

**1. Chose SpotifyCares over higher-volume brands such as AppleSupport or AmazonHelp.** The higher-volume brands cover far more sprawling issue types, which would have made a small, defensible intent taxonomy much harder to justify from the data. SpotifyCares had strong volume with a naturally narrow issue space.

**2. Used a two-dimensional taxonomy, intent plus resolution_flag, instead of a single flat intent list.** Reading the data showed that intent (what the customer wants) and resolution behavior (what Spotify does next) vary independently. Both billing and technical issues sometimes require DM verification. Folding these into one axis would have either inflated the category count or hidden a real signal.

**3. Kept the intent taxonomy at 5 categories rather than approaching Banking77's 77-category granularity.** A small, human-reviewable taxonomy is more defensible under time constraints and matches the level of distinction actually visible in this data.

**4. Excluded needs_dm replies from the retrieval and grounding pool.** These replies contain no real resolution content, since they only redirect the customer to a private channel. Grounding future replies on them would teach the system to deflect rather than resolve.

**5. Treated single tweet-reply pairs as the unit of analysis rather than reconstructing full threads.** The dataset's threading fields would support deeper reconstruction, but doing it reliably was out of scope for the time available. This is a known limitation, covered in Failure Analysis items 1 and 2.

**6. Retrieval safety went through two iterations after a confident-hallucination failure surfaced.** The first attempt used a cosine-similarity threshold, which proved insufficient: an irrelevant match scored 0.537 against a 0.35 cutoff, passing on surface phrasing similarity rather than topical relevance. The second attempt added an explicit LLM relevance check before drafting, which correctly caught the same failing case and escalated instead of hallucinating. Both attempts appear in the report rather than only the working one, since the progression itself shows why a naive similarity threshold is an insufficient safety mechanism here.

**7. Escalation logic is a direct rule on resolution_flag rather than a separately learned decision.** Since needs_dm cases require private account access that the system has no authority or information to perform, escalating on that flag is a safety default, not a judgment call that warrants a model.

**8. Golden set labeling: 141 of 200 examples were labeled manually, by reading each message and reply directly and applying the established criteria.** Early in the process, within roughly the first 21 rows, two systematic errors surfaced: misclassifying login and account issues as Technical_Troubleshooting, and over-applying already_closed to replies that were actually informative answers. Catching these sharpened the criteria applied to everything after. For the remaining 59 examples, AI assistance accelerated labeling under time pressure. A follow-up skim over those 59 rows found nothing obviously wrong, but time did not allow the same row-by-row verification the first 141 received, so this batch carries more uncertainty than the manually-labeled portion and is reported as such rather than presented as equally verified.

**9. Used openai/gpt-oss-120b through Groq rather than a larger paid model.** Quality is sufficient at this scope and cost is effectively zero, which matches the assignment's expectation of working on a subsample rather than at production scale.

**10. Measured baselines on the same subsample as the main system.** Using a different or larger test set for the baselines would have confounded the three-way comparison.

**11. Kept the surprising baseline result in the report rather than treating it as something to fix.** The trivial baseline scoring above the keyword baseline on resolution_flag is a genuine finding: naive rule-based heuristics can perform worse than doing nothing, which is itself an argument for the LLM-based approach.

**12. Did not build indexed vector search.** At this retrieval pool size, brute-force cosine similarity runs fast enough and stays far easier to reason about and debug than an approximate nearest neighbor index would be.

**13. Scored the judge's output against my own scores without adjusting mine to match.** After seeing the judge disagree with me on thirteen of sixty nine judgments, revising my scores toward the judge would have manufactured agreement rather than measured it. The disagreement is the finding.

**14. Reported Cohen's kappa alongside raw agreement for the judge evaluation.** Raw agreement of 73.9% on the SAFE criterion looks acceptable until you account for the fact that both scorers pass most items, which produces agreement by chance. Kappa of 0.14 tells a very different and more honest story.

**15. Excluded one Danish-language reply from human scoring rather than guessing at it.** I could not assess it reliably, and scoring it anyway would have quietly corrupted the agreement measurement. The exclusion is reported, and the underlying gap, that the pipeline drafts replies in languages nobody checks, is recorded as a limitation.
