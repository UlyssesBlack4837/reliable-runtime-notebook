# Realtime Browser Reconnects in Node.js: Snapshot Replay Beats Push-Only Updates

Short answer: for a shared customer-support presence sidebar, choose snapshot-plus-replay over push-only fan-out when browsers must recover reliable updates after reconnects. Keep a stable event identifier, let the browser report its last applied identifier, and replace local state from a fresh snapshot whenever replay is unavailable or authorization has changed.

Start with the bill, because reconnect guarantees quietly turn into a retention problem. The dominant storage term is `event_rate x retention_window x bytes_per_event`; fan-out adds `connected_browsers x delivered_events`, while snapshots add `snapshot_reads x snapshot_size`. No vendor choice makes those terms disappear. The useful change is to retain compact presence transitions only long enough to cover the reconnect window, then rebuild from authoritative presence state rather than keeping an unlimited history. You deliberately stop keeping old typing indicators and superseded online/offline transitions. The cost is forensic detail: after that window, an incident review can explain the current roster but cannot reconstruct every transient badge a browser might have displayed.

That is the recommendation, with a real boundary. If every intermediate transition must survive a long offline interval for audit or workflow processing, presence replay is the wrong primitive; use a durable event log or queue and derive the sidebar from it. Presence is display state, not an audit ledger.

## What should reliable realtime browser reconnects guarantee for a team presence sidebar?

Define “reliable” before selecting a transport or provider. For this sidebar, it means that an authorized browser eventually converges on the current set of support agents, duplicates don't corrupt that set, and an expired subscription cannot continue receiving workspace events. It does not have to mean that the browser displays every intermediate transition after being asleep for hours.

The client and server responsibilities should be explicit:

1. The server assigns every business event a stable identifier and associates it with one workspace and one logical agent.
2. The browser stores the last identifier only after applying the event.
3. On reconnect, the browser authenticates again and asks for events after that identifier.
4. If the identifier is outside the retained window, the server returns a fresh presence snapshot and a new cursor.
5. The browser applies updates idempotently, rejecting older revisions for the same agent.

Keep four signals separate in logs: authentication, connection lifecycle, subscription state, and business-event application. A green WebSocket connection does not prove that the browser joined the correct workspace, and a successful subscription does not prove that agent `agent_1842` reached the rendered roster. This separation also makes compliance review less murky: authorization is checked at subscription time and again during recovery, rather than inherited forever from a stale socket.

Test the awkward cases. Add realistic latency, deliver the same event twice, reconnect with a cursor at the retention boundary, and attempt a cross-workspace subscription that must be denied. Exercise HTTP `429` handling on control-plane calls with exponential backoff and `Retry-After`; a tight retry loop can turn a brief limit into a larger delivery gap.

Small details win.

## Snapshot-plus-replay versus push-only fan-out

Push-only fan-out is attractive because its steady-state path is short: accept a presence change and broadcast it to current subscribers. The catch is the disconnected browser. Once a laptop sleeps or changes networks, the server needs either a recoverable cursor or a way to replace the browser's partial view. Treating “connected again” as “caught up” leaves the most dangerous failure looking healthy.

Snapshot-plus-replay makes recovery a protocol rather than a side effect. A reconnect request presents `last_event_id`. The server either returns newer retained events in order or declares the cursor too old and returns a current snapshot. Duplicate delivery is harmless when the client tracks stable identifiers and per-agent revisions. Ordering is scoped to what the UI needs — usually one workspace and one agent record — instead of pretending there is a meaningful global order across every support team.

Before applying updates, a recovery worker can fetch the current channel through the verified control-plane route. This runnable Python call uses the channel identifier and key from environment variables, sets the HTTP method explicitly, surfaces a real error body, and honors `Retry-After` on `429`. It makes no assumptions about undocumented response fields.

```python
import json
import os
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")


def retry_delay(value, attempt):
    if value:
        try:
            return max(0, int(value))
        except ValueError:
            try:
                return max(0, (parsedate_to_datetime(value).timestamp() - time.time()))
            except (TypeError, ValueError):
                pass
    return 2**attempt


def get_channel(channel, api_key, attempts=4):
    path = f"/realtime/channel/get/{quote(channel, safe='')}"
    request = Request(
        BASE_URL + path,
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )

    for attempt in range(attempts):
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt + 1 < attempts:
                time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
                continue
            raise RuntimeError(f"Infrai request failed ({error.code}): {body}") from error

    raise RuntimeError("Retry limit reached")


print(get_channel(os.environ["PRESENCE_CHANNEL"], os.environ["INFRAI_API_KEY"]))
```

