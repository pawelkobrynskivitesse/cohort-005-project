# Knock for Cadence in-app notifications

Researched: **2026-11-13**, Europe/Warsaw. Scope: documentation and source investigation for [Can Knock support a persistent in-app inbox for Cadence, and what constraints matter?](https://github.com/pawelkobrynskivitesse/cohort-005-project/issues/3), within [Find the specification for in-app notifications in Cadence](https://github.com/pawelkobrynskivitesse/cohort-005-project/issues/1).

## Finding and evidence limits

**Knock documents the capabilities needed for a persistent personal inbox:** per-user feed items, server-triggered creation, persisted engagement state, a feed API, and managed realtime delivery. It is a viable candidate; this investigation does **not** select it or approve any event, recipient, UI, retention, or cost policy. [Feeds](https://docs.knock.app/in-app-ui/feeds/overview), [API overview](https://docs.knock.app/in-app-ui/api-overview).

Facts below are first-party documentation/source observations. Paragraphs marked **Implication** or **Proposed integration** are engineering inferences for later decisions. No account was created, paid plan purchased, credentials provisioned, or integration executed. There was no runtime SSR, accessibility, offline, security, or load test. Live pages can change; source observations use JavaScript commit [07c61ac](https://github.com/knocklabs/javascript/tree/07c61ac3ef1c968731ea9b83d81193f254ed1761) and docs commit [e6a9c70](https://github.com/knocklabs/docs/tree/e6a9c70ed5ec09391cdbfd32dba32715316c641a). No retrieved source explicitly dated after the research date was used; crawler timestamps are not publication dates.

## Identity, recipients, and authorization

Knock users have a required stable ID; email, name, avatar, and phone are optional according to channel needs. Knock recommends an internal primary key rather than an email address. Identifying a user synchronizes their data to Knock. IDs are immutable. [Users](https://docs.knock.app/concepts/users).

Cadence currently stores an integer `users.id`, one platform `users.role` (student/instructor/admin), and separate team membership roles; the session returns a numeric current user ID. These are source observations at Cadence commit [979e285](https://github.com/pawelkobrynskivitesse/cohort-005-project/commit/979e285): [schema](https://github.com/pawelkobrynskivitesse/cohort-005-project/blob/979e285/app/db/schema.ts), [session](https://github.com/pawelkobrynskivitesse/cohort-005-project/blob/979e285/app/lib/session.ts). “Explore four roles” therefore does not imply four identity namespaces or a currently implemented many-role platform model.

**Implication:** map one person to one stable Knock user, such as the string form of their Cadence user ID. Student, Instructor, Team Admin, and Platform Admin are candidate reasons to receive an event, not automatically separate recipients. Resolve all relevant role relationships on the server and deduplicate the recipient set. Whether the same person should receive one or multiple items for one event remains a product rule.

Enhanced security is **off by default**: a public key can otherwise act as any user. Production should enable enhanced security and use a backend-signed RS256 JWT with the current user's `sub`; include expiry. With `explicit_grants: true`, grant only required capabilities such as `user/feed_read` and `user/feed_write`. Tokens support tenant restrictions; filtering a feed in the browser is not equivalent to granting access. The SDK offers `onUserTokenExpiring` for refresh. [Security and authentication](https://docs.knock.app/in-app-ui/security-and-authentication).

**Proposed integration:** an authenticated loader/resource endpoint derives identity from the Cadence session and issues the scoped user token. Keep signing and secret API keys server-side. A browser-submitted user ID must not determine the JWT subject. On dev user switching/logout, tear down the prior feed and state before showing another person's items. Always authorize the linked course, question, refund, or team resource again in its Cadence route; an inbox item is not an access grant. Access removal and previously sent sensitive content require an explicit policy.

## Persistence and engagement semantics

Feeds return messages generated ahead of time by workflows/broadcasts, in reverse chronological order, with counts and filters for engagement/custom properties. Message content and metadata cannot be updated after generation: archive and resend when content becomes stale. [Feeds](https://docs.knock.app/in-app-ui/feeds/overview).

Read/unread, seen/unseen, and archived/unarchived are separate SDK operations. Archival is reversible; it is not deletion. [JavaScript quick start](https://docs.knock.app/in-app-ui/javascript/sdk/quick-start). Feed queries expose cursor pagination, statuses, tenant/workflow/custom-data filters, archive inclusion, and unread/unseen counts. [Feed API](https://docs.knock.app/api-reference/users/feeds).

**Implication:** “personal item returnable within Cadence” fits the feed model. It does not settle whether opening the panel, seeing a row, clicking a target, or completing work marks it read. Nor does it settle badge meaning, archive UI, history duration, self-notifications, duplicate-role events, batching, or deleted targets. Built-in component defaults must be reviewed against those decisions.

## React, React Router, SSR, and UI

The supported React library is `@knocklabs/react`; the older `@knocklabs/react-notification-feed` is superseded. It provides a bell, popover feed, full-page feed, and headless hooks. [React overview](https://docs.knock.app/in-app-ui/react/overview). CSS variables/replacement CSS and custom item renderers are documented; the badge can use unread rather than unseen, and popover opening behavior is configurable. [Feed component](https://docs.knock.app/in-app-ui/react/feed). Fully custom UI can use the React hooks and client feed store. [Headless UI](https://docs.knock.app/in-app-ui/react/headless/feed).

Cadence declares React 19.2.7 and React Router 8.3.0. [Cadence package manifest](https://github.com/pawelkobrynskivitesse/cohort-005-project/blob/979e285/package.json). The inspected Knock React manifest (version 0.15.1 on its source branch, **not a verified latest npm release**) accepts React/React DOM 17–19 and exports generic, Next, and TanStack entry points; Next and TanStack are optional peers. [Pinned Knock manifest](https://github.com/knocklabs/javascript/blob/07c61ac3ef1c968731ea9b83d81193f254ed1761/packages/react/package.json).

**Proposed integration:** use generic React APIs inside the authenticated application layout, with the signed token from a server loader. Use React Router navigation for permitted internal destinations and select prebuilt/custom rendering after the UI review. No React Router-specific adapter or tested React Router 8 recipe was established. SSR compatibility is **unverified**: validate server import/render safety, hydration, identity changes, stylesheet ordering, loading/error/empty states, keyboard/focus behavior, pagination, and mobile behavior before calling the integration ready. Client mounting of the feed after hydration is a possible fallback, with a stable initial placeholder.

## Triggering, duplicates, and operational responsibilities

A workflow trigger takes recipients and data, with optional actor/tenant context; workflows produce the configured channel messages. [Trigger API guide](https://docs.knock.app/send-notifications/triggering-workflows/api). An inactive workflow rejects triggers, and recipient preferences can prevent delivery. [Trigger overview](https://docs.knock.app/send-notifications/triggering-workflows/overview).

Only the workflow trigger endpoint supports `Idempotency-Key`: up to 255 characters, default 24-hour window, identical parameters required, successful responses cached. Later retries can create new work after that window. [API idempotency](https://docs.knock.app/api-reference/overview/openapi). The Node SDK retries eligible connection/408/409/429/5xx failures twice by default with backoff; its default timeout is one minute and configurable. [Node SDK](https://github.com/knocklabs/knock-node#retries).

**Implication:** client SDKs do not determine Cadence's authorized recipients or guarantee that a committed database change reaches Knock. A concrete integration design must cover post-commit dispatch failures, durable pending work/replay if required, stable event identity, deduplication beyond the provider window, workflow version/configuration deployment, and timeout/error reporting. Avoid claiming exactly-once delivery. No new infrastructure is selected here.

Knock documents sending retries up to eight times over 30 minutes for retryable provider/pipeline failures. This is a delivery-pipeline policy, not a guarantee that a disconnected browser saw an item. [Delivery](https://docs.knock.app/send-notifications/delivering-notifications). Workflow debugging and activity show why runs produced or skipped messages. [Debugging](https://docs.knock.app/send-notifications/debugging-workflows). In-app delivery logs from the downstream-provider log endpoint are always empty; message events and workflow activity are the appropriate evidence. [Delivery logs API](https://docs.knock.app/api-reference/messages/list_delivery_logs).

## Realtime, disconnection, and consistency

Managed websocket delivery supplements the fetch API. [API overview](https://docs.knock.app/in-app-ui/api-overview). Optional inactive-tab disconnection/reconnection and experimental BroadcastChannel synchronization across tabs are documented. [Feed component](https://docs.knock.app/in-app-ui/react/feed).

The inspected feed source uses optimistic status changes and explicitly notes missing offline/rollback handling for archiving, plus races around bulk “all read/seen” updates. [Pinned feed implementation](https://github.com/knocklabs/javascript/blob/07c61ac3ef1c968731ea9b83d81193f254ed1761/packages/client/src/clients/feed/feed.ts). Socket source subscribes to new-message events; this inspection establishes no durable replay guarantee for missed socket events. [Pinned socket manager](https://github.com/knocklabs/javascript/blob/07c61ac3ef1c968731ea9b83d81193f254ed1761/packages/client/src/clients/feed/socket-manager.ts).

**Implication:** treat fetched server state as authoritative. Validate refetch/reconciliation after reconnect/focus, token refresh, failed read/archive writes, multiple tabs, and a new arrival during “mark all.” Offline mutation queues, guaranteed socket replay, cross-device timing, and exactly-once toast display were **not established** by this research. Specify acceptable behavior; do not infer it from “realtime.”

## Data lifecycle and costs

Message log retention and inbox retention differ. Standard message/workflow/audit/source/webhook logs retain 30 days, Enterprise 90; API logs retain 30 on all plans. Expired logs become inaccessible and are eventually pruned. **In-app FeedItems remain available indefinitely**, and their engagement statuses remain updateable after logs expire. Custom log retention is not currently supported. [Retention policy](https://docs.knock.app/manage-your-account/data-retention), [pinned owning text](https://github.com/knocklabs/docs/blob/e6a9c70ed5ec09391cdbfd32dba32715316c641a/content/manage-your-account/data-retention.mdx).

Deleting a Knock user is documented as permanent deletion of the user and associated data; references to a deleted actor can become a missing-user marker. [Delete user API](https://docs.knock.app/api-reference/users/delete), [Users](https://docs.knock.app/concepts/users). The channel bulk API includes `delete` and filtering, but exact deletion scope for old retained FeedItems versus expired logs should be validated before promising a purge deadline. [Channel bulk actions](https://docs.knock.app/api-reference/channels).

**Implication:** Knock receives the identities/profile properties, event data, and rendered content sent to it. For in-app only, email/phone need not be supplied. Define an allowlisted minimal payload and safe internal target reference rather than copying entire Cadence records. Archiving cannot satisfy erasure. Decide how removed access, removed content, deleted actors, and account deletion affect already delivered items; verify any required deletion/export guarantees with the provider.

| Current advertised plan | Relevant terms |
| --- | --- |
| Free | $0; up to 10,000 messages/month; Knock branding required. |
| Starter | $250/month; 50,000 messages included; additional messages $0.005 each; branding removable. |
| Enterprise | Custom pricing; translations, per-tenant branding/preferences, observability extensions, and additional controls. |

A message is one delivery to one recipient on one channel; failures/bounces are excluded. All plans advertise unlimited stored profiles, workflows, feed retention, and custom feed UI/filtering. Illustrative Starter arithmetic: 100,000 messages/month costs $500 before any unlisted taxes/contract terms. This is not a Cadence volume estimate. [Pricing](https://knock.app/pricing).

Tenant API use is available on non-Enterprise plans, while tenant-specific preferences/branding are Enterprise features. In-app channels have no sandbox mode, so development workflow conditions/environment separation matter. EU residency is described as **coming by the end of 2026**, not as currently available; treat that roadmap statement as uncommitted until verified. [Plans and usage](https://docs.knock.app/manage-your-account/knock-plans).

## Inputs for the remaining decisions

Knock exposes no documented core capability blocker for the agreed working Notification concept. A provider selection still needs the approved scenarios and recipients; chosen read/badge/archive behavior; reviewed UI; acceptable dispatch/reconciliation behavior; minimal shared data/deletion policy; and expected monthly recipient deliveries. No requirement currently establishes a need for an alternative survey. Reopen comparison if, for example, branding at the acceptable budget, required residency, content mutation, log history, or verified offline behavior exposes a concrete mismatch. Existing email remains separate; no production feature, schema, or application code was changed.
