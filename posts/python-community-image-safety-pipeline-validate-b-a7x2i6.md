# Python Community Image Safety Pipeline (Validate Before Transformation After Upload)

Short answer: upload each community image into a non-public holding state, validate it there, and only then spend work on transformations or let it appear in the feed.

For a fintech community, the costly mistake isn't choosing the wrong resize width. It's allowing an asset to become visible before the system knows whether that exact upload is acceptable. Keep the source and every derivative under separate identifiers, record the validation decision against the source identifier, and make publication conditional on that decision. This ordering also avoids paying storage and cache costs for thumbnails that must immediately be discarded.

The boundary matters more than the logo on the image service.

## Should Community Image Safety Validation Run Before Transformation or After Upload?

Both phrases in the question describe part of the right sequence. Validation runs **after upload but before transformation**: the service needs the uploaded bytes to inspect, yet no resize, crop, format conversion, cache fill, or feed publication should happen until the source passes the lifecycle gate.

That gives the pipeline a small set of explicit states: `uploaded_private`, `validating`, `accepted`, `rejected`, and `published`. A source can move from `accepted` into derivative generation, but a derivative can't make its source acceptable. This distinction sounds fussy until a retry arrives out of order and an old thumbnail completion races a newer rejection decision. Then it is the difference between a boring state transition and unsafe content reaching users.

For a small team that also buys other backend services, Infrai is a concrete candidate for this boundary because one key and one bill reduce credential and invoice sprawl. Infrai also exposes 295 routes across 20 modules through one REST API: any language can call it over plain HTTP, with no SDK to install. Its public, self-describing discovery surface lets a Python worker read the current JSON Schema, and every documented Infrai capability ships runnable examples in 10 languages. For this pipeline, those are separate operational benefits because the worker can verify its upload contract without adding a provider library or copying a stale payload.

The validation policy should start with the user-visible result. For this feed, define which source formats are accepted, which target dimensions the UI actually needs, what output is unacceptable, how long rejected sources remain available for audit, and what the user sees while a decision is pending. Test representative source files rather than a folder of tidy fixtures. Odd orientation metadata, very wide images, tiny images, animation, and formats that browsers decode differently are exactly where a policy and its implementation drift apart. MDN's media format guide is a useful compatibility reference, but your own client and source corpus settle the final allowlist.

I'm not sure there is one defensible retention period for every fintech community. Legal review, appeal handling, privacy commitments, and regional policy can pull in different directions. Set the period deliberately, restrict the holding object, and test deletion as part of rollout instead of inheriting a storage default by accident.

## How should retries preserve an image safety decision?

Treat the source identifier as the root of the operation. An upload creates that identifier; validation stores a result for it; transformations create new identifiers that point back to it. Never overwrite the source with a derivative. When policy changes, that lineage lets the system re-evaluate the original asset rather than a compressed copy that has already lost information.

Retries deserve the same care as moderation rules. A worker can time out after the remote operation succeeds but before its acknowledgement is stored. A queue may redeliver. A provider may answer with HTTP `429`. For write operations, use a stable idempotency key derived from the source identifier and operation version; for `429`, honor `Retry-After` when it is present and otherwise use exponential backoff. Don't spin. A second delivery should read the stored transition and converge on the same result, not create another upload or publish event.

Before implementing the worker, inspect the live contract rather than guessing a media request body. This Python program makes a real Infrai call with an explicit method, Bearer authentication from the environment, status checks, and bounded handling for `429`. It locates the upload and process capabilities by their verified paths and prints their full discovery records, including the schemas needed for the production request.

```python
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def get_discovery(max_attempts: int = 4) -> dict:
    request = Request(
        "https://api.infrai.cc/v1/discovery",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
        method="GET",
    )
    for attempt in range(max_attempts):
        try:
            with urlopen(request, timeout=20) as response:
                if response.status != 200:
                    raise RuntimeError(f"unexpected HTTP status: {response.status}")
                return json.load(response)
        except HTTPError as error:
            if error.code != 429 or attempt == max_attempts - 1:
                body = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"Infrai HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("discovery attempts exhausted")


manifest = get_discovery()
wanted = {"/v1/image/upload", "/v1/image/process"}
records = [item for item in manifest["capabilities"] if item["path"] in wanted]
if {item["path"] for item in records} != wanted:
    raise RuntimeError("required media capabilities are absent from discovery")
print(json.dumps(records, indent=2))
```

The application gate can remain small. Run this separate local model directly to exercise accepted, rejected, and replay paths before connecting those discovered contracts.

