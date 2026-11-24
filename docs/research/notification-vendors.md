# Notification vendors for Cadence in-app Notifications

Researched 2026-11-24. Package versions come from the npm registry on that date (`npm view`). Pricing pages were read on the same date and can change.

## Question

Which third-party vendor should back Cadence's in-app Notifications? Cadence runs React Router 8, React 19, SQLite/Drizzle and a Node server. It has no real auth: a dev-only panel switches the current user on the client.

The criteria, from the planning map:

- a free tier big enough for development
- a React inbox component and a headless API or hooks
- a TypeScript server SDK
- server-side sending that doesn't need incoming webhooks
- room to add email later
- a story for local development and tests
- self-hosting is a plus, not a requirement

User identity has to survive client-side user switching. The obvious pattern is a signed user token that the server issues on each request.

## Summary: top two

1. **Knock** is the best fit.
   - **Auth:** Server-signed RS256 user tokens come from `signUserToken` in `@knocklabs/node`. The client can call `authenticate` again with a new token at any time, which fits the user-switching panel directly.
   - **Free tier:** 10,000 messages a month and unlimited environments.
   - **Testing:** It has the strongest built-in story. `sandbox_mode` and `skip_delay` trigger settings generate messages without delivering them, and each environment has its own keys and data.
   - **React:** `@knocklabs/react` declares React 19 support and ships both components and hooks (`useNotifications`, `useNotificationStore`).
   - **Weaknesses:** It can't be self-hosted, and the first paid tier costs $250 a month.
2. **Novu** is the runner-up.
   - **Self-hosting:** It is the only option that is open source (MIT core) and self-hostable with Docker Compose. That gives a no-network or offline local and CI path if Cadence ever wants one.
   - **Free tier:** 10,000 workflow runs a month, with Dev and Prod environments.
   - **React:** `@novu/react` 3.21 has `<Inbox />` plus headless `useNotifications` and `useCounts` hooks with real-time WebSocket events. React 19 is a declared peer.
   - **Identity:** It uses a deterministic HMAC-SHA256 of `subscriberId`, which the server can compute per request for whoever the current user is. The hash never expires, so it is simpler than Knock's token but weaker.
   - **Weaknesses:** The free tier caps you at 5 workflows and 3 team members and keeps the activity feed for only 24 hours. It has no documented sandbox or skip-delivery flag like Knock's.

SuprSend is a close third: a 10k free tier, ES256 JWTs, a Test Mode and strong headless hooks. But it can only be self-hosted on Enterprise, and its server SDK is less TypeScript-native. Courier is also capable, but its docs don't say whether the free tier includes Inbox. MagicBell's free tier is too small (1,000 deliveries and 1 project).

## Comparison table

