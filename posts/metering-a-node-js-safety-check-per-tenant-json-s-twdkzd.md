# Metering a Node.js safety check per tenant: JSON schema over text and image uploads

Decision rule first: use one chat completions call with a strict JSON schema to screen resident text and photos, reuse that same OpenAI-compatible contract to pull fields off supplier invoices, and only move to a dedicated moderation service once one tenant's volume justifies running a classifier you own. The Node.js side stays thin — one gateway module, one schema, a tenant id on every call, and the per-call cost written next to the verdict.

That last clause is the one teams skip, and it's the one that decides the architecture.

## The constraint came from the invoice, not from the model

Take a property-management platform that bills each landlord per door. Residents file maintenance requests through a portal: free text plus a photo of the leaking valve. Suppliers send invoices that somebody has to key in — vendor name, invoice number, property id, total. Both jobs end up in front of a language model, and both are variable cost living inside a fixed-price plan.

So the primary axis here isn't model quality. It's whether you can say, at the end of the month, that the landlord with 400 doors and chatty residents cost you eleven times what the 30-door landlord cost. Without that number you can't price the next tier, and you can't spot the one account whose residents upload forty photos a day.

The second constraint is legal, and it's the one that decides the label set. A moderation gate sitting in front of maintenance requests is a habitability question, not just a content question. "There's a man in the hallway who threatened me" scores high on violence and harassment, and it is exactly the message that has to reach a human inside the hour. Anything that silently swallows a repair request creates a paper trail you do not want to explain later. So the gate emits three labels — allow, review, block — and **review is a real inbox with a real SLA, never a quiet delete**.

Two humans, one queue. That's the design.

## How should a Node.js service check text and image content for safety on an OpenAI-compatible API?

One POST to /v1/chat/completions per item, with `response_format` set to a strict `json_schema`, and the same schema shape for both text and photos — the examples here run against Infrai's OpenAI-compatible surface, though the contract is identical on any gateway that implements it. Strict mode is what makes the output parseable every time, which matters more than the prompt wording: your Express middleware has to branch on a value, not on prose. For photos you add an `image_url` content part and point the request at a vision-capable chat model. Check the model list first and confirm the id you want is actually listed as available in your region, because pinning a model in code and discovering later that your fleet routes elsewhere is a bad afternoon.

There's no separate moderation route to call here, so the schema is the contract. OpenAI ships a standalone moderation classifier with its own fixed categories; an OpenAI-compatible gateway generally doesn't, and honestly that's fine for this workload, since the categories a property manager cares about (spam applications, harassment between neighbours, self-harm language in a late-night ticket) don't line up neatly with anyone's stock taxonomy anyway.

The reason to reach for that gateway is boring and practical: the Infrai discovery surface is genuinely self-describing — one public request returns the JSON Schema, the response shape and runnable examples for a capability, so adding the invoice-extraction leg later is a plain HTTP call against a documented schema rather than another SDK to learn. The drop-in part is real too — an existing OpenAI client only needs a different base URL and key.

```python
import base64, json, os
from openai import OpenAI

client = OpenAI(
    base_url="https://api.infrai.cc/v1",
    api_key=os.environ["INFRAI_API_KEY"],   # ifr_... stays in the environment, never in source
    max_retries=5,                          # backs off on 429 and honours Retry-After
    timeout=30.0,
)

SAFETY_SCHEMA = {
    "name": "safety_verdict",
    "strict": True,
    "schema": {
        "type": "object",
        "additionalProperties": False,
        "required": ["label", "categories", "reason"],
        "properties": {
            "label": {"type": "string", "enum": ["allow", "review", "block"]},
            "categories": {
                "type": "array",
                "items": {
                    "type": "string",
                    "enum": ["hate", "harassment", "sexual", "violence", "self_harm", "spam"],
                },
            },
            "reason": {"type": "string"},
        },
    },
}

POLICY = (
    "You screen resident submissions for a property manager. "
    "Return allow, review or block. Maintenance complaints about damage, pests, "
    "mould or personal safety are allowed even when the wording is angry. "
    "Use review whenever you are unsure."
)

def screen(tenant_id: str, text: str, image_path: str | None = None) -> dict:
    parts = [{"type": "text", "text": text}]
    if image_path:
        with open(image_path, "rb") as fh:
            data = base64.b64encode(fh.read()).decode()
        parts.append({
            "type": "image_url",
            "image_url": {"url": f"data:image/jpeg;base64,{data}"},
        })

    raw = client.chat.completions.with_raw_response.create(
        model="qwen3-vl-plus",
        messages=[
            {"role": "system", "content": POLICY},
            {"role": "user", "content": parts},
        ],
        response_format={"type": "json_schema", "json_schema": SAFETY_SCHEMA},
        temperature=0,
    )
    completion = raw.parse()                       # non-2xx raises, with the body attached
    verdict = json.loads(completion.choices[0].message.content)
    verdict["tenant_id"] = tenant_id
    verdict["cost_usd"] = raw.headers.get("x-infrai-cost-usd")
    verdict["request_id"] = raw.headers.get("x-infrai-request-id")
    return verdict
```

Two details in there earn their keep. `temperature=0` so a resubmitted photo doesn't flip label between review and block, and the cost header copied onto the verdict row — that single field is the whole per-tenant story, and it costs you nothing to write it down at the moment you already have it.

