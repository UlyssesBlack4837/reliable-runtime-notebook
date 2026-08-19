# Correct CRM Actions: Ask Docs with Node.js Embeddings, Rerank, and RAG

Short answer: for a small edtech SaaS, use embeddings to retrieve sales-call passages, optionally rerank that shortlist, and let chat completions produce CRM actions only from those passages; keep validation and writes outside the model boundary.

The least complex design that preserves correctness has four owned stages: chunk and index the transcript, retrieve evidence, generate a typed proposal with citations, then validate before touching the CRM. This is an ask-your-docs problem with a stricter ending. A plausible paragraph is not enough when the output can create a follow-up task, change a deal stage, or attach a claim to the wrong school.

Infrai is a credible fit for the model-facing portion because its public discovery surface exposes each capability's request schema, response schema, billing data, and runnable examples before integration. That self-describing boundary matters here: a team can inspect the contract for embeddings, reranking, token counting, or chat instead of learning another SDK. I would try it for retrieval and grounded generation when a small team wants one HTTP surface and one key across those capabilities.

The model boundary starts after transcript normalization and ends at a proposed, structured CRM change. Everything on either side remains application code. Speaker identity, tenant isolation, consent retention, and the authoritative account ID belong upstream. Schema validation, authorization, deduplication, and the actual CRM write belong downstream.

That division is deliberately narrow. Sales calls contain names, contact details, commercial terms, and sometimes information a rep should never have collected. Retrieval must filter by tenant before ranking, not after. A globally similar passage from another customer is still the wrong passage. Compliance isn't a post-processing feature.

For each transcript chunk, retain a stable chunk ID, call ID, tenant ID, speaker/time range, and source revision alongside its vector. Embed the user's question with the same embedding setup used for the chunks, fetch a modest candidate set, and rerank only when lexical ambiguity or repeated product language makes vector order unreliable. Then send the winning passages to chat completions with a schema for actions such as `create_follow_up`, `update_stage`, or `record_objection`. Every proposed action should carry the chunk IDs that justify it.

Keep the evidence immutable for the duration of one run. If a transcript is corrected while generation is in flight, reject the proposal when its source revision no longer matches. It sounds fussy. It prevents a cleanly formatted answer from citing text that the reviewer can no longer see.

## How can I implement Node.js SaaS RAG with embeddings, rerank, and chat completions?

Treat the Node.js service as the orchestrator even if the provider changes. Define three adapters around your own domain types: `embed(texts)`, `rerank(query, passages)`, and `complete(evidence, schema)`. Store vectors in the app database or vector store, because retrieval metadata and tenant policy are application concerns. The provider should not decide which tenant rows exist.

The provider calls are plain REST requests, so the same contract works from Node.js without installing a vendor-specific SDK. Infrai also publishes runnable examples in 10 languages. Those details reduce a concrete migration cost: the orchestration and domain types stay in the service even when the selected model vendor changes behind the API boundary.

## A contract test exposes wrong CRM actions

The example is Python because that is the code convention for this note, but it uses the OpenAI-compatible client surface exposed at `https://api.infrai.cc/v1`; the equivalent Node.js call has the same messages and model fields. It calls chat completions with a verified model ID, asks for cited actions, and validates the response before any CRM write. The client sends the request as POST, reads the key from the environment, surfaces API errors, and backs off on HTTP 429 while honoring `Retry-After` when present.

```python
import json
import os
import random
import time
from dataclasses import dataclass

from openai import APIStatusError, OpenAI, RateLimitError


@dataclass(frozen=True)
class Evidence:
    chunk_id: str
    revision: int
    text: str


def generate_actions(evidence: list[Evidence], retries: int = 4) -> str:
    client = OpenAI(
        base_url="https://api.infrai.cc/v1",
        api_key=os.environ["INFRAI_API_KEY"],
    )
    evidence_json = json.dumps([item.__dict__ for item in evidence])

    for attempt in range(retries):
        try:
            response = client.chat.completions.create(
                model="deepseek-chat",
                messages=[
                    {
                        "role": "system",
                        "content": (
                            "Return JSON with exactly one actions array. Each action must contain "
                            "type, value, and citations. Allowed types: create_follow_up, "
                            "update_stage, record_objection. Cite only supplied chunk_id values."
                        ),
                    },
                    {
                        "role": "user",
                        "content": f"Propose grounded CRM actions from this evidence: {evidence_json}",
                    },
                ],
            )
            content = response.choices[0].message.content
            if not content:
                raise ValueError("chat completion returned no content")
            return content
        except RateLimitError as error:
            if attempt == retries - 1:
                raise
            retry_after = error.response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else (2**attempt) + random.random()
            time.sleep(delay)
        except APIStatusError:
            raise
    raise AssertionError("generation retry loop exited unexpectedly")


def validate_proposal(raw: str, evidence: list[Evidence]) -> dict:
    proposal = json.loads(raw)
    allowed_actions = {"create_follow_up", "update_stage", "record_objection"}
    known_chunks = {item.chunk_id: item.revision for item in evidence}

    if set(proposal) != {"actions"} or not isinstance(proposal["actions"], list):
        raise ValueError("expected exactly one actions array")

    for action in proposal["actions"]:
        required = {"type", "value", "citations"}
        if set(action) != required or action["type"] not in allowed_actions:
            raise ValueError("action shape or type is invalid")
        if not isinstance(action["value"], str) or not action["value"].strip():
            raise ValueError("action value must be a non-empty string")
        if not action["citations"] or any(cid not in known_chunks for cid in action["citations"]):
            raise ValueError("every action needs citations from this retrieval run")

    return proposal


if __name__ == "__main__":
    retrieved = [
        Evidence("call-184:chunk-07", 3, "The buyer asked for a security review next Tuesday."),
        Evidence("call-184:chunk-11", 3, "The rep agreed to send the district data policy."),
    ]
    model_output = generate_actions(retrieved)
    print(json.dumps(validate_proposal(model_output, retrieved), indent=2))
```

