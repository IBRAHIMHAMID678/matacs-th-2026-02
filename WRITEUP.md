# AI-Assisted Proposal Triage Design Write-Up
Reference: MATACS-TH-2026-02

### 1. AI vs. Rules
This design divides tasks between language interpretation and deterministic code. Language models excel at reading unstructured text, finding contractor qualifications across diverse phrasing, and drafting plain explanations. Rules in code excel at arithmetic, sorting, logic gates, and compliance checks.

The workflow uses artificial intelligence in two places:
1. Information extraction: An AI model reads proposal text and maps details into structured fields (licence numbers, projects, cost, schedule, warranty, and crew). The model must return an exact quote for each field.
2. Reviewer explanation drafting: An AI model converts verified subscores and facts into a concise summary sentence explaining why a proposal achieved its rank.

Deterministic code handles all other steps:
- Document parsing and text pre-processing.
- Screening for prompt injection signatures.
- Verifying cited quotes match raw document text.
- Evaluating mandatory pass/fail criteria (licence, insurance, and WSIB).
- Calculating weighted scores using mathematical formulas.
- Sorting proposals by total score and assigning status.
- Persisting audit logs and reviewer actions.

We draw the line here because math, compliance, and ranking must be repeatable and legally defensible. If an AI calculates scores, it can make arithmetic errors, drift across runs, or obey prompt injections. Restricting AI to factual extraction keeps evaluation objective. The exact weighting formulas, including how each metric maps to a subscore, are specified in PROMPTS.md.

### 2. Consistency and Explainability
Two identical runs on the same proposals produce identical numerical scores and ranking positions.

Repeatability is achieved through three controls:
- The AI extraction prompt runs at temperature zero with strict schema validation.
- Extracted facts require exact grounding. A rule validator checks that every cited quote exists word-for-word in the proposal text. If string matching fails, the claim is discarded.
- Scoring, weighting, and ranking are executed entirely by deterministic formulas in code. Once factual inputs are verified, the arithmetic produces the exact same score every time. Flagged proposals are scored like any other. A flag changes the proposal's review priority, never its arithmetic.

Reviewers see why a proposal ranked where it did through an interactive scorecard breaking composite scores into criterion subscores. Beside each subscore, the reviewer sees the extracted metric, the quoted sentence, and a link to that passage. If a proposal fails a mandatory condition or omits an expected value, a warning tag explains the disqualification or deduction.

### 3. Human in the Loop
Human judgment remains the central authority. The software assists reviewers with intake, but never makes unilateral award decisions.

The design defines three human checkpoints:
1. Admin criteria configuration: Administrators set evaluation criteria, point weights, and thresholds before reviewers process bids.
2. Reviewer triage review: Reviewers inspect the ranked list, review warning flags, and verify evidence quotes. Reviewers can accept the automated result or override any extracted value, pass/fail determination, or subscore.
3. Final award decision: A facilities manager selects the winning contractor. The human may choose a lower-ranked vendor if operational needs require, provided they submit written justification.

When a reviewer overrides a finding, the system requires a reason code and explanation. The append-only audit log records proposal and project identifiers, user ID, UTC timestamp, criteria version, document hash, raw AI extracted facts and quotes, original scores and status, and modified values with reviewer justification.

### 4. Changing Criteria
Evaluation criteria are never hard-coded. Administrators configure them through administrative screens, storing them with incrementing version IDs (such as v1.0, v1.1).

If an administrator edits criteria or weights after proposals have been scored, the system executes four steps:
1. Version freeze: Existing evaluation records remain tied to the criteria version used during scoring. Past scores are never overwritten.
2. Invalidation notification: The reviewer interface displays a badge stating that scores were calculated under an earlier criteria version and are outdated.
3. Instant re-scoring branch: The reviewer can trigger a one-click re-score. Because extracted facts and quotes are stored separately from scoring rules, the system re-evaluates cached facts against new weights without re-running parsing or AI extraction. This saves time and avoids API fees.
4. Difference view: The dashboard displays a side-by-side comparison showing previous scores, new scores, and rank movements.

### 5. Risks and Safeguards
The system guards against five primary risks:

1. Wrong extraction:
An AI model might misread numbers or technical specifications. 
Safeguard: The model must provide verbatim quotes for every field. A code validator confirms that the quote exists in the source text. The interface highlights cited quotes side-by-side with original text for quick human verification.

2. Hallucinated facts:
A model might invent unstated details to fill information gaps.
Safeguard: Prompts enforce strict grounding: unstated items must return as null. Any claim lacking a verifiable quote is discarded. Missing values trigger automated flags rather than guessed scores.

3. Evaluation bias:
Subjective writing style, company prestige, or layout polish could influence results.
Safeguard: The scoring engine evaluates normalized objective data points rather than persuasive prose. Vendor identities and marketing statements are stripped before formulas compute experience, price, timeline, and warranty scores.

4. Gaming and manipulation:
A bidder could embed hidden text or prompt injections attempting to command the AI to award full marks.
Safeguard: Ingestion runs an adversarial pattern filter. Furthermore, separating extraction from scoring neutralizes injections. Because scoring is executed by code formulas, embedded prompt directives cannot enter the arithmetic at all. The note is recorded as a flag on the proposal, while the proposal itself is still scored on its verified facts. An injection can raise attention but cannot change a number.

5. API cost and latency:
Processing long technical proposals through large models can become slow and expensive.
Safeguard: Documents undergo single-pass entity extraction using lightweight models. Extracted facts are cached in a database, allowing criteria adjustments and re-scoring without extra AI API expenses.
