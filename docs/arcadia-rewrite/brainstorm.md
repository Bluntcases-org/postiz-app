# Arcadia Rewrite Brainstorm

## Baseline

- Local fork path: `/Users/bluntcases-dev/workspaces/repos/postiz-app/repo`
- Fork remote: `git@github.com:Bluntcases-org/postiz-app.git`
- Upstream remote: `git@github.com:gitroomhq/postiz-app.git`
- Baseline commit: `cc54137d10036f636404353a583d6c9f0b847fb9`
- Baseline status: local `main`, `origin/main`, and `upstream/main` matched before this note was added.

## Intent

Use the five Arcadia stages to reverse engineer Postiz into software-system specifications that can seed a rewrite as:

- A Rust-based PostgreSQL extension installed into a serviced Postgres database.
- Cloudflare-hosted execution services for APIs, workflows, provider calls, queue workers, rate coordination, and media storage.

The goal is not to clone Postiz implementation architecture. The goal is to extract product behavior, operational responsibilities, data semantics, and provider workflows from Postiz, then re-express them as a Postgres-native control plane paired with Cloudflare execution.

## Architecture Thesis

- Postgres is the source of truth, control plane, scheduler kernel, and audit log.
- The Rust PostgreSQL extension owns deterministic state transitions, due-job selection, leases, idempotency, and schedule calculations.
- Cloudflare is the side-effect execution plane.
- Workers expose APIs and run provider adapters.
- Queues dispatch asynchronous work.
- Durable Objects coordinate per-provider or per-account concurrency and rate limits.
- Cloudflare Workflows may orchestrate complex multi-step publish flows when simple queue workers are insufficient.
- R2 stores media objects.
- Restate.dev is not part of the baseline; it remains an alternative workflow executor if Cloudflare-native orchestration proves insufficient.

## Arcadia Stage 1: Operational Analysis

Initial operational actors:

- Content creator
- Brand/team member
- Agency operator
- API client
- AI agent or MCP client
- Social platform
- Scheduler/runtime
- OAuth provider
- Analytics consumer

Initial operational activities:

- Connect social accounts.
- Upload and manage media.
- Compose posts.
- Schedule posts.
- Publish posts.
- Publish threads and follow-up comments.
- Refresh provider tokens.
- Recover missed posts.
- Collect analytics.
- Notify users and webhooks.
- Auto-generate RSS/AI posts.
- Coordinate multi-client/customer groups.

## Arcadia Stage 2: System Analysis

Candidate system of interest:

`Postgres-native social publishing control plane`

Core system functions:

- Maintain tenants, users, provider accounts, media, post groups, post segments, schedules, attempts, webhooks, analytics, and outbox events.
- Validate provider-specific post intent.
- Select due work atomically.
- Lease jobs safely.
- Emit work to Cloudflare Queues or Workflows.
- Record execution attempts and provider results.
- Preserve idempotency across retries.
- Maintain auditable state transitions.

Important boundary:

- The PostgreSQL extension should not call social APIs directly.
- Network side effects belong in Workers, Workflows, Queues, and provider adapter services.

## Arcadia Stage 3: Logical Architecture

Candidate logical components:

- Tenant and identity model
- Provider account registry
- Provider token/secrets model
- Media catalog
- Post intent model
- Post group and post segment model
- Schedule engine
- Job leasing engine
- Provider execution adapter interface
- Token refresh coordinator
- Analytics snapshot model
- Webhook/outbox model
- Agent/API facade
- Policy and limits module

Postiz reference seams to inspect:

- Frontend app: `apps/frontend/`
- Backend API: `apps/backend/`
- Temporal orchestrator: `apps/orchestrator/`
- Public API routes: `apps/backend/src/public-api/`
- Provider abstraction: `libraries/nestjs-libraries/src/integrations/`
- Prisma schema: `libraries/nestjs-libraries/src/database/prisma/schema.prisma`
- Post workflow: `apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts`
- Publish activity: `apps/orchestrator/src/activities/post.activity.ts`
- Post service/repository: `libraries/nestjs-libraries/src/database/prisma/posts/`

## Arcadia Stage 4: Physical Architecture

Candidate physical mapping:

| Logical responsibility | Physical target |
| --- | --- |
| Durable source of truth | Serviced Postgres |
| Atomic due-job selection | Rust PostgreSQL extension functions |
| Scheduling tables | Postgres schema |
| Job dispatch | Cloudflare Queues |
| Complex multi-step publish chains | Cloudflare Workflows |
| Provider API calls | Cloudflare Workers |
| Rate/concurrency coordination | Durable Objects per provider/account |
| Media storage | R2 |
| Public API | Cloudflare Workers |
| Webhooks/notifications | Queue plus Worker outbox processor |
| Analytics collection | Queue plus Worker plus snapshot tables |
| OAuth callback handling | Worker plus Postgres token storage |

## Arcadia Stage 5: Data Modelling

Candidate seed tables:

- `tenant`
- `user_account`
- `tenant_member`
- `provider`
- `provider_account`
- `provider_token`
- `media_asset`
- `post_group`
- `post_segment`
- `post_schedule`
- `post_job`
- `post_attempt`
- `provider_result`
- `analytics_snapshot`
- `webhook_endpoint`
- `outbox_event`
- `rate_limit_bucket`
- `oauth_state`

Initial improvement targets over current Postiz data shape:

- Make post grouping first-class instead of relying on implicit `group` strings and parent chains alone.
- Make attempts, retries, leases, and idempotency first-class instead of relying primarily on Temporal workflow state.
- Normalize media references instead of embedding media JSON in post rows.
- Persist analytics snapshots instead of relying only on provider reads or Redis cache.
- Use a durable outbox for webhooks and notifications.
- Clarify secrets handling for provider tokens.

## Restate And Cloudflare Workflows

Restate.dev could replace Temporal-style durable workflows, but it is not part of the baseline because Cloudflare already provides enough adjacent primitives to keep the architecture Postgres + Cloudflare:

- Cloudflare Workflows for durable multi-step orchestration.
- Cloudflare Queues for async dispatch.
- Durable Objects for per-key coordination.
- Workers for APIs and provider side effects.

Current stance:

- Baseline: Postgres extension plus Cloudflare primitives.
- Restate: alternative if Cloudflare Workflows, Queues, and Durable Objects prove insufficient or too manual.

## Immediate Next Questions

- Which Postiz operational activities are core for the rewrite, and which are out of scope?
- Should the rewrite initially support all providers or a narrow provider set?
- What is the minimal viable scheduler kernel inside Postgres?
- Which provider workflows require Cloudflare Workflows rather than plain Queues and Workers?
- How should OAuth secrets be stored and rotated in a serviced Postgres deployment?
- What API surface should be preserved from Postiz public API, and what should be redesigned?
