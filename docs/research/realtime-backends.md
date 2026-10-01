# Research: cheap realtime backends for a shared synchronized timer

Charting research for the wayfinder map. Date: **2026-09-30**. All pricing is version-sensitive and changes without notice.

Question: which platform hosts the authoritative timer state, realtime sync to a handful of clients, auth, and a server-side trigger for push notifications — at tens of users and near-$0/month?

## Recommendation

1. **Supabase (free tier).** One platform does the whole job: Postgres as source of truth, Realtime (Broadcast + Presence + Postgres Changes), built-in Auth, and Edge Functions that call the Expo push service (official guide). $0, no credit card, comfortably inside quotas at this scale. *Gotcha:* free projects pause after ~1 week of inactivity — the honest price for always-on is Pro at $25/mo.
2. **Cloudflare Workers + Durable Objects** (or PartyKit pointed at your own Cloudflare account). A Durable Object holds the timer state; the **Alarms API** fires the server-side "timer ended" event with no polling; **WebSocket hibernation** keeps idle connections at ~zero duration cost; push is one outbound `fetch` to Expo/FCM. Runs on the free plan. You write more yourself (auth, presence, snapshot-on-reconnect) — the more educational pick if the goal is edge server logic.
3. **Firebase** — best native push path (FCM is free and handles APNs), but Cloud Functions require the Blaze plan (credit card), and Firestore-listener semantics make *server-authoritative time* a heavier protocol.

Skipped as primary: Ably (still needs a backend for state/auth; push likely wants the $29 Standard tier) and Pusher (tightest cap at 100 concurrent connections; two separate products). A bare WebSocket server on a ~$4.40/mo Hetzner VPS is cheapest but highest-ops and violates "minimal ops".

## Cost at target scale

Tens of users, several timers each (~50 users, ≤20 concurrent, well under 100K realtime messages/month).

| Option | Free tier (key limits) | Cost at this scale | Paid tier / gotchas |
|---|---|---|---|
| **Supabase** (Realtime + Edge Functions) | 2M msgs/mo, 200 peak realtime connections, 500K Edge Function invocations/mo, 500MB DB, 2 projects, 5GB egress | **$0** | Pro $25/mo; **free projects pause after ~1 week inactivity** |
| **Firebase** (Firestore + Cloud Functions + FCM) | Spark: Firestore 1GiB, 20K writes / 50K reads / 20K deletes per day; FCM free; **Cloud Functions require Blaze** | **$0 on Blaze** within quotas | Blaze needs a card (no minimum); Spark over-quota **shuts the product off for the month** |
| **Cloudflare** (Workers + Durable Objects) | Workers 100K req/day; DO free: 100K req/day (WS connection = 1 req; incoming WS msgs 20:1; outgoing free), 13,000 GB-s/day, SQLite 100K rows written/day, 5GB | **$0** | Paid $5/mo minimum only if you exceed free daily quotas |
| **PartyKit** | Individual: 10 live projects, **storage wiped every 24h**; Commercial: free on your own CF account | **$0** | Hosted free tier's 24h wipe would reset persisted state daily — self-deploy to CF |
| **Ably** | 200 concurrent connections, 500 msg/s, 6M msgs/mo | **$0** for realtime | Standard $29/mo; push on free tier not documented |
| **Pusher** (Channels + Beams) | Channels Sandbox: 200K msgs/day, **100 concurrent connections**; Beams Sandbox: 1,000 subscribers | **$0** | Startup $49/mo + $29/mo; two products |
| **Bare WebSocket on VPS** | none | **~$4.4–6/mo** | Highest ops: OS, TLS, monitoring, uptime, backups |

At this scale every provider's message/connection quotas are non-binding. The real constraints are **plan requirements** (Blaze card, 24h storage wipe, free-project pausing) and **where server-side timer logic + push triggering can live**.

## Realtime semantics (timer-relevant)

| Option | Broadcast & presence | Reconnect + resync | Server-authoritative time |
|---|---|---|---|
| Supabase | Built-in Broadcast + Presence + Postgres Changes | SDK auto-reconnects; **no event replay** — client re-reads the row from Postgres | Postgres `now()` / server timestamps; store absolute `endTime` |
| Firebase | Firestore snapshot listeners; **no native presence** (must build) | Auto-reconnect; offline persistence catches up; `serverTimestamp()` stamped server-side | `FieldValue.serverTimestamp()` |
| Cloudflare DO | Manual broadcast + presence | Manual handshake: client requests a snapshot on (re)connect; DO replies from SQLite | DO computes it from edge clock |
| PartyKit | `room.broadcast()`, `room.getConnections()` | Manual: `onConnect()` sends snapshot (documented pattern) | Party (DO) computes it |
| Ably | Channels + Presence + history API | **Automatic resume within ~2 min**; beyond that, history API | Server `timestamp` on every message |
| Pusher | Channels + Presence | Auto re-subscribe, **but missed messages are NOT replayed** — you fetch your own history | No native server timestamp |
| VPS | Manual | Manual | Your server clock |

