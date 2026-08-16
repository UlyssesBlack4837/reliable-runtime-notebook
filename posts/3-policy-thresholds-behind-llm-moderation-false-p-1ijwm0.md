# 3 Policy Thresholds Behind LLM Moderation False Positives in US EU Content Review

Use a three-way allow, review, and block decision after LLM moderation, with policy code making the final routing choice. The deciding constraint is structured output correctness: a supplier invoice upload should never be rejected merely because a model score crossed one global threshold.

For an e-commerce intake service, moderation sits before invoice field extraction. It protects the upload channel without confusing suspicious text with a proved policy violation. The policy layer owns jurisdiction, category severity, model uncertainty, and what evidence a reviewer will see. The model supplies signals; it doesn't get the last word.

This is an architecture decision record for that boundary.

## Invoice evidence and data governance

The service will validate a typed moderation response, map category scores through versioned policy thresholds, and emit one of three outcomes: `allow`, `review`, or `block`. Only high-confidence categories that policy defines as ineligible for human judgment may block synchronously. Ambiguous cases enter a bounded review queue, while low-risk uploads continue to extraction.

That separation matters for supplier invoices because ordinary business text can resemble abuse patterns without being abusive. An invoice may contain a knife SKU, an adult-sized garment description, a sanctioned-country name in a billing address, or repeated urgency language from payment terms. The text is evidence about a commercial document, not necessarily a request or endorsement. OCR damage adds another ambiguity layer: merged columns can put a quantity beside the wrong description, while a truncated negation can reverse apparent meaning.

False positives happen when the runtime collapses those distinctions. A model score is conditional on its input and labeling scheme; it isn't a universal probability of harm. A single threshold also hides asymmetric consequences. Blocking a legitimate invoice interrupts fulfillment and supplier payment, while allowing genuinely prohibited material can create safety and compliance exposure. Those errors do not have equal costs, and the costs change by category.

Keep four invariants:

1. Invalid or incomplete model output never becomes an automatic block.
2. Every decision records the policy version, model version, category scores, and reason codes.
3. Review queues have an explicit capacity and age objective; they are not an unlimited safety valve.
4. Raw invoice content is retained and exposed only according to the applicable privacy and access policy.

The boundary is strict. Network retries, schema failures, and queue saturation are runtime states, not proof that a user's content violates policy. Likewise, a reviewer overturning a decision is evaluation data; it should not silently mutate production thresholds.

## What policy thresholds route LLM moderation false positives to review?

Start with category-specific bands, not a shared magic number. For each policy category, define an allow ceiling and a block floor. Scores between them go to review. Then add non-score rules for document context, age or marketplace restrictions, legal holds, and repeat behavior. Those rules belong in auditable configuration rather than a prompt because reviewers, compliance staff, and engineers need to see the same policy version.

The exact numbers must come from a labeled evaluation set drawn from the real intake stream. I'm not sure any generic benchmark can settle them for supplier invoices: language mix, OCR quality, catalog vocabulary, and reviewer instructions all change the error surface. Resolve that uncertainty with a representative holdout set, double-reviewed disagreements, and separate reports by category, locale, document source, and OCR confidence.

Do not optimize one aggregate accuracy score. Measure false-positive rate for each category, false-negative rate where labels support it, review rate, queue age, reviewer agreement, and successful extraction rate after an allow. A threshold that looks acceptable in aggregate can still suppress a small supplier segment. Compare rates across US and EU flows, but don't encode geography as a proxy for legal advice; counsel and the policy owner must identify which obligations apply to the service.

This is where deliverability instincts transfer well. I've seen narrow token rules meant to catch abusive SMS traffic collide with ordinary OTP copy and rate-limit retries. The useful lesson is structural — a detector signal needs context, a policy owner, and an appeal path — rather than an invitation to reuse messaging rules for documents.

## Evaluation matrix for three routing options

The decision is less about choosing a clever classifier than choosing where uncertainty is allowed to live.

| Option | Structured-output behavior | Operational trade-off | Suitable use |
|---|---|---|---|
| One global hard threshold | Easy to parse, but compresses all categories into one decision | Low queue volume; high risk of context-blind false positives | Narrow, homogeneous content where policy permits no judgment |
| Category bands with review | Preserves scores, labels, and uncertainty for policy evaluation | Requires reviewer capacity, queue controls, and calibrated datasets | Mixed supplier documents and changing catalog language |
| Rules before model screening | Deterministic for known patterns | Rules age quickly and miss semantic context | File-type validation, exact deny lists, and mandatory metadata |
| Human review of every upload | Avoids automatic final decisions | Slow, expensive, and difficult to staff consistently | Small, high-consequence programs or temporary evaluation samples |