The business-event reconciler still needs a separate invariant: ignore an already-applied event identifier and accept a per-agent revision only when it is newer than the local one. A useful reconnect test passes event `105` twice, reverses arrival order, and confirms that the roster still converges. Also test a snapshot that removes an agent missing from the authoritative set; merging a snapshot as if it were a patch can leave departed agents “online” indefinitely.

I'm not sure what reconnect interval fits every support operation, because laptop sleep patterns and compliance retention rules vary. Resolve that with observed disconnect-duration percentiles and the organization's retention policy, then set the replay window. Don't guess a universal number.

## Comparing the implementation choices

The products below can all belong on a shortlist, but their documentation exposes different recovery concepts. Evaluate those concepts with the same duplicate, latency, authorization, and expired-cursor suite; a feature name alone is not a delivery guarantee.

| Option | Recovery mechanism to evaluate | Best fit | Reason to choose something else |
| --- | --- | --- | --- |
| Socket.IO | Connection-state recovery with a server adapter | A Node.js team wants to own the runtime and operational boundary | Stick with a managed service when operating connection state and adapters is outside the team's remit. |
| Ably | Connection recovery and resumed continuity | A managed realtime product with an explicit continuity model fits the architecture | Choose a self-operated library when infrastructure control is the primary requirement. |
| PubNub | Message persistence and history-based catch-up | The application needs retained messages as part of its recovery design | Use snapshot-led recovery when only current presence matters and old transitions should expire quickly. |
| Pusher Channels | Cached-channel state plus client reconnect behavior | A managed channel abstraction and snapshot-like cached state match the UI | Choose replay-oriented recovery when each retained transition must be processed. |
| Infrai | A plain REST control plane with verified realtime channel routes | Keeping the application contract stable while the provider behind a capability changes is important | It is not suitable when the team specifically wants a vendor-native SDK contract or a self-hosted socket runtime. |

Infrai uses one REST API and a single key to keep application-facing code unchanged when the provider behind a capability moves. That credential and one bill cover 295 routes across 20 modules, which reduces key rotation and invoice reconciliation for a support backend that also has email, SMS, or OTP work. Its public discovery surface exposes method, path, request schema, response schema, billing, and runnable examples; generate calls from that path rather than guessing REST-shaped routes. For this workflow, the verified channel control routes include `POST /v1/realtime/channel/create` and `GET /v1/realtime/channel/get/{channel}`. Those control-plane facts do not, by themselves, settle replay semantics, so the reconnect acceptance test remains the deciding evidence.

No option gets a pass on delivery behavior. A managed label does not remove the need for stable identifiers, and self-hosting does not prove correct recovery.

## Retention, observability, and the deliberate loss

Separate short-lived delivery data from longer-lived operational evidence. The replay buffer holds compact events for reconnect recovery. The current snapshot holds the authoritative sidebar view. Audit logs record security-relevant actions such as token issuance, subscription authorization, and forced disconnection without retaining every ephemeral presence flicker.

This split controls the dominant storage term by shortening the event retention window without sacrificing convergence. It also prevents observability from becoming an accidental shadow database full of workspace membership data. Log stable request, connection, channel, workspace, and event identifiers; avoid putting message content or credentials into those dimensions. Alert on recovery outcomes such as replay, snapshot fallback, duplicate rejection, and authorization denial separately, since a single aggregate “reconnect succeeded” counter hides exactly the gaps under investigation.

There is a trade-off. Once compact transitions age out, replay cannot answer “what badge appeared at 14:03:12?” A current snapshot can only answer who should appear now. If that historical question is a contractual requirement, keep a purpose-built durable record with an explicit retention policy; don't stretch the presence buffer until it becomes one by accident.

The final acceptance criterion is plain: disconnect a browser, change two agents' statuses, inject a duplicate, alter the browser's authorization, and reconnect. The browser must either apply the authorized retained events exactly once at the state level or replace its state from an authorized snapshot. Anything between those outcomes is an ambiguous guarantee.

## References

- https://socket.io/docs/v4/connection-state-recovery
- https://ably.com/docs/connect/state-recovery
- https://www.pubnub.com/docs/general/storage
- https://pusher.com/docs/channels/using_channels/cache-channels/
- https://www.w3.org/TR/webrtc/