| Vendor | Free tier | React inbox / headless | TS server SDK | User identity | Real-time | Local dev / tests | Email later | Self-host |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Knock | 10k messages/mo, unlimited environments | `@knocklabs/react` 0.15.1 (components + hooks; React 17-19 peer) | `@knocklabs/node` 1.41.1 (TS ≥4.9, Apache-2.0) | RS256 JWT (`sub`=user id) via `signUserToken`; re-`authenticate` to switch | Yes ("real-time service") | Dev/prod environments, own keys; `sandbox_mode` and `skip_delay` trigger flags; dashboard test runner | Yes, 12+ providers incl. Resend, SES, Postmark | No |
| Novu | 10k workflow runs/mo, 2 envs, 5 workflows, 24h activity retention | `@novu/react` 3.21.0 (`<Inbox/>`, `useNotifications`, `useCounts`); `@novu/js` | `@novu/api` 3.21.0 | HMAC-SHA256(secret, subscriberId) as `subscriberHash` | Yes, WebSocket events | Dev/Prod envs, own keys; demo email provider; self-host via Docker Compose | Yes, 19 providers | Yes: MIT core, `/enterprise` commercial |
| Courier | 10k sends/mo ("Developer") | `@trycourier/courier-react` 9.3.5 (`<CourierInbox/>`, `useCourier`) | `@trycourier/courier` 10.3.0 (Apache-2.0) | JWT from `POST /auth/issue-token` with `user_id:` scope; `signIn` / `signOut` | Yes, WebSocket (`listenForUpdates`) | Production + Test environments (+ custom), per-env keys; docs suggest `jest.mock` of `useCourier` | Yes (core product) | No |
| MagicBell | 1,000 deliveries/mo, 1 project | `@magicbell/react` 1.14.0 (Provider, Inbox); `@magicbell/react-headless` 5.5.1 | `magicbell-js` 1.8.0 (`project-client`) | HS256 JWT (`user_external_id`/`user_email`, `api_key`) | Yes (persistent connection; Ably per API ref) | Only 1 project on free, so no dev/prod split | Yes (per-channel overrides) | No |
| SuprSend | 10k notifications/mo, inbox included (branded), 30-day logs | `@suprsend/react` 1.6.0 (Inbox) + `@suprsend/react-core` 2.5.0 hooks | `@suprsend/node-sdk` 1.15.1 (ships `types/index.d.ts`, MIT) | ES256 JWT (`entity_type: subscriber`, `entity_id`); `userToken` + `refreshUserToken` on provider | Yes (socket host; reachability status) | Staging / Production / Sandbox workspaces, own keys; Test Mode in staging | Yes | Enterprise only (self-hosted / BYOC beta) |

## Knock