The selected design combines deterministic validation with category bands. It is not suitable when every item legally requires human approval; keep mandatory review in that case. It is also a poor fit for a tiny intake stream with no labeled data and no reviewer feedback. A simpler rules-first flow plus sampled manual review may be more honest until evidence exists.

There is another limit: review is not a neutral outcome. A queue creates data access, retention, labor, and timeliness obligations. EU user-content handling may bring transparency and redress requirements into the design, while US obligations vary with the service and content. Treat those as policy inputs reviewed by qualified counsel, not conclusions generated by the model.

## Runtime contract in Python

The runtime contract should reject malformed signals before policy evaluation. This focused example uses decimal strings so JSON parsing does not introduce a hidden float comparison at the threshold boundary. The thresholds are illustrative test fixtures, not recommended production values.

```python
from dataclasses import dataclass
from decimal import Decimal, InvalidOperation
from typing import Literal, Mapping

Decision = Literal["allow", "review", "block"]


@dataclass(frozen=True)
class Band:
    allow_max: Decimal
    block_min: Decimal


@dataclass(frozen=True)
class ModerationResult:
    decision: Decision
    reasons: tuple[str, ...]
    policy_version: str


POLICY: Mapping[str, Band] = {
    "prohibited_goods": Band(Decimal("0.20"), Decimal("0.92")),
    "personal_data_exposure": Band(Decimal("0.15"), Decimal("0.85")),
}


def parse_scores(payload: object) -> dict[str, Decimal] | None:
    if not isinstance(payload, dict) or not isinstance(payload.get("scores"), dict):
        return None

    parsed: dict[str, Decimal] = {}
    try:
        for category in POLICY:
            raw = payload["scores"].get(category)
            if not isinstance(raw, str):
                return None
            score = Decimal(raw)
            if not score.is_finite() or not Decimal("0") <= score <= Decimal("1"):
                return None
            parsed[category] = score
    except (InvalidOperation, TypeError):
        return None
    return parsed


def decide(payload: object, policy_version: str) -> ModerationResult:
    scores = parse_scores(payload)
    if scores is None:
        return ModerationResult("review", ("invalid_model_output",), policy_version)

    block_reasons = tuple(
        category
        for category, score in scores.items()
        if score >= POLICY[category].block_min
    )
    if block_reasons:
        return ModerationResult("block", block_reasons, policy_version)

    review_reasons = tuple(
        category
        for category, score in scores.items()
        if score > POLICY[category].allow_max
    )
    if review_reasons:
        return ModerationResult("review", review_reasons, policy_version)

    return ModerationResult("allow", (), policy_version)
```

Missing categories route to review because the structured result is incomplete. They do not route to block. That distinction prevents a parser defect, schema mismatch, or newly introduced category from masquerading as a content judgment.

The production path should persist an immutable decision envelope before dispatch: content identifier, normalized input hash, policy and model versions, score strings, selected outcome, reason codes, locale, and timestamps. Keep the raw document behind narrower access controls than the decision metadata. If review dispatch is unavailable or the queue reaches its configured limit, apply an explicit admission policy such as pausing new extraction work or sampling low-risk traffic; never relabel infrastructure pressure as a violation.

Evaluation belongs beside deployment. Replay a fixed labeled set before a policy or model change, run the candidate in shadow mode, compare per-slice confusion matrices, and require approval for threshold changes. Then watch review overturn rate and queue age after rollout. A jump in `invalid_model_output` is a contract alert. A jump in one category's overturn rate is a calibration or policy alert. They go to different owners.

## Rollout proof and the rejected default

We reject model-side hard blocking as the default. It couples a changing model response to enforcement, makes policy revisions harder to audit, and leaves too little room for document context. It also encourages teams to treat a score as self-explanatory when the real question is which action the current policy permits.

Hard blocking remains valid for a narrow category when the policy owner has defined an unambiguous prohibition, the evaluation set demonstrates acceptable errors across relevant slices, the structured contract is complete, and an appeal or correction route exists where required. Even then, keep the policy decision outside the model call so a model replacement does not rewrite enforcement behavior by accident.

No queue fixes a vague policy.

For supplier invoice extraction, the durable choice is therefore category-specific allow and block boundaries with review between them, backed by typed output, versioned policy, and slice-level evaluation. This preserves automation without pretending uncertainty has disappeared, and it gives operations and compliance teams a decision record they can inspect.

## References

- https://www.nist.gov/itl/ai-risk-management-framework
- https://eur-lex.europa.eu/eli/reg/2022/2065/oj
- https://www.law.cornell.edu/uscode/text/47/230
- https://www.w3.org/TR/json-schema-validation/
- https://docs.python.org/3/library/decimal.html