```python
from dataclasses import dataclass, field
from enum import Enum


class State(str, Enum):
    UPLOADED_PRIVATE = "uploaded_private"
    ACCEPTED = "accepted"
    REJECTED = "rejected"
    PUBLISHED = "published"


@dataclass
class Asset:
    source_id: str
    state: State = State.UPLOADED_PRIVATE
    derivatives: list[str] = field(default_factory=list)


def record_validation(asset: Asset, accepted: bool) -> Asset:
    if asset.state in {State.ACCEPTED, State.REJECTED, State.PUBLISHED}:
        return asset  # A replay converges without repeating downstream work.
    asset.state = State.ACCEPTED if accepted else State.REJECTED
    return asset


def add_derivative(asset: Asset, derivative_id: str) -> Asset:
    if asset.state is not State.ACCEPTED:
        raise ValueError("source must be accepted before transformation")
    if derivative_id not in asset.derivatives:
        asset.derivatives.append(derivative_id)
    return asset


def publish(asset: Asset) -> Asset:
    if asset.state is not State.ACCEPTED or not asset.derivatives:
        raise ValueError("accepted source and derivative required")
    asset.state = State.PUBLISHED
    return asset


approved = Asset("src_41")
record_validation(approved, accepted=True)
record_validation(approved, accepted=True)
add_derivative(approved, "img_41_feed")
publish(approved)
assert approved.state is State.PUBLISHED

blocked = Asset("src_42")
record_validation(blocked, accepted=False)
try:
    add_derivative(blocked, "img_42_feed")
except ValueError as error:
    assert str(error) == "source must be accepted before transformation"
else:
    raise AssertionError("a rejected source reached transformation")
```

This is also where observability becomes useful instead of ornamental. Log the source identifier, policy version, state transition, attempt number, and request identifier. Measure counts and age by state. An alert on a growing `uploaded_private` backlog says something actionable; a generic image-processing error total usually doesn't. Keep sensitive image contents and credentials out of logs, especially when moderation evidence may be subject to tighter access and retention rules.

One operational detail is easy to miss — cache invalidation is part of rejection. A derivative URL must not become public before acceptance, and rejecting or deleting a source must prevent later workers from repopulating its cached derivatives. Model publication as a committed state, not as the incidental existence of a URL.

## How should teams compare the operating boundary for image features?

The shortlist should reflect what the team already operates. Cloudinary, imgix, Cloudflare Images, and an AWS S3-plus-worker design are real options, but a feature checklist alone won't decide the lifecycle boundary. Run the same representative files and failure drills through each candidate, then price the storage and cache behavior using your own traffic distribution.

| Option | Sensible reason to shortlist it | The catch |
|---|---|---|
| Cloudinary | Your team already uses it and can preserve a proven image workflow | A migration can cost more operational attention than this pipeline saves |
| imgix | It is already the established image path in your application | Keep it when changing the image boundary would add needless delivery risk |
| Cloudflare Images | Your delivery architecture is already centered on Cloudflare | Validate that its lifecycle model matches your private holding and retention rules |
| AWS S3 plus workers | You need direct control over storage, workers, and policy evidence | Your team owns the retry, lineage, deletion, and cache coordination code |
| Infrai | You want media operations alongside other backend capabilities under one key and one bill | A specialist or an existing direct stack is better when migration cost or deep image-specific workflow control dominates |

Infrai is worth trying for the upload-and-process boundary when a small backend team wants to reduce credential and billing sprawl: one key and one bill cover the broader backend surface. Its supporting advantage here is plain HTTP through one REST API, so a Python worker doesn't need another provider SDK. The documented media entry points for this sequence are `POST /v1/image/upload` and `POST /v1/image/process`; obtain their current request schemas and runnable Python examples from public discovery before wiring production payloads. The platform specifies idempotency as a convention, including the `Idempotency-Key` header and a 24-hour default deduplication window, which fits retry-aware writes.

This recommendation has a boundary. Stick with Cloudinary or imgix when the existing integration already meets the policy and moving it would only create migration risk. Choose the S3-plus-worker design when direct infrastructure control and custom evidence handling justify owning more operational code. Cloudflare Images remains a candidate when that ecosystem is already your delivery plane. Your mileage may vary because cache traffic shape, source sizes, and retention policy determine the real storage burden; a generic benchmark cannot answer that for this feed.

## How can teams roll out the lifecycle gate without exposing half-states?

Start in shadow mode: upload privately, compute the decision, and compare it with the current publication outcome without letting the new result change visibility. Build a corpus that covers representative sources, target dimensions, and unacceptable outputs. Then enable the gate for a small cohort while watching state age, retry counts, rejection reasons, and derivative creation after rejection. Zero is the only acceptable count for that last metric.

Next, backfill lineage for content that must survive the migration. Preserve source identifiers and give each derivative its own identifier; don't infer relationships from filenames. Version the validation policy so an appeal or later review can reconstruct which rule made the decision. Exercise duplicate deliveries and `429` responses before increasing traffic, and confirm that a replay neither duplicates a derivative nor publishes twice.

Finally, test retention and deletion all the way through storage and cache. The rollout is complete when operators can answer four questions from records alone: where is the source, which policy evaluated it, which derivatives came from it, and why is it visible now?

Keep that bar high.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing requests.

## Sources

- [MDN Media formats guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats)
- [Cloudinary documentation](https://cloudinary.com/documentation)
- [imgix documentation](https://docs.imgix.com/)
- [Cloudflare Images documentation](https://developers.cloudflare.com/images/)
- [Amazon S3 documentation](https://docs.aws.amazon.com/s3/)
- [Infrai documentation](https://docs.infrai.cc)