- **Free tier.** $0 a month for up to 10,000 messages. Stored user profiles are unlimited, and so are isolated environments. The next tier is Starter at $250 a month plus usage. [knock.app/pricing](https://knock.app/pricing)
- **React.** `@knocklabs/react` contains the pre-built components (`NotificationFeedPopover`, `NotificationFeed`, `NotificationIconButton`) and "a complete set of React hooks" for headless UIs. `@knocklabs/react-notification-feed` is legacy. [docs.knock.app/in-app-ui/react/overview](https://docs.knock.app/in-app-ui/react/overview)
  - **Hooks.** `useNotifications(knock, feedChannelId, options)` returns a Feed client and manages setup, real-time updates and cleanup. `useNotificationStore(feedClient, selector?)` returns `items`, `metadata` (unread, unseen and total counts), `pageInfo`, `loading` and `networkStatus`. [use-notifications](https://docs.knock.app/in-app-ui/react/sdk/hooks/use-notifications), [use-notification-store](https://docs.knock.app/in-app-ui/react/sdk/hooks/use-notification-store)
  - **Gotcha.** Headless hooks don't call `markAllAsSeen()` for you, so you must do it or the badge won't clear. [custom-notifications-ui](https://docs.knock.app/in-app-ui/react/custom-notifications-ui)
  - **Version.** npm shows v0.15.1, with peers `react ^17 || ^18 || ^19`. Its peer list also names `next` and `@tanstack/react-router` (from `npm view @knocklabs/react peerDependencies`). Whether those two are optional was not checked.
- **Server SDK.** `@knocklabs/node` (v1.41.1, Apache-2.0) supports "TypeScript >= 4.9" and types every request and response. `client.workflows.trigger(key, { recipients, data })` sends from the server. Nothing has to call you back, so no incoming webhooks are needed. `signUserToken('user-1', { expiresInSeconds })` signs with `jose`, which is ESM-only, so CJS projects on older Node versions need `jose` v5. [github.com/knocklabs/knock-node](https://github.com/knocklabs/knock-node)
- **User identity.** In enhanced security mode, client requests need a JWT signed with RS256 using the environment's private signing key (`KNOCK_SIGNING_KEY`). The `sub` claim (the user ID) is required, and `iat` and `exp` are recommended.
  - **Passing it to the client.** The token reaches React through `KnockProvider userToken`. Without enhanced security, the public key "can act as any user."
  - **Refresh.** The `onUserTokenExpiring` callback supports refreshing the token.
  - **Switching users.** You can call `authenticate` again at any time with a new user token, which restarts the real-time connection. This maps cleanly onto Cadence's user switcher: the root loader signs a token for the current user and passes it down. Re-keying the provider on the user ID is my suggestion, not something the docs say. [security-and-authentication](https://docs.knock.app/in-app-ui/security-and-authentication)
- **Real-time.** The feed is "backed by the Knock Feed API and real-time service", and the hooks handle real-time updates. The docs pages I read don't name the transport. [custom-notifications-ui](https://docs.knock.app/in-app-ui/react/custom-notifications-ui)
- **Local dev and tests.**
  - **Environments.** Enabling environments adds a development environment alongside production. Each environment has its own API key pair and isolated data. Workflows are committed in development and then promoted. [concepts/environments](https://docs.knock.app/concepts/environments)
  - **Trigger flags.** The trigger API takes `settings: { sandbox_mode, skip_delay }`. Sandbox mode generates and previews messages without delivering them to providers. [triggering-workflows/api](https://docs.knock.app/send-notifications/triggering-workflows/api), [testing-workflows](https://docs.knock.app/send-notifications/testing-workflows)
  - **CLI.** The Knock CLI can commit and promote workflows from CI. [knock-cli](https://docs.knock.app/developer-tools/knock-cli)
  - **Mocking.** I found no official guidance on mocking the SDK.
- **Email later.** Supported email providers include Amazon SES, Mailgun, Postmark, Resend, SendGrid, SMTP and others. "Knock (testing only)" is a built-in test email provider. [integrations/email/overview](https://docs.knock.app/integrations/email/overview)
- **Self-host.** No self-hosted offering appeared on the pricing page or in the docs I read (not verified as an explicit statement).

## Novu

- **Free tier.** 10,000 workflow runs a month, unlimited subscribers, and 2 environments (Dev and Prod). It caps you at 5 workflows and 3 team members, allows 60 events per second, and keeps the activity feed for 24 hours. Pro starts at $30 a month and Team at $250 a month. [novu.co/pricing](https://novu.co/pricing)
- **React.** `@novu/react` provides `<Inbox />` and headless hooks. `@novu/js` is the framework-agnostic client. [platform/inbox/overview](https://docs.novu.co/platform/inbox/overview)
  - **Hooks.** `useNotifications` covers the list, pagination and real-time updates. `useCounts` gives the unread, unseen and total counts. `useNovu` gives access to the client. `NovuProvider` sits at the root. [headless-mode](https://docs.novu.co/platform/inbox/headless-mode), [react/hooks](https://docs.novu.co/platform/inbox/react/hooks)
  - **Version.** npm shows v3.21.0 (ISC), with peers `react ^18 || ^19`.
- **Server SDK.** `@novu/api` (v3.21.0) is "Novu's Typescript SDK" and runs on Node, Bun, Deno and Workers. You send with `novu.trigger({ to: { subscriberId, email, ... }, workflowId, payload })`. Yarn users must add `zod@^3`. [sdks/server/typescript](https://docs.novu.co/platform/sdks/server/typescript)
  - **Code-first workflows.** `@novu/framework` (v2.14.0) defines workflows in code, served from your app as a "bridge". I did not evaluate it in depth. Note that a bridge is an endpoint Novu calls into your app. That is not needed with dashboard-defined workflows.
- **User identity.** Turn on "Security HMAC encryption" in the In-App provider settings. The server computes `HMAC-SHA256(secretKey, subscriberId)` as lowercase hex and passes it as `<Inbox subscriberHash>`. With HMAC on and no hash, "the Inbox will not load." [prepare-for-production](https://docs.novu.co/platform/inbox/prepare-for-production)
  - **Switching users.** The hash is deterministic and has no expiry. Switching users means passing a new `subscriber` and hash. This is easy for Cadence, but a leaked hash is valid forever. That last point is my inference from the scheme.
- **Real-time.** `useNotifications` and `useCounts` "automatically listen for real-time events". You can also subscribe by hand with `novu.on('notifications.notification_received')`. The `socketUrl` prop points at the WebSocket server, for example `wss://eu.socket.novu.co`. [headless-mode](https://docs.novu.co/platform/inbox/headless-mode), [react/hooks](https://docs.novu.co/platform/inbox/react/hooks)
- **Local dev and tests.**
  - **Environments.** Development and Production have separate Application Identifiers (public) and secret keys. You publish from Development to Production. [concepts/environments](https://docs.novu.co/platform/concepts/environments)
  - **Mocking.** The docs give no mocking or sandbox-flag guidance.
  - **Email.** A built-in demo email integration exists for testing before you connect a real provider. [integrations/email](https://docs.novu.co/platform/integrations/email)
  - **Self-host for dev and CI.** A Docker Compose script runs the API, worker, ws and dashboard services with MongoDB and Redis. The dashboard runs on `localhost:4000`. [deploy-with-docker](https://docs.novu.co/community/self-hosting-novu/deploy-with-docker)
- **Email later.** 19 email providers are supported, and you add an Email step to a workflow. [integrations/email](https://docs.novu.co/platform/integrations/email)
- **Self-host and license.**
  - **License.** The core is MIT. The `/enterprise` directory is under a commercial license. [github.com/novuhq/novu](https://github.com/novuhq/novu)
  - **Self-hosting limits.** Self-hosted Novu has no social login, and "some features exclusive to Novu's cloud-managed solution will not be available." [self-hosting overview](https://docs.novu.co/community/self-hosting-novu/overview)
  - **Kubernetes.** There is no official Helm chart. [deploy-with-docker](https://docs.novu.co/community/self-hosting-novu/deploy-with-docker)

## Courier

- **Free tier.** The Developer plan is free and includes 10,000 sends a month, where one send is one template to one recipient. Business is pay-as-you-go at $0.005 per send, including the first 10k. [courier.com/pricing](https://www.courier.com/pricing)
- **React.** `@trycourier/courier-react` (v9.3.5, peer `react >=18`) is for React 18+, and `@trycourier/courier-react-17` covers React 17. The source is MIT-licensed. [github.com/trycourier/courier-web](https://github.com/trycourier/courier-web)
  - **Hooks.** `useCourier()` returns `{ shared, auth, inbox, toast, preferences }`. `inbox` exposes `feeds`, `totalUnreadCount`, `readMessage`, `archiveMessage`, `fetchNextPageOfMessages` and more. [courier-react-web](https://www.courier.com/docs/sdk-libraries/courier-react-web#usecourier-hook), [build-a-custom-inbox](https://www.courier.com/docs/in-app/build-a-custom-inbox)
  - **v8 migration.** The old `@trycourier/react-hooks` and HMAC/clientKey auth were removed, and the SDK is now JWT-only. [v8 migration guide](https://www.courier.com/docs/sdk-libraries/courier-react-v8-migration-guide)
- **Server SDK.** `@trycourier/courier` (v10.3.0, Apache-2.0) is typed and reads `COURIER_API_KEY`. You send with `client.send.message({ message: { to, content } })` and mint client tokens with `client.auth.issueToken({ scope, expires_in })`. [github.com/trycourier/courier-node](https://github.com/trycourier/courier-node)
- **User identity.** The backend calls `POST /auth/issue-token` with scopes such as `user_id:<id> inbox:read:messages inbox:write:events`. Leave out `expires_in` and the token "never expires". The SDKs don't refresh tokens.
  - **Client.** The client calls `courier.shared.signIn({ userId, jwt })` and `auth.signOut()`.
  - **Switching users.** Signing out and back in fits Cadence's user switcher. The docs also warn that in dev, effects that run twice can sign the user out again, so guard `signIn` with a ref. [inbox/authentication](https://www.courier.com/docs/platform/inbox/authentication), [add-an-inbox-to-nextjs](https://www.courier.com/docs/guides/add-an-inbox-to-nextjs)
- **Real-time.** It uses a WebSocket. Headless UIs must call `inbox.listenForUpdates()` after `signIn`. The JS SDK's `inbox.socket.connect()` does not push the unread total, so you refetch the count when an event arrives. [build-a-custom-inbox](https://www.courier.com/docs/in-app/build-a-custom-inbox), [courier-js-web](https://www.courier.com/docs/sdk-libraries/courier-js-web)
- **Local dev and tests.**
  - **Environments.** Every workspace starts with Production and Test environments, and you can add custom ones. Each API key belongs to one environment. [workspaces-overview](https://courier.com/docs/platform/workspaces/workspaces-overview), [changelog: custom environments](https://www.courier.com/changelog/new-custom-environments)
  - **Mocking.** The docs show mocking `useCourier()` with `jest.mock('@trycourier/courier-react', ...)`. [build-a-custom-inbox](https://www.courier.com/docs/in-app/build-a-custom-inbox)
- **Email later.** Email is Courier's core multi-channel product. The SDK's send example targets an email address. [courier-node](https://github.com/trycourier/courier-node)
- **Self-host.** Not offered, as far as I found.

## MagicBell

- **Free tier.** The Builder plan has 1,000 deliveries a month and 1 project. Deliveries count per channel. Startup costs $249 a month for 50,000 deliveries. [magicbell.com/pricing](https://www.magicbell.com/pricing)
- **React.**
  - **Inbox.** `@magicbell/react` (v1.14.0) puts the `Provider` in `@magicbell/react/context-provider`, which takes a required `token` (a User JWT). The Inbox is in `@magicbell/react/inbox`. That docs page mentions no headless hooks. [libraries/magicbell-react](https://www.magicbell.com/docs/libraries/magicbell-react)
  - **Headless.** `@magicbell/react-headless` v5.5.1 (peer `react >= 18.3.1`) exists on npm. I did not confirm that it is current or recommended.
- **Server SDK.** `magicbell-js` (v1.8.0) exposes `magicbell-js/project-client`. You send with `client.broadcasts.createBroadcast(...)`, which has per-channel overrides for email, in-app, push and SMS. The examples are TypeScript with typed models. [libraries/magicbell-js](https://www.magicbell.com/docs/libraries/magicbell-js)
- **User identity.** The User JWT is HS256, signed with the project's Secret Key. Its claims are `user_email` and/or `user_external_id`, plus `api_key`. The examples use a 1-year expiry. To switch users, the server signs a token per user and passes it to `Provider token`. [api/authentication/user](https://www.magicbell.com/docs/api/authentication/user)
- **Real-time.** The in-app channel keeps "a persistent connection" open. The API reference's `connection_id` refers to an Ably connection. [channels/in_app](https://www.magicbell.com/docs/channels/in_app), [api/reference](https://www.magicbell.com/docs/api/reference)
- **Local dev and tests.** The free tier allows 1 project, so there is no free dev/prod split. [pricing](https://www.magicbell.com/pricing) I found no test-mode or mocking docs.
- **Self-host.** Not offered, as far as I found. The `@magicbell/magicbell-react` license is "SEE LICENSE IN LICENSE" (npm).

## SuprSend

- **Free tier.** 10k notifications a month, with no credit card. It includes the in-app feed with SuprSend branding, "Multiple Environments", and 30-day log retention. Essentials costs $100 a month. [suprsend.com/pricing](https://www.suprsend.com/pricing)
- **React.**
  - **Packages.** `@suprsend/react` (v1.6.0, MIT) builds on `@suprsend/react-core` (v2.5.0) and adds drop-in `Inbox`, `NotificationsFeed` and `Preferences`. The old `@suprsend/react-inbox` is no longer supported. [inbox-react](https://docs.suprsend.com/docs/inbox-react)
  - **Headless.** It uses `SuprSendProvider` (props `publicApiKey`, `distinctId`, `userToken`, `refreshUserToken`) and `SuprSendFeedProvider`. The hooks are `useFeedData`, `useFeedClient` and `useFeed` (reachability: ONLINE, CONNECTING, AUTH_ERROR…). [react-headless](https://docs.suprsend.com/docs/react-headless)
- **Server SDK.** `@suprsend/node-sdk` (v1.15.1, MIT) has README examples in CommonJS JavaScript. The npm `types` field points to `types/index.d.ts`, so type definitions ship, though they aren't generated from TS source. You send with `supr_client.workflows.trigger(new WorkflowTriggerRequest(body, { idempotency_key }))`. [github.com/suprsend/suprsend-node-sdk](https://github.com/suprsend/suprsend-node-sdk)
- **User identity.** In enhanced security mode, the client sends an ES256 JWT signed with a dashboard-generated signing key. Its claims are `entity_type: "subscriber"`, `entity_id` (the distinct ID), `exp`, `iat`, and an optional `scope`. A refresh callback handles expiry.
  - **Switching users.** `SuprSendProvider` "identifies and resets the user" from its `distinctId` and `userToken` props. [client-authentication](https://docs.suprsend.com/docs/client-authentication), [react-headless](https://docs.suprsend.com/docs/react-headless)
- **Real-time.** The feed provider takes `host: { socketHost, apiHost }`, and the feed client emits `feed.new_notification`. [react-headless](https://docs.suprsend.com/docs/react-headless), [react-toast-notifications](https://docs.suprsend.com/docs/react-toast-notifications)
- **Local dev and tests.**
  - **Workspaces.** Staging, Production and Sandbox workspaces each have their own keys, so "staging credentials can never accidentally trigger production notifications."
  - **Test Mode.** Available in staging, it delivers only to designated internal testers, with a catch-all channel for everyone else. [workspaces](https://docs.suprsend.com/docs/workspaces)
- **Email later.** The workflow trigger body carries `$email` on recipients. [suprsend-node-sdk](https://github.com/suprsend/suprsend-node-sdk)
- **Self-host.** Only on Enterprise ("Self-Hosted Infrastructure"). BYOC is in beta through sales. [pricing](https://www.suprsend.com/pricing)

## Dropped alternatives

- **Engagespot.** The latest npm publishes are old: `@engagespot/react-component` 3.0.1 was last modified 2025-05-09, and `@engagespot/node` 1.7.0 on 2024-11-06. Its pricing page (`engagespot.co/pricing`) returned 404. The release cadence is weak compared with the five above.
- **Stream (Activity Feeds).** It is a feed and social-activity product, not a notification orchestrator with email workflows. It was not researched further.
- **Customer.io in-app.** Its in-app messaging is aimed at marketing campaigns rather than a product notification inbox driven by server-side workflow triggers. It was not researched further. This is a judgment call that was not checked against Customer.io docs.

## Unverified items

- Courier: whether the free Developer plan includes Inbox. The pricing page doesn't map Inbox to any plan.
- Novu: whether the free tier includes the Inbox with no extra limits. The pricing page lists runs, environments and retention only. The self-hosted vs cloud comparison page (the Community edition's Inbox feature gaps) returned 404.
- Knock: which real-time transport it uses (the docs say only "real-time service"). There is no explicit "not self-hostable" statement, only the absence of an offering. Whether the `next` and `@tanstack/react-router` peers of `@knocklabs/react` are optional (likely, but not checked in `peerDependenciesMeta`).
- MagicBell: whether `@magicbell/react-headless` is the current recommended headless path or legacy. Whether the realtime transport is Ably, which is inferred from the API reference's `connection_id` field. That any test or dev mode exists.
- SuprSend: how many workspaces the free tier allows (the page says only "Multiple Environments").
- Courier: the Draft/Published key split mentioned in its Environments announcement blog may be outdated. The current docs mention only per-environment keys.
- None of the five documents an official offline mock or emulator. Novu's self-hosted Docker stack is the only way to run a vendor locally. For unit tests, wrap the vendor SDK behind a Cadence-owned notifier module and fake it. This is my recommendation, not vendor guidance, except that Courier documents `jest.mock` of its React hook.
- npm package pages (npmjs.com) returned 403 to the fetcher. All versions and licenses came from `npm view` against the registry.

## Sources

- Knock: https://knock.app/pricing, https://docs.knock.app/in-app-ui/security-and-authentication, https://docs.knock.app/in-app-ui/react/overview, https://docs.knock.app/in-app-ui/react/sdk/hooks/use-notifications, https://docs.knock.app/in-app-ui/react/sdk/hooks/use-notification-store, https://docs.knock.app/in-app-ui/react/custom-notifications-ui, https://docs.knock.app/concepts/environments, https://docs.knock.app/send-notifications/testing-workflows, https://docs.knock.app/send-notifications/triggering-workflows/api, https://docs.knock.app/developer-tools/knock-cli, https://docs.knock.app/integrations/email/overview, https://github.com/knocklabs/knock-node
- Novu: https://novu.co/pricing, https://docs.novu.co/platform/inbox/overview, https://docs.novu.co/platform/inbox/headless-mode, https://docs.novu.co/platform/inbox/react/hooks, https://docs.novu.co/platform/inbox/prepare-for-production, https://docs.novu.co/platform/concepts/environments, https://docs.novu.co/platform/sdks/server/typescript, https://docs.novu.co/platform/integrations/email, https://docs.novu.co/community/self-hosting-novu/overview, https://docs.novu.co/community/self-hosting-novu/deploy-with-docker, https://github.com/novuhq/novu
- Courier: https://www.courier.com/pricing, https://www.courier.com/docs/platform/inbox/authentication, https://www.courier.com/docs/sdk-libraries/courier-react-web, https://www.courier.com/docs/sdk-libraries/courier-js-web, https://www.courier.com/docs/sdk-libraries/courier-react-v8-migration-guide, https://www.courier.com/docs/in-app/build-a-custom-inbox, https://www.courier.com/docs/guides/add-an-inbox-to-nextjs, https://courier.com/docs/platform/workspaces/workspaces-overview, https://www.courier.com/changelog/new-custom-environments, https://github.com/trycourier/courier-web, https://github.com/trycourier/courier-node
- MagicBell: https://www.magicbell.com/pricing, https://www.magicbell.com/docs/api/authentication/user, https://www.magicbell.com/docs/libraries/magicbell-react, https://www.magicbell.com/docs/libraries/magicbell-js, https://www.magicbell.com/docs/channels/in_app, https://www.magicbell.com/docs/api/reference
- SuprSend: https://www.suprsend.com/pricing, https://docs.suprsend.com/docs/inbox-react, https://docs.suprsend.com/docs/react-headless, https://docs.suprsend.com/docs/react-toast-notifications, https://docs.suprsend.com/docs/client-authentication, https://docs.suprsend.com/docs/workspaces, https://github.com/suprsend/suprsend-node-sdk
- npm registry (`npm view <pkg> version time.modified license peerDependencies`, 2026-11-24): @knocklabs/node, @knocklabs/react, @knocklabs/client, @novu/api, @novu/react, @novu/js, @novu/framework, @trycourier/courier, @trycourier/courier-react, @magicbell/react, @magicbell/react-headless, @magicbell/magicbell-react, magicbell-js, @suprsend/node-sdk, @suprsend/react, @suprsend/react-core, @suprsend/react-inbox, @engagespot/react-component, @engagespot/node