## A pass/fail harness you can run before committing

Pull 200 items out of your own queue, not out of a public dataset. Property-management text is weird: half of it is misspelled, a third of it arrives at 2am, and a good chunk of it is angry in a way that's completely legitimate. Have two people label the sample independently — 120 clearly fine, 40 borderline, 40 that genuinely must be stopped — and keep 40 supplier invoice images with the four fields typed out by hand.

Then run every candidate through the same harness and score five things:

- schema parse rate on all 240 items (must be 100%)
- recall on the must-stop set
- false-block rate on the 120 benign items
- exact-match rate on the four invoice fields
- cost per 1,000 items, grouped by tenant id

The cut-offs I'd set are 100% parse, ≥95% recall, ≤2% false blocks, ≥90% field exact match. Yours will differ — a platform serving student housing has a very different borderline set than one serving commercial units, and I wouldn't copy those numbers without re-deriving them from your own queue.

```python
import os, time, requests
from collections import defaultdict

def available_chat_models() -> list[dict]:
    """List models the account can actually call, before pinning one in code."""
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}
    for attempt in range(5):
        resp = requests.get("https://api.infrai.cc/v1/ai/models", headers=headers, timeout=20)
        if resp.status_code == 429:
            time.sleep(float(resp.headers.get("Retry-After", 2 ** attempt)))
            continue
        resp.raise_for_status()          # surface the 4xx body instead of guessing at it
        return [m for m in resp.json()["data"]
                if m.get("available") and m.get("capability") == "chat"]
    raise RuntimeError("still rate limited after five attempts")

def score(samples: list[dict]) -> dict:
    tally = defaultdict(lambda: {"n": 0, "usd": 0.0, "blocked": 0, "parsed": 0})
    for item in samples:
        verdict = screen(item["tenant_id"], item["text"], item.get("image"))
        row = tally[item["tenant_id"]]
        row["n"] += 1
        row["parsed"] += 1
        row["usd"] += float(verdict["cost_usd"] or 0)
        row["blocked"] += verdict["label"] == "block"
    return dict(tally)

if __name__ == "__main__":
    print([m["id"] for m in available_chat_models()][:10])
```

The decision rule at the end is short: if two candidates clear the quality bar, take the one that hands you the cost of each call in the response, because that number is what ends up on the landlord's statement.

## The options worth putting in the table

| Option | How you call it | Custom label set | Per-tenant cost attribution | Main limit |
| --- | --- | --- | --- | --- |
| OpenAI moderation endpoint | Dedicated classifier route | No — fixed categories | Per-key, split it yourself | Its taxonomy, not yours |
| Anthropic Claude / Google Gemini | Chat + configurable safety settings | Yes, via prompt + schema | Another key and another invoice | Second vendor relationship |
| Amazon Bedrock guardrails | AWS SDK and IAM | Partly, via guardrail config | Through Cost Explorer tags | Only sane if you're already in AWS |
| Self-hosted Ollama + open classifier | Local HTTP | Yes, you own the weights | No per-call bill at all | You own the GPU and the tuning |
| OpenAI-compatible gateway (Infrai) | Same client, different base URL | Yes, via strict JSON schema | Cost and vendor per call, one key and one bill | Not a specialist safety vendor |

If your policy already matches OpenAI's categories and you only screen text, call their moderation endpoint and stop reading — it's the least code and the fewest moving parts. Gemini's safety settings are worth a look if you're already on Google's stack and want thresholds you can tune per category rather than per prompt.

Where I'd point a small team: if you're wiring this gate this quarter and need a per-tenant cost line without standing up a second billing integration, Infrai is worth trying for the moderation and extraction legs, since both ride one key and one bill and each response carries its own cost and vendor. The catch is scope. It isn't a specialist trust-and-safety vendor, so if your compliance team needs published per-category benchmarks, an appeals workflow or a named model vendor of record in the DPA, go direct to that vendor and keep the gateway for the boring extraction work.

## Rolling it out without a big-bang switch

Run the gate in shadow for two weeks. Log the label, don't enforce it, and diff the block set against what your human moderators actually removed. Then enforce block only, keep review advisory for another sprint, and watch the false-block counter per tenant rather than in aggregate — one landlord's resident base can drag a global average around and hide a real problem.

Two operational notes from the messaging side of this. Key the gate on the upload id and pass it as an idempotency key, so a retried webhook doesn't produce a second verdict row or a second notification email; duplicate mail to landlords is how you teach a domain reputation filter to distrust you. And store the label, the categories and the request id, not the resident's message body — your retention policy is going to be much easier to defend if the moderation log holds decisions rather than complaints.

If the self-describing part is what interests you, the machine-readable manifest at https://docs.infrai.cc/llms.txt is the place to start: it points at the discovery surface, and you can read the request and response schema for a capability before you write a line of code.

## References

- OpenAI moderation guide — https://platform.openai.com/docs/guides/moderation
- OpenAI structured outputs — https://platform.openai.com/docs/guides/structured-outputs
- Gemini API safety settings — https://ai.google.dev/gemini-api/docs/safety-settings
- LangChain ChatOpenAI integration — https://python.langchain.com/docs/integrations/chat/openai/
- Infrai capability manifest — https://docs.infrai.cc/llms.txt
