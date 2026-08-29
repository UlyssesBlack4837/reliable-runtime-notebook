# Template Governance for a Cheap Beginner 2FA Login Across US/EU SMS and Email

A marketplace has two messages that look similar but carry very different risk: the OTP that lets a seller into an account and the new-order alert that asks the seller to act. **Short answer: keep both templates and the OTP state in the application, use SMS as the primary login channel, make email an explicit fallback, and treat polled delivery events as operational evidence rather than authentication proof.** This is a low-complexity starting architecture for a beginner team serving the US and EU; it is not a claim that one transport is always cheapest or best.

Template ownership changes the answer. If meaning lives inside a provider dashboard, a channel migration can change login copy, order fields, and compliance text without a normal code review. If every byte lives in one shared string, though, SMS segmentation and email rendering become easy to break. The useful boundary is between application-owned meaning and channel-owned presentation.

This ADR chooses that boundary.

## How can a cheap beginner US/EU 2FA login govern SMS and email templates?

The application owns versioned template definitions, locale selection, required variables, and the policy that decides which template may be sent. A small channel adapter owns the final SMS or email rendering and submits the result through a generic delivery interface. Provider dashboards may be useful for emergency operational controls, but they are not the source of truth for message copy.

That split keeps the seller's order number, shop name, and sign-in purpose under the same deployment and review process as the feature that created them. It also lets the adapter enforce channel constraints without teaching the order service about encodings, HTML, or transport-specific metadata.

“Cheap” means few moving parts here. It doesn't mean choosing by a temporary per-message price. A beginner implementation can start with one template registry, one challenge table, one outbox, and two delivery adapters. It can still preserve an exit path because business meaning is not trapped in an external editor.

**The login flow follows from that ownership decision.**

Start with one application challenge per login attempt. The record should identify the account, purpose, selected channel, template version, expiry policy, remaining attempt policy, and current state. Keep secrets and raw OTP values out of ordinary logs. The browser receives an opaque challenge identifier, never delivery credentials.

Send SMS first only if that is the product's declared primary path. Do not fire an email at the same time. A visible “use email instead” action is easier to explain, rate-limit, audit, and support than two codes racing toward the same person. When the seller chooses fallback, make the transition atomic: supersede the SMS path and activate an email path under the same login attempt. The email is a fallback transport, not a second independent authentication ceremony.

One active path.

Consider the awkward case before the happy one. A seller requests an SMS code while checking a new-order alert, waits, switches to email, and then receives the delayed SMS. If both codes remain valid, the UI says email while the phone presents a plausible alternative; support cannot state which code should work, and the application has enlarged the number of credentials in play. The state transition should therefore invalidate or supersede the earlier channel according to one documented policy, share the attempt budget across the transition, and reject a code tied to an inactive template version. A later delivery event can explain the timing, but it cannot reactivate that code. This single edge case crosses database transactions, resend behavior, customer support language, and abuse controls, so it deserves a test fixture rather than a sentence in a runbook.

Polling belongs off the verification decision. A bounded worker may poll outstanding delivery records, store the latest transport state, and stop at a terminal result or deadline. The login endpoint verifies the code against application challenge state; it does not infer identity from a “delivered” status. Polling on every browser refresh couples authentication latency to a messaging system and makes an innocent reload produce more downstream traffic.

Don't poll forever.

For SMS, test the exact localized text rather than its character count in an editor. GSM-7 and UCS-2 have different single-message and multipart limits, and concatenated messages reserve space for segmentation metadata. A curly quote, non-Latin seller name, or localized currency may change the encoding and segment count. That is why the semantic template can accept a shop name while the SMS adapter still needs an explicit length and encoding policy. The linked SMS segmentation reference documents the limits; the architectural point is to test rendered fixtures for every supported locale.

For email, authenticate the sending domain and keep OTP mail narrowly transactional. Google's sender guidelines describe SPF, DKIM, DMARC, TLS, alignment, and subscription-message requirements at different sending volumes. The exact obligations depend on how the marketplace sends and at what volume, so I'm not sure a generic checklist can settle compliance for a particular US/EU deployment. Legal review, current regional rules, and the receiving domains' published requirements resolve that question better than an ADR does.


## Security and reliability invariants

The governing invariants make the boundary enforceable.

The first invariant is simple: transport acceptance is not login success. Only verification of an active challenge can authenticate the seller. Likewise, delivery of a new-order alert is not proof that the seller read or fulfilled the order.

Second, template version is data. Persist it with the challenge or outbox item so a retry renders the same approved meaning after a deployment. A resend policy must say whether it reuses a challenge, replaces it, or refuses the request; it cannot create an unrelated valid code by accident. Idempotency belongs at the application boundary as well, since a caller may lose a response after the send was already accepted.

Third, authentication and commerce notifications have separate rate budgets and urgency rules. A burst of marketplace orders should not consume the capacity reserved for seller login, while repeated login requests should not suppress legitimate order alerts. Apply limits by account, destination, IP risk, purpose, and time window as appropriate to the threat model. A transport-level HTTP 429 is useful backpressure, but it arrives too late to serve as the abuse policy.

The outbox is the main failure boundary. Commit the business event and its notification intent together, then let a worker render and send. For an order alert, this prevents a database commit from succeeding while an in-process send disappears during shutdown. For an OTP, latency is tighter, but the challenge still needs a durable state transition before delivery is requested. Retries read the persisted template version and idempotency key; they don't reconstruct intent from a log line.

Observability should answer four different questions: did the application create the intended message, did a transport accept it, what delivery state was last observed, and did the user complete the business action? Combining these into one “success” metric hides the exact boundary operators need. Log identifiers, template versions, channels, transitions, and coarse destination geography when justified, while redacting codes, credentials, message bodies, and unnecessary personal data.

