# Research: Pick the notification service

Resolves GitHub issue #2 ("Pick the notification service").

- **Date:** 2026-11-14
- **Question:** Which third-party service should Cadence use as the single source of truth for an in-app notification feed **and** browser web push?
- **Status:** Recommendation only. Final adoption is the user's call to confirm.

> This file follows no pre-existing repo convention — the repo had `docs/adr/` and
> `docs/agents/` but no research-notes location, so this introduces `docs/research/`.
> Every factual claim links to the primary source (official docs / pricing / npm registry)
> that owns it. Facts verified on 2026-11-14; versions/pricing can drift, so re-check the
> cited pages before relying on them.

## TL;DR recommendation

**Use Knock.** It is the only candidate that ships drop-in React in-app components with
explicit **seen + read + unread** state, pairs them with a fully-documented browser
web-push tutorial from the same source of truth, uses the simplest localhost/no-auth model
(token signing is *off by default* — identify a recipient by a raw user id), triggers
cleanly server-side from a Node SDK, and has by far the most generous free tier
(10k messages/mo, **unlimited recipients, channels, and workflows**). Email remains a
first-class channel, so nothing about this choice precludes adding email later.

**The one caveat that applies to all three:** none offers native raw-VAPID browser web
push. Every candidate routes browser push through **Firebase Cloud Messaging (FCM)**, so
"web push" means standing up a Firebase project + a `firebase-messaging-sw.js` service
worker + the client Firebase SDK regardless of which vendor you pick. The VAPID key is
real, but it is the Firebase Web Push certificate, used via the Firebase SDK.

## Requirements scorecard

