# Notifications v1: hosted services vs. in-house build on SQLite + Drizzle

Research date: **2026-11-12**. All pricing pages were read on 2026-11-12. Every claim links to the primary source it came from. Anything that could not be confirmed is marked **Unconfirmed**. This document lays out facts and trade-offs. It does **not** make the build/buy decision.

Cadence's requirements, used as the comparison frame:

- In-app bell/feed as the primary Channel.
- Email next, then push, SMS and other Channels.
- Every notification type can be opted out per recipient.
- Scheduled re-engagement reminders that have no triggering action (for example, "hasn't continued the course in N days").
- Four recipient roles.
- The free or low tier must cover current small scale.
- EU data handling is a comparison point, not a filter.

## Summary table

| | Knock | Novu (Cloud) | Courier | MagicBell | SuprSend | In-house (SQLite/Drizzle) |
|---|---|---|---|---|---|---|
| **Free tier** | 10,000 messages/mo | 10,000 workflow runs/mo | 10,000 sends/mo | 1,000 deliveries/mo | 10,000 notifications/mo | n/a (email provider's free tier applies) |
| **Billable unit** | 1 message = 1 user × 1 channel (in-app counts) | 1 workflow run per subscriber (any number of channels) | 1 send = 1 template × 1 recipient (multi-channel = 1) | 1 delivery per channel | 1 notification per channel per user | n/a |
| **First paid tier** | Starter $250/mo, 50k msgs, then $0.005 each | Pro from $30/mo, 30k+ runs, $1.20/1k extra | Business $0.005/send from the first send (no included volume) | Startup $249/mo, 50k deliveries, $0.0025 each extra | Essentials $100/mo, 50k (no preferences or digest; those start at Business $250/mo) | n/a |
| **React feed** | `@knocklabs/react`: prebuilt + hooks, WebSocket | `@novu/react`: `<Inbox/>` + hooks, WebSocket | `@trycourier/courier-react` v9: `<CourierInbox/>` + custom views, WSS | `@magicbell/react`: prebuilt (hooks in older packages) | `@suprsend/react` (drop-in) / `@suprsend/react-core` (headless) | Build it yourself: Drizzle table + loader + polling or SSE |
| **Delay / digest** | Delay, batch, throttle, wait-for-event | Delay, digest. **Max 1 day on Free**, 7 on Pro, 30 on Team | Delay (up to 93 days), batch, digest. Pricing page lists digest/batching under **Enterprise** | `wait` (seconds), pause, abort. **No digest found** | Delay, batch, digest, wait-until, time window, throttle. Batch/digest listed on **Business+** | Build it yourself |
| **Recurring schedule (no trigger)** | Yes: Schedules API (per-recipient, timezone-aware) | No recurring/cron trigger found; scheduled digests and delays only | Legacy Automations had a cron trigger. The current Journeys product lists only API and Segment triggers | Not found | Not found as a workflow trigger | In-process cron/interval job |
| **Cancel pending** | `cancellation_key` | `transactionId` cancel | `cancelation_token` / `run_id` | `abort` step | Cancel Workflow API | Delete or mark job rows |
| **Preferences** | Per workflow / category / channel type / channel. Hosted page. All plans | Global per channel + per workflow per channel, "critical" workflows. Inbox includes a preferences UI | Topics: OPTED_IN / OPTED_OUT / REQUIRED + per-topic channel routing. Hosted page | Channel × category. `UserPreferences` component | Global channel, category, channel-within-category, "can't unsubscribe". Hosted page. Listed on **Business+** | Build it yourself: a preferences table |
| **EU hosting** | **No.** "Expected by end of 2026". Subprocessors list US only | Yes, EU (Frankfurt), on the Free plan | Yes, EU datacenter on **Business+** | Region not stated. GDPR + DPA claimed | DPA: US or EU hosting region. Pricing page: custom region is Enterprise | Wherever the app runs |
| **Self-host** | None found | Yes: MIT core + commercial `enterprise/` code. Needs MongoDB + Redis + S3 | None found | None found | Enterprise only (BYOC in beta) | n/a |

## Per-vendor detail

### Knock

- **In-app feed (React):**
  - `@knocklabs/react` replaces the legacy `@knocklabs/react-notification-feed`. It ships prebuilt components (`NotificationFeedPopover`, `NotificationFeed`, `NotificationIconButton`) and React hooks for headless use. [docs](https://docs.knock.app/in-app-ui/react/overview)
  - Real time: clients connect to Knock's managed real-time service, which pushes new messages over a WebSocket. [feeds](https://docs.knock.app/in-app-ui/feeds/overview)
  - The feed API returns unread, read and seen counts for badges. Message states are unread, read, seen and archived. [feeds](https://docs.knock.app/in-app-ui/feeds/overview)
- **Channels:** email, SMS, push (APNs/FCM), chat (Slack, Teams, Discord), webhook, in-app feed and guides. [integrations](https://docs.knock.app/integrations/overview)
- **Scheduling and workflows:**
  - Workflow functions: batch (fixed or dynamic window), delay, throttle (cannot throttle across workflows), wait-for-event. [delay](https://docs.knock.app/designing-workflows/delay-function), [throttle](https://docs.knock.app/designing-workflows/throttle-function), [batch](https://docs.knock.app/send-notifications/designing-workflows/batch-function)
  - **Schedules:** "a managed, recipient-timezone-aware cron job". Supports recurring rules (`repeats`: frequency, days, hours, minutes in 5-minute steps) and one-off `scheduled_at`. Up to 100 recipients per create call. Exclusion dates are not supported. [schedules](https://docs.knock.app/concepts/schedules)
  - Cancel a run with a `cancellation_key`. Only runs paused in a batch, delay, wait or fetch step can be cancelled. Cancelling does not close an open batch window. Delivered in-app messages must be archived instead. [cancel](https://docs.knock.app/send-notifications/canceling-workflows)
- **Preferences:**
  - The PreferenceSet has `workflows`, `categories`, `channel_types` and `channels`, and these can be nested per workflow. Unset keys default to `true`. [preferences](https://docs.knock.app/preferences/overview)
  - There is a hosted preference page (signed URL) or a custom UI built on the API. No embeddable React preference component is mentioned on that page. [preferences](https://docs.knock.app/preferences/overview)
  - Preferences are included on all plans. [pricing](https://knock.app/pricing)
- **Pricing (read 2026-11-12, https://knock.app/pricing):**
  - Free: 10,000 messages/mo.
  - Starter: $250/mo with 50,000 messages, then $0.005 per message.
  - A "message" is one user on one channel. An in-app + email workflow counts as 2.
  - Messages suppressed by preferences or bounced do not count. Free is not hard-capped.
  - Feed retention is unlimited on all plans. Logs are kept 30 days on Free and Starter.
- **EU:**
  - "We expect to support EU data residency by the end of 2026." [plans](https://docs.knock.app/manage-your-account/knock-plans)
  - The subprocessors list (AWS, ClickHouse) shows only the USA. [subprocessors](https://knock.app/legal/subprocessors)
  - A DPA exists. [DPA](https://knock.app/legal/data-processing-addendum)
  - Knock's blog claims GDPR compliance and SOC 2 Type II. [blog](https://knock.app/blog/gdpr-announcement)
- **Self-host:** none found on the pricing or plans pages. Enterprise lists "dedicated infrastructure". [pricing](https://knock.app/pricing)
- **Lock-in:**
  - Workflows and templates are proprietary JSON. They can be pulled and pushed with `knock workflow pull/push` into a git-versioned `.knock/` directory. [CLI](https://docs.knock.app/cli/workflow/push), [CI/CD](https://docs.knock.app/developer-tools/integrating-into-cicd)
  - **Warehouse export is Enterprise-only.** [plans](https://docs.knock.app/manage-your-account/knock-plans)
  - Per-tenant preferences and translations are also Enterprise-only. [plans](https://docs.knock.app/manage-your-account/knock-plans)

### Novu

- **In-app feed (React):**
  - `@novu/react` provides a prebuilt `<Inbox />` or headless hooks (`useNotifications`, `useCounts`, `useNovu`). [headless](https://docs.novu.co/platform/inbox/headless-mode), [SDK](https://docs.novu.co/platform/sdks/react)
  - The hooks update from real-time WebSocket events such as `notifications.notification_received`. [useNotifications](https://docs.novu.co/platform/sdks/react/hooks/use-notifications)
  - Requires React 18 or 19. [SDK](https://docs.novu.co/platform/sdks/react)
  - Older v0.x docs use different imports, so watch the docs version. [v0 docs](https://v0.x-docs.novu.co/inbox/react/hooks/use-notifications)
- **Channels:** email, in-app, SMS, chat and push, all on Free. [pricing](https://novu.co/pricing)
- **Scheduling and workflows:**
  - Delay types: fixed, scheduled (subscriber timezone) and dynamic (from the payload). [delay](https://docs.novu.co/platform/workflow/add-and-configure-steps/configure-action-steps/delay)
  - **Max delay is 1 day on Free, 7 days on Pro and 30 days on Team.** [delay](https://docs.novu.co/platform/workflow/add-and-configure-steps/configure-action-steps/delay)
  - Digest types: regular and scheduled (every minute, hour, day, week or month, in the subscriber's timezone). The same 1/7/30-day plan caps apply. [digest](https://docs.novu.co/platform/workflow/add-and-configure-steps/configure-action-steps/digest)
  - The delay page says scheduled delays are one-time, not recurring. **No recurring or cron trigger was found** in the docs searched. A "no activity for N days" reminder would need an external scheduler that calls the trigger API.
  - Cancel pending delay or digest steps by `transactionId`. [cancel API](https://docs.novu.co/api-reference/events/trigger/delete)
  - Idempotency is supported. [idempotency](https://docs.novu.co/api-reference/idempotency)
- **Preferences:**
  - Global per-channel preferences, per-workflow per-channel preferences, and "critical" workflows that users cannot disable. [concepts](https://docs.novu.co/platform/concepts/preferences), [inbox prefs](https://docs.novu.co/platform/inbox/configuration/preferences)
  - **The docs conflict on precedence:** the concepts page and the Inbox page give opposite orders for global vs. per-workflow preferences. Test before relying on either.
  - The Inbox ships a preferences UI. It is available in both self-hosted and Cloud. [differences](https://docs.novu.co/community/self-hosted-and-novu-cloud)
- **Pricing (read 2026-11-12, https://novu.co/pricing):**
  - Free: 10,000 workflow runs/mo, up to 5 workflows, 3 team members, 24-hour activity retention.
  - Pro: from $30/mo with 30,000+ runs, 20 workflows and 7-day retention.
  - Team: from $250/mo.
  - Extra runs cost $1.20 per 1,000 on Pro and Team.
  - A run is one workflow for one subscriber, however many channels it uses. Runs filtered by preferences **are still billed**. Novu does not throttle over-limit usage. [billing](https://docs.novu.co/platform/account/billing)
  - **The 5-workflow cap on Free** could matter if each notification type is its own workflow.
- **EU:**
  - EU region in Frankfurt. The region is chosen at signup and data cannot be moved between regions. [security](https://docs.novu.co/platform/additional-resources/security)
  - EU residency is included on Free. [pricing](https://novu.co/pricing)
  - DPA/SCC template: [novu.co/dpa](https://novu.co/dpa).
  - A community thread reported that the DPA does not mention the EU region option. [thread, not primary](https://www.answeroverflow.com/m/1217100118751187065) **Unconfirmed** against the current DPA text.
- **Self-host:**
  - The README says the core is MIT, and the `enterprise/` folder "require[s] a commercial license". The repo contains LICENSE-MIT, LICENSE-ENTERPRISE and EE-PACKAGES-LICENSE. [repo](https://github.com/novuhq/novu)
  - **Self-hosted deployment is listed under the Enterprise plan** on the pricing page, separate from the free Community edition. [pricing](https://novu.co/pricing)
  - The Community edition lacks Inbox Snooze, social login and more than 2 environments. [differences](https://docs.novu.co/community/self-hosted-and-novu-cloud), [overview](https://docs.novu.co/community/self-hosting-novu/overview)
  - Infrastructure: MongoDB, Redis (AOF on the queue cluster) and S3-compatible storage. A single VM needs 4 vCPU / 8 GB. [overview](https://docs.novu.co/community/self-hosting-novu/overview)
  - **This conflicts with Cadence's one-SQLite-file decision.**
  - The exact text of LICENSE-ENTERPRISE was not fetched, so **Unconfirmed**.
- **Lock-in:** the hosted model plus an open-source core lowers exit cost. Workflows can be built in the dashboard or in code (`@novu/framework`). The bridge-endpoint mechanics were **not confirmed** in this pass.

### Courier

- **In-app feed (React):**
  - `@trycourier/courier-react` v9 (React 18+), or `@trycourier/courier-react-17`. Uses the `useCourier()` hook and a prebuilt `<CourierInbox />`. [inbox](https://www.courier.com/docs/platform/inbox/inbox-overview)
  - Customization comes in three levels: theme, custom views, or a fully custom UI. [inbox](https://www.courier.com/docs/platform/inbox/inbox-overview)
  - `@trycourier/react-inbox` and related packages are deprecated, so the API has changed recently. [inbox](https://www.courier.com/docs/platform/inbox/inbox-overview)
  - Real time uses a WSS endpoint (`wss://realtime.eu.courier.io` for EU). [EU](https://www.courier.com/docs/platform/workspaces/eu-datacenter)
- **Channels:** email, SMS, push, in-app inbox, and chat (Slack, Teams, WhatsApp). Provider routing with failover. [overview](https://www.courier.com/docs/overview), [routing](https://www.courier.com/docs/send/routing)
- **Scheduling and workflows:**
  - **Automations are "superseded"; new workflows are Journeys.** [journeys](https://www.courier.com/docs/platform/automations/overview)
  - Journey delays can be relative or absolute, up to 93 days. [journeys](https://www.courier.com/docs/platform/automations/overview)
  - Journeys have `batch` and `add-to-digest` nodes. [journeys](https://www.courier.com/docs/platform/automations/overview)
  - Cancel with `cancelation_token`, `run_id`, or a cancel node. [journeys](https://www.courier.com/docs/platform/automations/overview)
  - The Journeys page lists only API and Segment triggers. The legacy Automations product had a schedule trigger (one-time, recurring or cron, accurate to about 12 minutes). [scheduling](https://www.courier.com/docs/platform/automations/scheduling) It is **unclear** whether the current product has a recurring trigger.
  - **The pricing page lists "Digest and batching" under Enterprise.** [pricing](https://www.courier.com/pricing)
- **Preferences:**
  - Subscription topics with defaults OPTED_IN, OPTED_OUT or REQUIRED. Users cannot opt out of REQUIRED topics. [prefs](https://www.courier.com/docs/platform/preferences/preferences-overview)
  - Per-topic channel choice via `custom_routing`. A hosted preference page is available. [prefs](https://www.courier.com/docs/platform/preferences/preferences-overview)
  - Some custom-routing API writes can return `402`, which suggests plan gating. A global opt-out across tenants requires contacting support. [prefs](https://www.courier.com/docs/platform/preferences/preferences-overview)
- **Pricing (read 2026-11-12, https://www.courier.com/pricing):**
  - Developer: free, 10,000 sends/mo.
  - Business: $0.005 per send for every send, **including the first 10k** (50k sends = $250).
  - A send is one template to one recipient. Multiple channels count as one send.
  - Test environments count toward usage.
- **EU:**
  - Separate EU datacenter (`api.eu.courier.com`) on **Business and Enterprise**. Data is not shared between regions, and existing US data stays in the US. [EU](https://www.courier.com/docs/platform/workspaces/eu-datacenter)
  - The pricing page also lists EU Data Residency under Business. [pricing](https://www.courier.com/pricing)
- **Self-host:** none found.
- **Lock-in:** proprietary Journeys, templates and brands, and each region is set up separately. A data export path was **not checked**.

### MagicBell

- **In-app feed (React):**
  - The current docs install `@magicbell/react`, with subpath imports such as `@magicbell/react/floating-inbox`. [React](https://www.magicbell.com/docs/libraries/magicbell-react)
  - Components: `ContextProvider`, `FloatingInbox`, `Inbox`, `UserPreferences`, `ToastContainer`, `WebPushButton` and others. **The page does not mention hooks.** [React](https://www.magicbell.com/docs/libraries/magicbell-react)
  - The older packages `@magicbell/magicbell-react` and `@magicbell/react-headless` did have real-time hooks. [headless](https://magicbell.com/docs/libraries/react-headless) The package name changed, so this is a recent change.
  - The real-time transport is **not stated** on the current page.
- **Channels:** email (SES, Mailgun, SendGrid, SMTP), SMS (Twilio), mobile push (APNs, FCM, Expo), web push, Slack, Teams, in-app. [docs](https://www.magicbell.com/docs)
- **Scheduling and workflows:**
  - Steps: `wait` (seconds), `pause` (resume via API), `abort`, `broadcast`, and `if` conditions. "More commands coming soon." [workflows](https://magicbell.com/workflows)
  - **No digest step and no recurring schedule found.**
- **Preferences:** set per channel and category. [categories](https://community.magicbell.io/docs/categories) The `UserPreferences` component toggles channels per category. [React](https://www.magicbell.com/docs/libraries/magicbell-react)
- **Pricing (read 2026-11-12, https://www.magicbell.com/pricing):**
  - Builder: $0 with **1,000 deliveries/mo**, the smallest free tier here.
  - Startup: $249/mo with 50,000 deliveries, then $0.0025 each.
  - Deliveries count per channel (100 in-app + 50 email = 150).
- **EU:** the security page claims GDPR compliance and offers DPAs. **Hosting region is not stated** and no EU option is mentioned. [security](https://www.magicbell.com/security)
- **Self-host:** none found.
- **Lock-in:** proprietary categories, workflows and broadcasts. Export was **not checked**.

### SuprSend

- **In-app feed (React):**
  - `@suprsend/react` (drop-in) or `@suprsend/react-core` (headless hooks and providers). [overview](https://docs.suprsend.com/docs/react-inbox-integration)
  - `@suprsend/react-inbox` is deprecated, and auth moved from HMAC to signed JWT, so the API has changed recently. [migration](https://docs.suprsend.com/docs/react-migration-guide)
  - The real-time transport is **not confirmed**.
- **Channels:** email, SMS, WhatsApp, mobile and web push, Slack, MS Teams, inbox. [single-channel](https://docs.suprsend.com/docs/delivery-single-channel)
- **Scheduling and workflows:**
  - Nodes: Delay (fixed, dynamic, or a specific time), Batch, Digest (fixed recurring schedule, optionally per user), Wait Until, Time Window (user's local hours) and Throttle. [delay](https://docs.suprsend.com/docs/delay), [batch](https://docs.suprsend.com/docs/batch), [digest](https://docs.suprsend.com/docs/digest), [time window](https://docs.suprsend.com/docs/time-window), [throttle](https://docs.suprsend.com/docs/throttle)
  - Cancel Workflow API and idempotency keys. [cancel](https://docs.suprsend.com/reference/cancel-workflow), [idempotency](https://docs.suprsend.com/reference/idempotent-requests)
  - Scheduled sends exist for broadcasts, which are out of scope. [broadcast](https://docs.suprsend.com/docs/broadcast) **No per-user recurring workflow trigger was found.**
- **Preferences:**
  - Global channel opt-out, category opt-in/out, and `opt_out_channels` within a category. [prefs](https://docs.suprsend.com/docs/user-preferences)
  - A "Can't Unsubscribe" category setting with mandatory channels. [prefs](https://docs.suprsend.com/docs/user-preferences)
  - Auto-generated hosted preference page (`{{$hosted_preference_url}}`) and an embeddable React preference UI. [prefs](https://docs.suprsend.com/docs/user-preferences)
- **Pricing (read 2026-11-12, https://www.suprsend.com/pricing):**
  - Free: 10,000 notifications/mo with SuprSend branding in the inbox.
  - Essentials: $100/mo with 50k, then $2 per 1k.
  - Business: $250/mo with 50k, then $5 per 1k.
  - **User preferences and batching/digest are listed only from Business.**
  - Billing is per channel per user, and a digest counts as 1. Opt-out suppressions are not billed.
  - Free gives a 30-day premium trial, then reverts.
- **EU:**
  - The DPA says data is "stored in the hosting region of your workspace, the United States or the European Union". It also says staff may access data from India, and lists DPF and SCC transfer mechanisms. [DPA](https://suprsend.com/data-processing-addendum)
  - The pricing page lists "Custom Data Center Region" only under Enterprise. **Whether EU is selectable on Free or Essentials is unconfirmed.**
  - Note: "supersend.io" is a different company and does not show SuprSend's subprocessors.
- **Self-host:** Enterprise only. BYOC is in beta. [pricing](https://www.suprsend.com/pricing)
- **Lock-in:** proprietary workflows and categories. The docs include a migration guide *from* MagicBell. [guide](https://docs.suprsend.com/docs/migrate-from-magicbell-to-suprsend) Export was **not checked**.

### Other contenders considered

- **Engagespot:** `https://engagespot.co/` and `/pricing` returned **HTTP 404** on 2026-11-12, while `docs.engagespot.co` returned 200. No primary pricing source could be read, so it was not evaluated. Third-party listings (G2 and others) claim a free tier of 10k triggers/mo, but those are not primary sources. Flag this: the vendor's site status may have changed.
- **Customer.io:** not evaluated. It is oriented toward marketing automation, and broadcast/marketing is out of scope for v1.

## In-house build on SQLite + Drizzle (React Router)

Relevant facts about the current app:

- It runs as a long-lived Node server (`react-router-serve`) on `better-sqlite3` with `journal_mode = WAL` (see `app/db/index.ts`).
- WAL allows concurrent readers alongside **one writer at a time**, and all processes must be on the same host. [sqlite WAL](https://www.sqlite.org/wal.html)
- `lesson_progress` has `completed_at`, and `video_watch_events` has `created_at`. Either could serve as a "last activity" signal for reminders, but there is no per-enrollment `last_activity_at` column today.

### What would be needed

1. **Tables (Drizzle):**
   - `notifications`: id, recipientUserId, type, payload JSON, createdAt, seenAt, readAt, archivedAt, dedupeKey UNIQUE.
   - `notification_preferences`: userId, type, channel, enabled, with a unique index on (userId, type, channel). A missing row means the default.
   - `email_deliveries` or an outbox: notificationId, status, providerMessageId, attempts.
   - `scheduled_jobs`, or deriving reminders from a query: kind, runAt, key UNIQUE, lockedAt, doneAt.
2. **Badge and unread count:** `SELECT count(*) WHERE recipient=? AND read_at IS NULL` with an index on (recipient, read_at), computed in the root or layout loader.
3. **Live updates, options:**
   - (a) No push: React Router revalidates loaders after actions and navigation, and the client can revalidate on an interval.
   - (b) SSE from a resource route. A route that exports a `loader` but no default component is a resource route and can return a raw `Response`. [RR resource routes](https://reactrouter.com/how-to/resource-routes) The docs do not show an SSE/streaming example, so `text/event-stream` with a `ReadableStream` is standard Web API usage, not a documented React Router recipe.
   - With a single process, an in-memory event emitter can fan out to SSE connections. This breaks if the app ever runs more than one process.
4. **Email via a transactional provider (pricing read 2026-11-12):**
   - **Resend:** free 3,000/mo, **100/day cap**, 3 domains. Pro $20/mo for 50k. [pricing](https://resend.com/pricing)
   - **Postmark:** free 100/mo, never expires. Basic $15/mo from 10k. [pricing](https://postmarkapp.com/pricing)
   - **Amazon SES:** no SES-specific free allowance on the page, only general AWS Free Tier credits (up to $200, free plan for 6 months). $0.10 per 1,000 emails à la carte. [pricing](https://aws.amazon.com/ses/pricing/) The older "62k free/month" SES offer is not on the page today, so treat it as changed.
5. **Scheduler for reminders (single SQLite file, single process):**
   - Option A: an in-process timer (`setInterval` or a cron library) started from the server entry. It queries "enrolled, not completed, no activity in N days, no reminder sent for this window" and inserts notifications using a deterministic `dedupeKey` (for example `reminder:{userId}:{courseId}:{yyyy-ww}`) under a UNIQUE constraint. That makes reruns and restarts idempotent.
   - Option B: an OS or host cron calling a protected resource-route action.
   - Either way, keep writes short because SQLite allows one writer. Cancellation means deleting or marking pending `scheduled_jobs` rows, or simply re-checking the condition at send time.
6. **Opt-out enforcement:** one `shouldSend(userId, type, channel)` check in the single send path, covering both in-app and email. Optionally add "required" types that cannot be disabled (the hosted services offer this as REQUIRED / critical / can't-unsubscribe).
7. **Unsubscribe links and headers for email:**
   - **RFC 8058 one-click** [rfc8058](https://www.rfc-editor.org/rfc/rfc8058):
     - The `List-Unsubscribe` header MUST contain an HTTPS URI.
     - `List-Unsubscribe-Post: List-Unsubscribe=One-Click` is required.
     - Both headers MUST be covered by the DKIM signature.
     - The endpoint MUST NOT redirect. The POST carries no cookies or auth, so the URL needs an opaque, hard-to-forge token.
     - In React Router this maps to a resource-route `action`.
   - **Gmail** [guidelines](https://support.google.com/a/answer/81126), [FAQ](https://support.google.com/a/answer/14229414):
     - All senders need SPF or DKIM and a spam rate below 0.3%.
     - Bulk senders (about 5,000+/day to Gmail, and the status is **permanent** once reached) also need DMARC and one-click unsubscribe for marketing/subscribed mail.
     - "Transactional messages are excluded." Unsubscribes should be honored within 48 hours.
   - **Yahoo** [best practices](https://senders.yahooinc.com/best-practices/):
     - Bulk senders need SPF + DKIM + DMARC (p=none or stricter).
     - One-click unsubscribe for marketing/subscribed messages, plus a visible unsubscribe link in the body.
     - Honor unsubscribes within 2 days, and keep the spam rate below 0.3%.
   - Classification is not settled by the sources: re-engagement reminders are arguably "subscribed" rather than transactional, and the sources above do not define this precisely. Adding one-click headers regardless costs little.
   - The hosted services generate hosted preference or unsubscribe pages: Knock [prefs](https://docs.knock.app/preferences/overview), Courier [prefs](https://www.courier.com/docs/platform/preferences/preferences-overview), SuprSend [prefs](https://docs.suprsend.com/docs/user-preferences). Whether each one emits RFC 8058 headers automatically was **not verified**.

### Rough effort and complexity (estimate, not sourced)

| Piece | Complexity | Notes |
|---|---|---|
| Tables + migrations + seed | Low | 3–4 tables via the existing Drizzle/migration flow |
| Bell/feed UI + unread badge + mark read | Low–Med | shadcn/radix is already in the repo. Loader + actions |
| Live updates | Low (polling/revalidate) to Med (SSE + emitter, reconnect handling) | |
| Preferences UI + enforcement | Med | Types × channels matrix, defaults, required types, 4 roles |
| Email send + templates + retries/outbox | Med | Provider SDK, HTML templates, bounce/complaint webhooks |
| Unsubscribe (RFC 8058 + signed tokens + DKIM via provider) | Low–Med | |
| Reminder scheduler + idempotency | Med | Activity query, dedupe keys, restart safety |
| Digests/batching, push, SMS | Med–High each | Not needed for v1. This is where hosted services save the most work |

What a hosted service removes is mainly the scheduler/delay engine, digests, multi-channel provider plumbing, real-time transport and the prebuilt UI. What it adds:

- An external dependency and per-unit billing.
- User identity sync to the vendor.
- A data location outside the single SQLite file (relevant to the project decision).
- Proprietary workflow definitions.