## Cost and operations comparison

The practical cost model is broader than message price: engineering time for two adapters, polling traffic, multipart SMS, support contacts, compliance work, and migration effort all count. Your mileage may vary by destination mix. Measure those inputs with production-safe aggregates before optimizing a unit price that may not dominate the total.

| Ownership option | What it simplifies | Failure boundary | Best fit |
|---|---|---|---|
| Application-owned templates | Review, versioning, tests, and provider changes | The team must build rendering and localization discipline | Small systems where auth and order semantics change with product code |
| Provider-owned templates | Non-code editing and provider-native tooling | Copy, variables, and approvals can drift from an application release | Organizations with a dedicated messaging operations workflow |
| Hybrid ownership | Channel specialists can tune presentation while the app controls meaning | Two sources of truth need an explicit contract | Mature teams with separate product and deliverability owners |

The hybrid option is tempting, but “hybrid” must name a boundary. In this design, the application owns the template ID, semantic fields, locale, and version. An adapter may own a channel-specific subject line, whitespace, and encoding-safe rendering. It may not decide that a new-order alert is an OTP, add a new variable, or silently change the authentication purpose.

## Contract tests at the channel boundary

The following code models the internal contract, not a commercial API. It makes ownership visible: product code selects a semantic template and supplies validated fields; the registry renders a channel-specific message; the delivery port receives an idempotency key. Real implementations should use a cryptographically secure OTP generator, store an appropriate verifier rather than the raw code, and enforce challenge policies in a transactional store.

```python
from dataclasses import dataclass
from enum import Enum
from typing import Mapping, Protocol


class Channel(str, Enum):
    SMS = "sms"
    EMAIL = "email"


@dataclass(frozen=True)
class RenderedMessage:
    channel: Channel
    destination: str
    subject: str | None
    body: str
    template_version: str


class DeliveryPort(Protocol):
    def send(self, message: RenderedMessage, idempotency_key: str) -> str:
        """Return an opaque delivery ID."""


class TemplateRegistry:
    VERSION = "seller-messages-7"

    def render(
        self,
        template: str,
        channel: Channel,
        destination: str,
        fields: Mapping[str, str],
    ) -> RenderedMessage:
        if template == "seller_login_otp":
            code = fields["code"]
            if channel is Channel.SMS:
                return RenderedMessage(
                    channel, destination, None,
                    f"Marketplace sign-in code: {code}", self.VERSION
                )
            return RenderedMessage(
                channel, destination, "Your marketplace sign-in code",
                f"Use {code} to finish signing in. If you did not request it, ignore this email.",
                self.VERSION,
            )

        if template == "seller_new_order" and channel is Channel.EMAIL:
            order_id = fields["order_id"]
            shop_name = fields["shop_name"]
            return RenderedMessage(
                channel, destination, f"New order {order_id}",
                f"{shop_name} received order {order_id}. Open the marketplace to review it.",
                self.VERSION,
            )

        raise ValueError(f"unsupported template and channel: {template}/{channel.value}")


def send_login_code(
    delivery: DeliveryPort,
    registry: TemplateRegistry,
    challenge_id: str,
    channel: Channel,
    destination: str,
    code: str,
) -> str:
    message = registry.render(
        "seller_login_otp", channel, destination, {"code": code}
    )
    return delivery.send(message, idempotency_key=f"otp:{challenge_id}:{channel.value}")
```

There is deliberately no provider URL in this boundary. Each adapter can translate `RenderedMessage` to a verified external contract, while tests can assert the chosen template version, fallback transition, redaction, and idempotency behavior without making a network call. Add fixture tests for characters that change SMS encoding, missing fields, stale template versions, late delivery events, duplicate sends, and the SMS-to-email transition.

## Portability and vendor migration plan

This ADR rejects provider-owned templates as the initial default because template meaning is tightly coupled to authentication state and marketplace order data. It also rejects simultaneous SMS and email OTP delivery: that path creates ambiguous challenge ownership and spends operational capacity before the user asks for recovery. Webhooks are not assumed; polling is a bounded adapter capability because event delivery support varies by transport contract.

The catch is that application ownership is not suitable when a regulated communications team must approve and publish copy independently of software releases, when marketers need frequent visual email editing, or when a provider-managed authentication product is expected to own challenge generation and verification end to end. In those cases, stick with provider-owned templates or a tightly specified hybrid, then mirror template IDs and versions into deployment review so application behavior remains auditable.

A migration should start from evidence, not preference. Count how often copy changes outside software releases, how many locales require separate approval, how many incidents involve renderer drift, and how much adapter work is repeated across teams. Move presentation ownership only when those signals show that the current owner has become the bottleneck; preserve semantic fields and template versions as an explicit cross-system contract during the move.

A push-event design is valid when the selected transport offers authenticated, replay-resistant event delivery and the team can operate signature verification, deduplication, ordering tolerance, and dead-letter handling. A managed orchestration layer is valid when many channels, countries, routing rules, and compliance workflows outweigh the value of a small in-house state machine. Neither choice is a universal upgrade. The migration decision turns on who must change message meaning, who must approve it, and which failure boundary the team can actually operate.

For the beginner marketplace, keep the first system legible: product-owned semantics, channel-aware renderers, one active OTP path, explicit email fallback, and bounded event polling. Revisit the ADR when template editors, locales, channels, or regulatory ownership become separate organizational concerns.

## References

- https://support.google.com/a/answer/81126
- https://www.twilio.com/docs/glossary/what-sms-character-limit