| Requirement | Knock | Novu | Courier |
| --- | --- | --- | --- |
| Drop-in React in-app inbox/bell | Yes — `NotificationFeed`, `NotificationFeedPopover`, `NotificationIconButton` | Yes — single `<Inbox />` (bell + feed + prefs) | Yes — `CourierInbox` (popup or full-screen) + `useCourier` |
| Tracks read/unread/**seen** | read + unread + **seen** | read + unread (no "seen") | read + unread |
| Real-time delivery | Yes | Yes | Yes — one WebSocket per tab |
| Browser web push from same source | Via FCM; **dedicated step-by-step tutorial** | Via FCM/Pushpad/OneSignal; guide exists but has gaps | Via FCM provider; browser story least documented |
| Localhost / no-auth viable | Best — signing **off by default**, raw user id | Works in dev (plain `subscriberId`); **HMAC enforced in prod** | JWT client token minted server-side even in dev |
| Server-side trigger | Node SDK `workflows.trigger` | Node SDK `novu.trigger` | Send API |
| Free tier | 10k msgs; **unlimited recipients/channels/workflows** | 10k workflow runs; 5 workflows, 3 members, 24h retention | 10k sends/mo |
| Doesn't preclude email | Email is a channel | Email is a channel | Email is a channel |
| Self-hostable / OSS | No (closed SaaS) | **Yes (open source)** | No (closed SaaS) |

Current npm `latest` (npm registry, 2026-11-14): `@knocklabs/react` **0.15.1** (pub 2026-11-06),
`@novu/react` **3.19.2** (pub 2026-09-07), `@trycourier/courier-react` **9.3.5** (pub 2026-11-09).
Source: npm registry `dist-tags`/`time` for each package.

---

## Knock

### React in-app inbox
Knock ships three drop-in components under `@knocklabs/react`: `NotificationFeedPopover`
(managed popover list), `NotificationFeed` (full-page list), and `NotificationIconButton`
(bell showing the unread/unseen count). The popover pattern composes `KnockProvider` +
`KnockFeedProvider` around them. Knock tracks **seen, read, and unseen/unread** as distinct
states (the bell shows "the unread or unseen count").
Source: <https://docs.knock.app/in-app-ui/react/overview>

### Web push
Knock has **no first-party raw web-push channel**; browser web push goes through **FCM**.
The dedicated tutorial walks through: add FCM as a push channel in the Knock dashboard
(record its `channel_id`); install the `firebase` client package; set the Knock FCM channel
id, Firebase VAPID key, and Firebase config as env vars; add a `firebase-messaging-sw.js`
service worker in `public`; add a React `FirebaseProvider` (under `KnockProvider`) that
registers the SW; request permission and call Firebase `getToken` with the VAPID key +
SW registration; store the token on the recipient via `knock.user.setChannelData` on the
FCM channel. Workflows then fire an FCM channel step. This is the most complete
in-app-plus-web-push-from-one-source story of the three.
Sources: <https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm>,
<https://knock.app/blog/how-to-send-browser-push-notifications-from-nodejs>

### Server-side triggering
Workflows are triggered server-side with the Node SDK; the dashboard workflow fans out to
in-app + FCM steps for a recipient, so a single React Router action/loader call drives both
the feed and push. Source: <https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm>
(tutorial triggers a workflow with an FCM step).

### Localhost / no-auth
**Best fit.** "Enhanced security mode" is **off by default**; while off, the public API key
alone authenticates and Knock "will not reject requests that do not include a signed user
token." The client calls `knockClient.authenticate({ id: currentUser.id })` with just the
public key — exactly the dev-user-switcher model. Signing (RS256 JWT) is opt-in for
production, toggled per environment. Source: <https://docs.knock.app/in-app-ui/security-and-authentication>

### Free tier (Developer, $0, no card)
Up to **10,000 messages/month** (one message = one user on one channel); **no MAU cap**;
**unlimited** delivery channels, stored profiles, workflows, broadcasts, in-app guides, and
team members; Google SSO; 30-day log retention; 99.9% SLA; SOC 2 / GDPR / CCPA. In-app
guides (banners/popups) are the one metered sub-feature (500 guide recipients/mo). Gated
items are branding removal, i18n, SSO/SCIM, roles, analytics — none needed here. Knock
doesn't hard-cap; it contacts you on overage. Source: <https://knock.app/pricing>

---

## Novu

### React in-app inbox
Novu ships a single drop-in `<Inbox />` (bell + notification feed + preferences panel) in
`@novu/react`. You render it with your `applicationIdentifier` and `subscriberId`. It
manages the unread count and syncs read-state with the backend in **real time**. "Seen"
(as distinct from read/unread) is not called out.
Sources: <https://docs.novu.co/platform/inbox/overview>, <https://novu.co/comparison/magicbell>

### Web push
Push is delivered through a provider, not raw VAPID. Supported push providers: **FCM,
Expo, APNS, OneSignal, Push Webhook, Pusher Beams, Pushpad**. For the browser, the
documented path is FCM: create `firebase-messaging-sw.js`, generate a VAPID key pair in the
Firebase console (Web Push Certificates), call `getToken` with the VAPID key, then store the
device token on the subscriber via `subscribers.setCredentials` with the FCM provider id;
trigger a workflow with a Push step. Note: the FCM-web guide has rough edges (reads the
token from a `DEVICE_TOKEN` env var instead of posting the browser token; a service worker
using `import.meta.env`; Firebase 8.x scripts alongside a modern SDK).
Sources: <https://docs.novu.co/platform/integrations/push>,
<https://v0.x-docs.novu.co/guides/novu-fcm-web>

### Server-side triggering
Server-side `novu.trigger(workflowId, { subscriberId, payload })` from the Node SDK; a
single workflow can carry both an in-app step and a push step.
Source: <https://v0.x-docs.novu.co/guides/novu-fcm-web>

### Localhost / no-auth
Works in development with a plain `subscriberId` and the `applicationIdentifier` (HMAC off).
**But HMAC is mandatory in production** — Novu's API rejects unauthenticated Inbox requests
in production environments, so the server must compute `HMAC-SHA256(subscriberId)` and pass
`subscriberHash` to `<Inbox />`. For a localhost course project this is fine, but it means
the "no-auth" posture is dev-only and you must add signing before any real deployment.
Sources: <https://docs.novu.co/.mintlify/skills/inbox-integration/references/security.md>,
<https://docs.novu.co/platform/inbox/prepare-for-production.md>

### Free tier (Free, $0, no card)
**10,000 workflow runs/month**; unlimited subscribers; up to **3 team members**; all
channels (Email, In-app, SMS, Chat, Push — "Push" listed generically); **max 5 workflows**;
2 environments; 60 events/s; **24-hour** activity-feed retention. Branding removal and
longer retention are paid. Novu is also **open-source and self-hostable**, which is the main
reason to prefer it if vendor lock-in or self-hosting matters.
Sources: <https://novu.co/pricing>, toolradar listing (checked 2026-11-02) <https://toolradar.com/tools/novu>

---

## Courier

### React in-app inbox
Courier consolidated everything into one package, `@trycourier/courier-react`. It provides
`CourierInbox` (a drop-in inbox UI that renders as a popup or full-screen view), a
`useCourier` hook for direct inbox state/events, and theming via `CourierInboxTheme` /
`render*` props. Real-time updates run over a **single WebSocket per tab**, synced across
tabs and channels. The "new Courier Inbox for web" blog doesn't foreground a bell or
explicit seen/read semantics.
Sources: <https://www.courier.com/blog/the-new-courier-inbox-for-web-faster-more-flexible-in-app-notifications>,
<https://github.com/trycourier/courier-react>

### Web push
Courier's push providers are **APNs, FCM, Expo, OneSignal, Airship, Amazon SNS, Pusher
Beams, Pusher, MagicBell, Pushbullet, Beamer, NowPush**. The push docs do not foreground a
dedicated browser/web-push or VAPID guide; Courier's own SDKs register device tokens for
APNs and FCM, and the browser path is the least documented of the three candidates.
Browser web push would again run through FCM.
Source: <https://www.courier.com/docs/platform/channels/push/>

### Server-side triggering
Courier sends via its server-side Send API; a single send can target multiple channels
(email, SMS, push, Slack, in-app) and counts as one "send."
Source: <https://www.courier.com/pricing>

### Localhost / no-auth
Courier's inbox authenticates the browser with a **client JWT minted server-side** (scoped
to the user id) via Courier's auth token endpoint — more wiring than Knock's public-key
model even for local dev. (Courier docs; see auth/issue-token flow referenced from the
inbox SDK docs.)

### Free tier (Developer, $0)
**Up to 10,000 sends/month** (a multi-channel template delivery counts as one send); above
that you move to Business at **$0.005/send** pay-as-you-go. Courier states inbox components
and visual journeys are available on the free tier. A "Device Preview" add-on (from
$20/mo) is separate. Enterprise is custom (below $0.005/send).
Sources: <https://www.courier.com/pricing>,
<https://www.courier.com/blog/best-in-app-notification-centers>

---

## Decision and trade-offs

**Recommended: Knock.**

Why it wins for Cadence specifically:
1. **In-app feed quality.** All three have a drop-in inbox, but Knock exposes the richest
   state model (seen + read + unread) and a clean component split (bell / popover / page)
   that maps directly to a course-platform header bell + notifications page.
2. **Lowest localhost/no-auth friction.** Signing is off by default — authenticate with just
   a public key and the dev-user-switcher's user id. Novu forces HMAC in production (extra
   work before any deploy) and Courier needs a server-minted JWT even in dev.
3. **Most complete web-push-from-one-source tutorial.** Same FCM mechanics as the others,
   but Knock's is the most fully worked, with ready React provider/service-worker code.
4. **Most generous free tier.** Unlimited recipients, channels, and workflows vs Novu's
   5-workflow / 3-member / 24h-retention caps — more headroom for a course project.
5. **Email stays open.** Email is a first-class Knock channel; adding it later is adding a
   step to an existing workflow.

Trade-offs you are accepting:
- **FCM is unavoidable for web push (all three).** Browser push requires a Firebase project,
  a service worker, and the client Firebase SDK no matter which vendor you choose. There is
  no pure-VAPID, no-Firebase path among these candidates. Budget setup time for Firebase
  regardless of the pick.
- **Knock is closed-source SaaS.** If self-hosting or avoiding vendor lock-in is a hard
  requirement, **Novu** is the fallback — it's open-source and self-hostable, with a very
  similar `<Inbox />` + FCM-push model, at the cost of a thinner free tier, a rougher
  web-push guide, and mandatory production HMAC.
- **Courier** is a fine inbox but the weakest browser-web-push documentation and the most
  client-auth wiring, so it's third for this particular (localhost, no-auth, web-push)
  brief despite a solid real-time inbox.

### Other candidates noted
MagicBell, OneSignal, SuprSend, and NotificationAPI also play in this space (seen across the
comparison pages). OneSignal is push-first (weaker as an in-app feed source of truth);
MagicBell/SuprSend/NotificationAPI are plausible but were not required by the brief and
none clearly beats Knock on the combined (rich in-app feed + web push + generous free tier +
trivial localhost auth) axis. Worth a look only if the Knock/Novu trade-off above doesn't
settle it.