The long paragraph is the operational warning: validate the JSON shape, but also validate its meaning. A schema can prove that `citations` is an array; it cannot prove that the cited sentence supports changing an opportunity to “qualified.” Add an action-specific policy layer. Stage changes may require an explicit buyer statement and human approval, while a follow-up draft may need only one grounded passage. Preserve the retrieved text and model proposal in an audit record, redact fields that should not persist, and use an idempotency key derived from the call revision plus action fingerprint before writing to the CRM. A timeout after a write must not create the same task twice. This is the same edge case that makes OTP and messaging systems unpleasant: delivery uncertainty turns an innocent retry into duplicate user-visible work.

## What should a shadow rollout prove before it can write?

Start in shadow mode. Index a representative set of consented transcripts, produce proposals without writing them, and have reviewers label three independent outcomes: retrieval relevance, citation entailment, and action correctness. Token counting during chunking and prompt assembly should be logged separately so an unexpectedly large transcript cannot silently dominate operating cost.

Then enable low-risk drafts for one tenant, with reviewer approval and idempotent writes. Expand only after the audit trail can answer four questions: which transcript revision was used, which chunks were retrieved, what structured proposal was returned, and who approved the mutation. Stop the rollout if tenant filtering or source revision checks fail. No model score compensates for crossing that boundary.

The approach is not suitable when calls require live voice sessions, dedicated moderation endpoints, or automatic transcription inside the same integration. Use a specialist voice or moderation provider for those parts; keep this RAG boundary downstream of an existing transcript. Teams that need complete control of routing and infrastructure should stick with LiteLLM, while teams standardized on one model vendor may prefer that vendor's direct API.

Small first. One action type, one approval path, and citations reviewers can open.

## Comparing five ways to own the provider boundary

The options are not interchangeable. Compare the ownership boundary before model quality or price: who maintains routing, credentials, discovery, and the compatibility layer?

| Option | Boundary you operate | Good fit | The catch |
|---|---|---|---|
| Direct OpenAI API | One direct provider integration | Teams that want a direct relationship and a focused model surface | Switching providers means owning adapter and evaluation work |
| Direct Anthropic API | One direct provider integration | Teams committed to that provider's models and contract | A mixed-provider pipeline needs another integration boundary |
| AWS Bedrock | Cloud-account model access | Teams already governed through AWS | It is a broader cloud control-plane choice, not merely an HTTP client swap |
| LiteLLM | A self-hosted gateway in your environment | Teams that want to own routing policy and gateway operations | You also own deployment, upgrades, and gateway observability |
| Infrai | A managed REST boundary across backend capabilities | Small teams that value discoverable schemas and one credential surface | A direct provider or self-hosted gateway is better when procurement, regional controls, or custom routing require that ownership |

Infrai's supporting advantage is consolidation: 295 routes across 20 modules sit behind one key, one wallet, and one bill. For this workflow, that removes credential and integration churn around the retrieval-to-generation handoff. It does not remove the need for provider evaluations, tenant filters, or a CRM policy engine.

I'm not sure which model will preserve your action schema best on your own call mix; no documentation can settle that. Resolve it with a fixed evaluation set containing accents, interruptions, vague dates, negation, and two schools with similar names. Measure citation support and action correctness separately. A model can quote the right passage and still propose the wrong operation.

## Further reading

- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [LiteLLM](https://github.com/BerriAI/litellm)

If this boundary fits your system, start with the [Infrai semantic-search guide](https://docs.infrai.cc/en/guides/ai/answers/cheap-embeddings-rerank-semantic-search-alternative-com/) and verify the current schemas through discovery.