## Push triggering

| Option | How |
|---|---|
| Supabase | Edge Function + DB webhook → Expo push service (official guide) or FCM/APNs; 500K invocations/mo free |
| Firebase | Cloud Function on Firestore write → **FCM** (free; delivers to Android + iOS via APNs); needs Blaze |
| Cloudflare / PartyKit | `alarm()` / `onRequest` at `endTime` → outbound `fetch` to Expo push service or FCM HTTP v1 |
| Ably | Native push product (publish to channel) |
| Pusher | Beams (separate product) |
| VPS | Your server calls Expo push service / FCM HTTP v1 |

## Auth, deploy, RN/Expo support, ops

| Option | Auth | Deploy | RN/Expo | Ops |
|---|---|---|---|---|
| Supabase | Built-in (50K MAU free) | Dashboard/CLI; SQL + functions together | Official RN/Expo quickstart + Expo push guide | Minimal (gotcha: pausing) |
| Firebase | Built-in (50K MAU free) | Console/CLI; Blaze linked to GCP | expo-notifications (dev build + FCM creds) or React Native Firebase | Minimal (Blaze card) |
| Cloudflare | None built-in — you write tokens | `wrangler deploy` | Raw WSS (RN native); no vendor SDK | Low–medium |
| PartyKit | None built-in (`onConnect` check) | `npx partykit deploy` | PartySocket or raw WSS | Low (24h wipe on hosted free) |
| Ably | Token auth (JWT from your backend) | SaaS | Official RN SDK + Expo guide | Low |
| Pusher | Channel auth via your server | SaaS | `pusher-websocket-react-native` | Low |
| VPS | You build it | ssh + systemd/Caddy | Any WS client | **High** |

## Key architecture takeaway

Per timer, the server persists an **absolute `endTime`** computed once at start; clients render `endTime − now` locally, so per-second message traffic disappears and "sync" degrades to **clock skew** rather than network jitter. Server owns authoritative state (start/pause/reset), broadcasts transitions, serves a **snapshot on (re)connect**, and at `endTime` fires the completion event + push trigger. The push path is orthogonal to the realtime path.

## Uncertain / version-sensitive

- All pricing as of 2026-09-30. Most volatile: Supabase free-project pausing, Pusher (now under Bird; docs at docs.bird.com), Cloudflare DO SQLite storage billing (announced to begin ~Jan 2026).
- Ably free-tier push inclusion inferred from "Build with any Ably product" wording — unverified.
- Latency figures are engineering estimates, not vendor SLAs.
- Pusher has no message replay and no server timestamp (confirmed by official guidance).
- No official Cloudflare/PartyKit RN SDK; compatibility rests on the standard WebSocket protocol.
- Supabase "re-read row on reconnect" is SDK-architecture inference, not a stated guarantee.

## Sources

Supabase: [pricing](https://supabase.com/pricing) · [realtime pricing](https://supabase.com/docs/guides/realtime/pricing.md) · [realtime](https://supabase.com/docs/guides/realtime) · [Expo push guide](https://supabase.com/docs/guides/functions/examples/push-notifications?platform=expo)
Firebase: [pricing](https://firebase.google.com/pricing) · [pricing plans](https://firebase.google.com/docs/projects/billing/firebase-pricing-plans) · [serverTimestamp](https://firebase.google.com/docs/firestore/manage-data/add-data#server_timestamp)
Cloudflare: [DO pricing](https://developers.cloudflare.com/durable-objects/platform/pricing/) · [Workers pricing](https://developers.cloudflare.com/workers/platform/pricing/) · [DO Alarms](https://developers.cloudflare.com/durable-objects/api/alarms/) · [WS hibernation](https://developers.cloudflare.com/durable-objects/best-practices/websockets/)
PartyKit: [pricing](https://www.partykit.io/) · [how it works](https://docs.partykit.io/how-partykit-works) · [deploy to Cloudflare](https://docs.partykit.io/guides/deploy-to-cloudflare)
Ably: [pricing](https://ably.com/pricing) · [connection states](https://ably.com/docs/connect/states) · [push](https://ably.com/docs/push) · [React Native](https://ably.com/docs/getting-started/react-native)
Pusher: [Channels pricing](https://pusher.com/channels/pricing/) · [Beams pricing](https://pusher.com/beams/pricing/) · [missed messages](https://docs.bird.com/pusher/channels/channels/events/how-can-i-get-missed-messages-after-reconnecting-to-channels) · [React Native client](https://pusher.com/docs/channels/getting_started/react-native)
Expo push: [overview](https://docs.expo.dev/push-notifications/overview/) · Hetzner: [CX plans](https://www.hetzner.com/pressroom/new-cx-plans/)
