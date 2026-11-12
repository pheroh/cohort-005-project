# Research: options for outside Channels

Answers [#7](https://github.com/pheroh/cohort-005-project/issues/7) for the map [#4](https://github.com/pheroh/cohort-005-project/issues/4). This gives facts and trade-offs only. Choosing the v1 Channels belongs to #9.

Context: Cadence is a web-only React Router 8 app. It uses Drizzle and one SQLite file (better-sqlite3), and its users are international, including in the EU. The in-app bell is the main Channel. Every Notification type must be possible to turn off, and scheduled reminders are in scope.

Research date: **2026-11-12**. Prices are list prices read on that date and change often. Recheck them before you commit to a provider.

## Summary

| Channel | Reach for Cadence | Cost at small scale | Consent and legal load | Work in a React Router app |
|---|---|---|---|---|
| **Email** | Every user already has an address | $0 on free tiers (Resend 3,000/mo, capped at 100/day). About $15-20/mo for 10k-50k. SES is about $0.10 per 1,000 | Low for service Notifications. Re-engagement reminders are close to the marketing line, so give them an unsubscribe link and the one-click header | Low to medium. Provider SDK call, DNS (SPF/DKIM/DMARC), templates, an unsubscribe route, bounce webhook |
| **Web push** | Desktop Chrome, Edge, Firefox and Safari. iOS only from the Home Screen as an installed PWA (16.4+) | $0, because browser push services are free | Browser permission is the consent. If the user blocks, the site cannot ask again | Medium. Service worker, manifest, VAPID keys, a subscriptions table, `web-push` sender, handling expired subscriptions |
| **Mobile push (APNs/FCM)** | Not available. It needs a native or wrapped app, and Cadence has none | n/a | n/a | High. You would have to build an app first |
| **SMS** | Every user with a phone number, which Cadence doesn't collect today | About $0.012/msg in the US with carrier fees, about $0.056/msg in the UK, plus US 10DLC registration fees | High. US TCPA consent and STOP handling, carrier registration, and a phone number counts as personal data under GDPR | Medium to high. Number collection and verification, registration, inbound STOP webhook |
| **Slack / Discord** | Only users who use those tools and link an account | Free APIs | Low, but the user has to link an account | Medium. OAuth or bot install and account linking. A poor fit for a course app's Students |

## 1. Email

### Providers (prices as of 2026-11-12)

| Provider | Free tier | Cheapest paid | Notes | Source |
|---|---|---|---|---|
| Resend | 3,000 emails/mo, **100/day**, 3 domains | Pro $20/mo for 50,000/mo. Overage $0.90 per 1,000 | Page lists "Multi-region" sending and GDPR compliance on all plans. It does not state EU data residency | [resend.com/pricing](https://resend.com/pricing) |
| Postmark | 100 emails/mo with no overage. Does not expire | Basic $15/mo for 10,000. Overage $1.80 per 1,000 | Transactional and broadcast "Message Streams" are separate sending infrastructure | [postmarkapp.com/pricing](https://postmarkapp.com/pricing) |
| Amazon SES | No permanent free tier. New AWS accounts get up to $200 in credits (6-month free plan) | $0.10 per 1,000 (à la carte), with no monthly fee. Dedicated IP $24.95/mo | Cheapest at volume. You build more yourself: bounce and complaint handling via SNS, templates, a sandbox exit request | [aws.amazon.com/ses/pricing](https://aws.amazon.com/ses/pricing/) |
| SendGrid (Twilio) | **No longer has a permanent free plan.** New accounts since 2025-03-25 get a 60-day trial at 100/day. Existing free plans were retired from 2025-05-28 | Essentials about $19.95/mo for up to 50k (per Twilio's pricing summary, secondhand) | | [Twilio changelog 2025-05-28](https://www.twilio.com/en-us/changelog/changes-coming-to-sendgrid-s-free-plans), Twilio support article 35270136965403 (seen in search results; page unreachable) |

Note on scale: Resend's free tier caps sending at **100 emails a day**. A daily re-engagement reminder run, or a burst of Comment Notifications, could hit that cap before the 3,000/month limit.

### Deliverability rules (Gmail and Yahoo, enforced since Feb 2024)

- **All senders** must have SPF or DKIM, valid forward and reverse DNS, and TLS, must keep the spam rate in Postmaster Tools below 0.3% (Google suggests aiming for below 0.1%), and must follow RFC 5322. ([Google](https://support.google.com/a/answer/81126), [Yahoo](https://senders.yahooinc.com/best-practices/))
- **Bulk senders**, meaning "close to 5,000 messages or more to personal Gmail accounts within a 24-hour period" counted across the whole primary domain, must also have SPF and DKIM, a DMARC record (`p=none` is enough), and From: alignment. Once a sender reaches bulk status it is **permanent**. ([Google FAQ](https://support.google.com/a/answer/14229414))
- **One-click unsubscribe** is required of bulk senders for "marketing and subscribed messages". Google's FAQ says: "Transactional messages are excluded from this requirement." Its example of a transactional message is a password reset. Yahoo requires unsubscribes to be honored **within 2 days**. ([Google FAQ](https://support.google.com/a/answer/14229414), [Yahoo](https://senders.yahooinc.com/best-practices/))
- **RFC 8058 mechanics**: two headers, `List-Unsubscribe: <https://...>` and `List-Unsubscribe-Post: List-Unsubscribe=One-Click`. Both headers must be covered by the DKIM signature. The mail provider sends a POST with no cookies or auth, so the URL itself must carry a token that is hard to forge and identifies the user and list, and the endpoint must not redirect. ([RFC 8058](https://www.rfc-editor.org/rfc/rfc8058))

Open question for #9: whether a scheduled "you haven't continued this course" reminder is a "subscribed" message or a transactional one. The rules don't settle this. Adding the RFC 8058 headers anyway is cheap, and it also gives users the per-type opt-out the map already requires.

### Legal

- **CAN-SPAM (US)**: what counts is the email's "primary purpose". Transactional or relationship messages, such as notices about changes to an account or membership or delivery of a service the user agreed to, are exempt from most of the Act. They still must not use false or misleading header or routing information. The FTC reads the transactional categories narrowly. Commercial email must honor opt-outs **within 10 business days**, and the opt-out mechanism must keep working for **30 days** after sending. The penalty is up to **$53,088 per email**. ([FTC](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business))
- **EU/UK (ePrivacy Directive Art. 13 / UK PECR, plus GDPR)**: the consent and "soft opt-in" rules cover *direct marketing*. A message sent "for administrative or customer service purposes only" is a service message and falls outside those rules. If a message includes any promotional element, the ICO treats the whole message as marketing. In 2026 it fined a company that called its promotional texts "service updates". ([summary of ICO position and enforcement](https://www.mondaq.com/uk/privacy-protection/1233426/a-service-email-on-your-bike-says-the-ico), [ICO Jan 2026 fines](https://cy.ico.org.uk/about-the-ico/media-centre/news-and-blogs/2026/01/fines-of-225-000-for-nuisance-marketing-messages/). I could not reach the ICO's guidance page and EUR-Lex directly from this environment, so check them before relying on this.) Under GDPR, a provider that processes users' email addresses is a processor and needs a DPA. That is standard on all four providers above. Where the data is processed is a factor to compare, not a filter (map #4).
- For Cadence: Comment, Reply and Enrollment Notifications fall clearly on the service side. A "come back to your course" nudge is close to the line, and the copy must not upsell, for example by mentioning other courses.

### Work in a React Router app

- Send from server code (actions, or a job runner for scheduled reminders) through the provider's HTTP API or SDK. There is no client-side work.
- DNS setup for SPF, DKIM, DMARC and a sending subdomain.
- Templates, which can be React Email (Resend's ecosystem) or plain HTML.
- A one-click unsubscribe **resource route**: a POST that skips session auth and uses a signed token in the URL, as RFC 8058 requires. It writes to the same opt-out settings as the in-app settings page.
- A bounce and complaint webhook route that marks addresses as undeliverable, so you keep sending to bad addresses rarely and stay under the 0.3% spam rate.

## 2. Web push

### How it works

The page registers a **service worker**. `PushManager.subscribe({ userVisibleOnly: true, applicationServerKey: <VAPID public key> })` returns a `PushSubscription`, which holds an endpoint URL owned by the browser vendor's push service plus encryption keys. The server stores it and later POSTs an encrypted payload, signed with the VAPID private key, to that endpoint. The service worker's `push` handler calls `showNotification()`. The endpoint is a capability URL and must be kept secret. `pushsubscriptionchange` fires when a subscription expires. Push requires HTTPS. ([MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)) The browser vendors' push services cost nothing to use.

### Support (caniuse, read 2026-11-12)

- Chrome 50+, Edge 17+/79+, Firefox 44+, Samsung Internet, and Chrome for Android are supported. Safari on macOS has partial support from 16.1 and full support from 18.0. ([caniuse](https://caniuse.com/push-api))
- **iOS/iPadOS Safari: partial support from 16.4.** Push only works for a web app the user has **added to the Home Screen**. The site needs a manifest with `display: standalone` or `fullscreen`, and the permission request must come from a user gesture. No Apple Developer account is needed. ([WebKit blog](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)) In practice, most iPhone Students on Cadence can't get push unless they install the PWA.

### Permission UX

- Don't ask when the page loads. Ask at a moment when the value is obvious. The recommended pattern is a **double permission**: show your own in-app explanation first, and only trigger the browser prompt if the user accepts. If the user blocks, the site **cannot ask again**, and the user has to dig through browser settings to undo it. Provide a visible way to turn push off. ([web.dev](https://web.dev/articles/push-notifications-permissions-ux))
- Push fits best as an opt-in toggle in the Notification settings page.

### Work in a React Router app

- Add `public/sw.js` and a web app manifest (the manifest is needed for iOS). Register the service worker from `root.tsx`.
- Generate VAPID keys once and keep them in env vars.
- Add a `push_subscriptions` table (user, endpoint, p256dh, auth, created). A user can have many devices.
- Send with the `web-push` npm library, which handles VAPID signing and payload encryption (I could not reach github.com/web-push-libs/web-push from this environment to re-verify). Delete subscriptions when the push service returns 404 or 410.
- Delivery is best-effort with no read receipts. If you have both push and the bell, plan the copy and click target so the notification opens the matching feed item.

## 3. Mobile push (APNs / FCM)

Native push goes through Apple's APNs and Google's FCM to an installed app. **Cadence is web-only**, so this only applies after building or wrapping an app (Capacitor or React Native shell, Android TWA). On iOS, the only route that needs no app is web push from the Home Screen (above). Leave native push out of scope until an app exists.

## 4. SMS

### Cost (Twilio, read 2026-11-12)

- **US**: $0.0083 per segment plus carrier fees of $0.0035-$0.005 per message, so about $0.012-0.013 per message. A long code number costs $1.15/mo. ([Twilio US pricing](https://www.twilio.com/en-us/sms/pricing/us))
- **US A2P 10DLC registration is mandatory** for application SMS from 10-digit numbers, including for hobbyists. Unregistered traffic pays extra carrier fees and gets filtered. ([Twilio docs](https://www.twilio.com/docs/messaging/compliance/a2p-10dlc)) Fees, according to Twilio support pages found by search: brand registration $4.50 one-time for low-volume brands (raised 2025-08-01), about $46 for standard brands with secondary vetting, a **$15 campaign vetting fee** per campaign, and $1.50-$10 a month per campaign. ([Twilio support 1260803965530](https://support.twilio.com/hc/en-us/articles/1260803965530), [vetting FAQ](https://support.twilio.com/hc/en-us/articles/11587910480155-A2P-10DLC-Campaign-Vetting-FAQ)) Registration means manual review and takes time.
- **UK**: $0.056 per segment, and alphanumeric sender IDs are free and send-only. ([Twilio UK pricing](https://www.twilio.com/en-us/sms/pricing/gb)) Each country is priced and regulated separately, and many EU countries cost about the same as the UK or more.
- At small scale, one SMS costs roughly 100 times as much as one email on SES.

### Consent and legal

- **US TCPA**: automated texts to mobiles need prior express consent, and marketing texts need prior express *written* consent. FCC rules adopted in 2024 let consumers revoke consent **by any reasonable means**. Replies such as "stop", "quit", "end", "revoke", "opt out", "cancel" and "unsubscribe" count as revocation per se. A sender must honor a revocation within **10 business days**, and may not require the user to use one exclusive method. Parts of this were waived until 2026-04-11, and the sources disagree on which parts. ([Covington summary](https://www.covingtonblogs.com/2024/02/26/fcc-adopts-new-tcpa-consent-revocation-rules/), [Nixon Peabody on the waiver](https://nixonpeabody.com/insights/alerts/2025/04/11/fcc-partially-delays-new-tcpa-consent-revocation-rules). I couldn't reach the FCC and eCFR primary texts from here.) TCPA allows private lawsuits with statutory damages per message, which makes it the riskiest Channel legally.
- **EU/UK**: ePrivacy and PECR treat SMS the same way as email ("electronic mail"), so the service-versus-marketing distinction above applies. A phone number is also extra personal data to justify and protect under GDPR.

### Work

Cadence doesn't collect phone numbers today. You would need phone collection with OTP verification, per-country sender setup, US 10DLC registration, an inbound webhook that handles STOP/HELP keywords and syncs them to opt-out settings, and segment-aware message length.

## 5. Other: Slack, Discord and similar

- **Slack**: an incoming webhook posts to **one channel chosen at install** and can't change the channel, name or icon. ([Slack webhooks](https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks)) A per-user DM needs a bot token with `chat:write` and `chat.postMessage` with the user ID as `channel`. Each workspace has to install the app, and each Cadence user has to be mapped to a Slack user ID. The rate limit is about 1 message/sec per channel. ([chat.postMessage](https://docs.slack.dev/reference/methods/chat.postMessage))
- **Discord**: bots DM users via `POST /users/@me/channels` and then post a message. The usual expectation is that the bot shares a server with the user. I couldn't reach the official docs from here to confirm the rules on shared servers and spam.
- Fit for Cadence: this could make sense for **Team Admins** whose organisation uses Slack, or for a course community on Discord. It is a poor fit for individual Students. Each user has to set up an OAuth or account link, so few users will turn it on. This is usually a later addition rather than a v1 Channel.

## Cross-cutting notes for #9

- **Opt-out per type × Channel**: email (RFC 8058 POST), SMS (STOP keyword), and push (browser revoke, or a 410 from the push service) can each turn a user off outside the app. Whatever settings model you choose has to accept those writes.
- **Scheduler**: email and SMS sends for scheduled reminders need a job runner beside the React Router server. Inside a request, send work after the response is committed, or run it from a queue.
- **Build vs buy (#8)**: hosted notification services (Knock, Novu, Courier and others) wrap these same providers. This doc doesn't evaluate them.

## Not verified here

The ICO, EUR-Lex, eCFR/FCC, Discord docs, SendGrid support and GitHub were unreachable from this environment (DNS failure or 403). Claims that depend on them cite law-firm or third-party summaries and are marked as such above.
