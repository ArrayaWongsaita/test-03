# TaskFlow stack integration research

Research ticket: [TaskFlow — ตรวจความเข้ากันได้ของ Next.js Prisma Better Auth และ Google SMTP](https://github.com/ArrayaWongsaita/test-03/issues/2). Investigated 2026-10-07. Findings inform decisions; no application code, credentials, databases, or provider choices were changed.

## Actual installed baseline

Read from the main workspace, not the clean research worktree: `next/package.json` reports **16.3.8**; `prisma/package.json` reports **8.0.0-rc.19**; TypeScript is **5.9.3**, current local Node **25.8.1**. The root manifest has Prisma CLI but no Better Auth, PostgreSQL runtime, `@prisma/client`, or `@prisma/adapter-pg`; no application Prisma schema/contract was found. Existing `prisma.config.ts` only configures agent skills. These are local observations, not proof of a working database integration.

Installed Prisma exports `definePrismaConfig` from `prisma/config`, requires Node >=22.18.0, and depends on `@prisma/orm-toolchain@8.0.0-rc.13`. CLI `--help` exposes `contract`, `db`, `migration`, and `orm init`, confirming the changed command surface.

## Prisma 8 is a different integration path

Official [release status](https://www.prisma.io/docs/orm/release-status) confirms Prisma 8 is still a release candidate, needs Node >=22.18 and TypeScript >=5.9, and uses `@prisma/orm-postgres` for PostgreSQL. The Prisma CLI and runtime version numbers differ. `@prisma/client` remains Prisma 7. The v8 CLI has no `generate`, `migrate dev`, or `db push`. `npm view prisma dist-tags --json` during this investigation returned `latest=8.0.0-rc.20`, `prev=7.10.0`: an unqualified install selects an RC. Prisma 7 remains supported for 18 months after v8 GA; this is a support policy, not a guaranteed calendar GA date.

[Prisma 8 configuration](https://docs.prisma.io/docs/cli/configuration) wraps an `orm` section with `definePrismaConfig`, and PostgreSQL configuration comes from `@prisma/orm-postgres/config`: contract path, output, and `db.connection`. `contract emit` emits artifacts; database verification/signing and migration operations have their own commands. A Prisma 7 config is rejected by this CLI. Do not copy v7 commands into the installed v8 setup.

[Better Auth's Prisma adapter](https://better-auth.com/docs/adapters/prisma) explicitly demonstrates **Prisma 7**, a generated PrismaClient, and a PostgreSQL driver adapter. It generates auth schema but delegates migration application to Prisma. This is evidence for v7 support, not evidence that the rewritten v8 runtime can be passed to `prismaAdapter`. No verified native v8 adapter was established in this investigation.

| Option for the human version decision | Integration consequences |
| --- | --- |
| Prisma 7 for auth and application data | Matches the documented Better Auth adapter. Pin major/exact versions, install v7 CLI/client/adapter; change config and account for the current skills-sync postinstall separately. |
| Keep Prisma 8 for app data, use Prisma 7 for auth | Official side-by-side migration exists. Separate clients/configs/commands, one database schema migration owner; additional complexity for a new application. |
| Prisma 8 only | Requires a separately verified Better Auth integration/custom adapter. Do not label compatible based on schema-reading alone. More integration and RC risk. |

The [official v7-to-v8 migration guide](https://docs.prisma.io/docs/guides/upgrade-prisma-orm/postgresql) demonstrates `@prisma/prisma7` and `prisma7.config.ts` alongside the v8 runtime. It explicitly separates migration ownership from route migration. Current guide examples use newer RCs than installed rc.19, so they are architectural evidence, not a compatibility test of that exact installed release.

For the v7 option, [v7 client setup](https://www.prisma.io/docs/orm/v7/prisma-client/setup-and-configuration/introduction) uses `prisma-client` with explicit output, generated-client imports, and `PrismaPg`/`@prisma/adapter-pg`; share a client/pool per process and account for serverless concurrency. DATABASE_URL belongs to server configuration; migrations need a production-safe deployment path. Package installation, client generation, migration rehearsal, and an actual PostgreSQL integration run remain execution work.

## Next.js session and authorization boundaries

Read installed `node_modules/next/dist/docs/01-app/02-guides/authentication.md`, `01-app/03-api-reference/03-file-conventions/proxy.md`, and the route-handlers guide. This installed version calls the convention **proxy.ts**; middleware is deprecated. Proxy is optional for early redirects and cookie checks; enforce real session and object permissions near the data source in a server-only data access layer. Server Actions and Route Handlers are public entry points requiring their own authorization. Hiding buttons or protecting a layout cannot protect writes or nested entry points. Local guides recommend avoiding database reads on every prefetched Proxy request. [Public authentication guide](https://nextjs.org/docs/app/guides/authentication), [Proxy reference](https://nextjs.org/docs/app/api-reference/file-conventions/proxy).

[Better Auth Next integration](https://better-auth.com/docs/integrations/next) mounts GET/POST via `toNextJsHandler` at `/api/auth/[...all]`; server session lookup passes `headers: await headers()`. If invoking cookie-setting auth APIs through Server Actions, put `nextCookies()` last. React Server Components cannot set cookies. Use Node-capable deployment for the PostgreSQL/SMTP path; verify chosen hosting's outbound SMTP and background work support rather than assuming an Edge runtime can run it.

TaskFlow roles/division/project visibility need explicit server predicates. [Admin plugin](https://better-auth.com/docs/plugins/admin) supports custom access control and user/session management; built-in admin/user roles do not automatically implement Supervisor scope. An app's task/project operations must call its chosen policies. Bootstrap of the first Super Admin and disabling staff must be specified; the plugin's ban operation revokes sessions.

## Invite-only accounts and password setup

[Email/password](https://better-auth.com/docs/authentication/email-password) supports `disableSignUp`, reset callbacks and tokens, and optional session revocation on password reset. SMTP delivery is supplied by the app. Avoid email-response timing leaks; serverless unawaited work needs a supported lifetime mechanism such as waitUntil or durable work.

[Organization invitations](https://better-auth.com/docs/plugins/organization) supply invite/resend/cancel/expiry and email hooks. Acceptance requires an authenticated matching-email session; this is not automatically a first-password account-provisioning flow. Verified-email requirements are configurable and opaque invitation IDs matter. Using this plugin for one fixed organization is an option, not a requirement or a choice to build SaaS.

Two candidate flows require a human decision and later integration verification:

- **Pre-created account + setup/reset link:** keep public signup disabled, create the invited credential account from a protected admin flow, send the user a token link to choose a password, gate activation, and never disclose a temporary password. Documented admin creation requires a password; exact precreation/reset behavior and email-verification semantics need a test against the pinned Better Auth version.
- **Token-gated account creation:** invitee sets password during signup, but server validation admits only a live invitation for that email, then consumes it atomically. Public unauthenticated signup must be gated, not merely hidden. [Database hooks](https://better-auth.com/docs/concepts/database) can block account creation, but replay/race handling and invitation consumption remain app design. `disableSignUp=true` cannot be combined blindly with this client signup flow.

Whichever flow wins: define expiry, cancel/resend invalidation, existing-account behavior, duplicate acceptance, Super Admin bootstrap, and verified-email activation. Reset and invitation tokens must not appear in logs or task notifications. These are proposed acceptance checks, not resolved business rules.

## Google SMTP variants and limits

[Google app passwords](https://support.google.com/accounts/answer/185833?hl=en) require 2-Step Verification; availability depends on account/security policy, and changing the Google password revokes app passwords. A normal account password is not an adequate modern integration plan. [Google XOAUTH2](https://developers.google.com/workspace/gmail/imap/xoauth2-protocol) supports OAuth SMTP; that adds token/consent maintenance.

| Google account / service | Verified constraints and trade-off |
| --- | --- |
| Personal Gmail | [Gmail limits](https://support.google.com/mail/answer/22839?hl=en) document blocking after more than 500 emails/day or more than 500 recipients in one email, and recovery within 1–24 hours. Suitable volume must be checked; this is not a delivery SLA. |
| Workspace Gmail SMTP | [Google's app sending guide](https://knowledge.workspace.google.com/admin/gmail/send-email-from-a-printer-scanner-or-app) documents smtp.gmail.com, SSL 465 or TLS 587, full account identity/app password. [Workspace limits](https://knowledge.workspace.google.com/admin/gmail/gmail-sending-limits-in-google-workspace) document 2,000 messages/day, trial 500, and SMTP 100 recipients/message, with additional recipient limits. |
| Workspace SMTP relay | Google's app sending guide documents smtp-relay.gmail.com, administrator setup, up to 10,000 recipients/user/day, and possible static-IP requirements. Distinct quota/policy regime; requires Workspace administrative access. |

Limits can change and other activity shares quotas; confirm account type, administrator policy, daily staff/task volume, and burst volume before choosing. Hosting must allow outbound connection on the selected TLS SMTP port. Custom-domain sender authentication/deliverability should be verified during deployment; SMTP acceptance does not prove inbox delivery.

## Retries and durable sending: proposal, not an approved design

[RFC 5321](https://www.rfc-editor.org/rfc/rfc5321) distinguishes temporary 4xx from permanent 5xx failures. [Google SMTP errors](https://knowledge.workspace.google.com/admin/support/troubleshooting/gmail-smtp-errors-and-codes) show throttling, authentication/policy problems, and nonexistent recipients: inspect actual codes rather than retrying every failure.

Proposed design: transactional outbox with task-assignment writes; a worker records queued/sending/sent/failed, bounded attempts, and backoff with jitter. Retry transient network/4xx errors; pause quota exhaustion instead of a tight retry loop; surface auth/policy/recipient failures for intervention. Preserve the same event identifier to reduce duplicate job creation. A timeout after SMTP accepted DATA can still produce duplicate delivery: claim at-least-once behavior, not exactly-once. Expired/revoked invite/reset links should not be delivered from a stale queue. Email failure should not undo a committed assignment; display failed notification and allow controlled retry. User expectations and the hosting-supported worker mechanism remain decisions.

## Evidence limits and next checks

This is documentation/source research plus local read-only inspections, not a running compatibility proof. No PostgreSQL, SMTP account, or Better Auth install exists here to test. The research closes the factual investigation, not the version, invitation-flow, authorization, email reliability, or hosting decisions. Before production implementation, pin chosen packages and verify migration/generation, session cookies, direct unauthorized requests, invitation races, password setup/reset, revocation, and SMTP transient/permanent failures on the chosen runtime.
