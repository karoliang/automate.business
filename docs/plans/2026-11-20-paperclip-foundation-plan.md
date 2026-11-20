# Plan: automate.business on a Paperclip foundation

Status: research complete, no code built. Date: 2026-11-20.
Owner decision (2026-11-20): automate.business will be based on Paperclip.
Reviewed upstream: `paperclipai/paperclip` commit `19a738139854718bf9d8c70d9e631fbb28895a9a` (2026-11-19), release line `v2026.1117.0`.
Method: web research plus six read-only reviews of the cloned repo (architecture, security code review, advisories and governance, company isolation, integrations and knowledge, knowledge layer and self-improvement). Nothing from the repo was installed, built or executed. Every code finding below comes from reading source, so items marked "confirmed" mean confirmed by code reading, not by a running test.

## 1. Product shape

automate.business is an agentic business operating system. Agents run as services and work through a business's workflows end to end. Two delivery models, one product:

- **Model A, self-run.** A business runs the system on its own infrastructure, connects its existing workflows, APIs, internal systems and, where no API exists, computer-use agents.
- **Model B, automate.business cloud.** A small-business operator buys a secure business cloud from us, runs the business on it, and brings their own model keys (BYOK) to power their agents.

Model B is Model A hosted by us as one isolated cell per customer.

## 2. Fixed platforms (owner decision 2026-11-20)

| Platform | Role |
|---|---|
| Supabase | Postgres for each business (system of record, pgvector knowledge base, queues via pgmq, cron); our own control plane (accounts, billing, cell registry, portal auth) |
| Netlify | Marketing site (and the customer portal front end) |
| Infisical | All secrets: cell master keys, auth secrets, connector OAuth tokens, customer BYOK model keys |
| AWS | Each business runs in its own isolated instance (cell): Paperclip server, agent sandboxes, computer-use VMs, document parsing workers, S3 raw files |

No other host, database or secrets manager without owner approval.

## 3. Landscape research summary

Resale as a hosted service requires a permissive license (MIT, Apache-2.0). AGPL is usable only if we publish our modifications.

| Layer | Choice | License | Notes |
|---|---|---|---|
| Company control plane | Paperclip | MIT | Companies, org chart, goals, issues, budgets, heartbeats, approvals. Chosen base. |
| Alternative control plane | OpenClaw Enterprise (OCE) | MIT, pre-1.0 | Foundation-run (OpenAI, Red Hat, Nvidia). Watch, do not adopt now. |
| Agent runtimes | Claude Code, Codex, Hermes Agent (MIT), OpenClaw (MIT) | mixed | OpenClaw skills registry had over 1,184 malicious skills by 2026-03-01. No public skills in customer cells. |
| Agent code frameworks | LangGraph (MIT), Mastra (Apache core), OpenAI Agents SDK (MIT), Microsoft Agent Framework (MIT), Google ADK (Apache) | | For custom business-logic agents exposed as HTTP services. |
| Workflow builder | Activepieces | MIT core | Only resale-safe visual builder found. |
| Durable jobs | Supabase Queues first; Trigger.dev (Apache) or Temporal (MIT) later | | |
| Computer use | Cua (MIT), Browser Use (MIT) | | |
| Code sandbox | E2B | Apache | Self-hostable on AWS via Terraform. Daytona went closed source 2026-06-11. |
| Model gateway | LiteLLM (optional) | MIT | PyPI releases backdoored on 2026-03-24 (1.82.7, 1.82.8) and 2026-10-21 (1.107.0). Pin by digest if used. |
| Chat UI | LibreChat (MIT) or our own | | LibreChat needs MongoDB; a small Supabase-native UI may fit better. |

Ruled out for hosted resale: n8n (Sustainable Use License), Dify (multi-tenant clause), Open WebUI (branding clause), LobeChat (derivative license), Nango (ELv2, embed only), Composio (closed infra), Honcho (AGPL), Firecrawl server (AGPL), Arize Phoenix (ELv2).

## 4. Paperclip facts

- Owner: Paperclip Labs, Inc., San Francisco. Founder @dotta (`cryppadotta`, 5,543 commits); next contributors 1,088 and 577. Bus factor is one person.
- Repo: created 2026-03-02, about 115K stars, 19K forks, about 6,950 open issues, default branch `master`.
- Releases: CalVer `vYYYY.MDD.patch`, every 1 to 2 weeks, npm dist-tags `latest`, `beta`, `nightly`, `canary`. No LTS, no backports. `v2026.1117.0` ran migrations 0375 to 0451, some lock tables, and declares itself a rollback floor. Migration 0423 silently grants `agents:read` to existing agents.
- Competitor risk: Paperclip Labs is building Paperclip Cloud (README waitlist, ROADMAP "Cloud deployments" in progress, cloud tenant code paths). Third-party hosts already sell managed Paperclip (Hostinger, Railway, DigitalOcean).
- Security reporting: GitHub advisories only, no stated timelines. No CodeQL, SAST or secret scanning in CI (issue #13008).

## 5. Findings

### 5.1 Architecture

- Server: Express 5, better-auth, websockets. UI: React 19 + Vite, served by the server (`SERVE_UI`). pnpm/turbo monorepo; `server/src/services` holds 1,422 files.
- Database: Drizzle ORM, Postgres only, 499 migrations, 198 schema files. Embedded Postgres 18 when `DATABASE_URL` is unset. Supabase is documented (`doc/DATABASE.md:68-125`): use `DATABASE_MIGRATION_URL` for the direct connection and `DATABASE_PREPARED_STATEMENTS=false` behind a pooler. Uses `pg_trgm`, `fuzzystrmatch`, plpgsql triggers, advisory locks and LISTEN/NOTIFY, so it needs a direct or session connection, not the transaction pooler. Up to 73 connections per process.
- Heartbeats: in-process scheduler (default 30 s tick). Dispatch via `ServerAdapterModule.execute()` (`packages/adapter-utils/src/types.ts:508`).
- Agent execution: `*_local` adapters spawn CLIs (Claude Code, Codex and others) as child processes of the server, or over SSH. Environment drivers: `local | ssh | sandbox | plugin`. A Rust `paperclip-runner` daemon exists but is experimental and off by default.
- Scaling: one server process per instance (`doc/INSTALLING.md:207`). Multi-process failover is only planned.
- Storage: `local_disk` or `s3` with custom endpoint.
- Secrets providers: `local_encrypted` (AES-256-GCM, one instance key) and AWS Secrets Manager with per-company KMS. GCP and Vault are stubs. No Infisical provider.
- Auth: better-auth, email and password only. No SSO, OIDC or SAML (issue #3028 open). Roles: `instance_admin`, company owner/admin/operator/viewer/member, 22 permission keys.
- Deployment: Dockerfile (node:24, port 3100, bundles the Rust runner), compose (Postgres 17 + server), ECS Fargate + RDS + EFS guide for a single task.

### 5.2 Security advisories (published)

| GHSA | Severity | Published | Issue |
|---|---|---|---|
| 68qg-g8mg-6pr7 | Critical 10.0 | 2026-04-10 | CVE-2026-41679, unauthenticated RCE via import authorization bypass. Public Metasploit module. |
| x8hx-rhr2-9rf7 | Critical 9.6 | 2026-07-22 | Drive-by RCE on local instances via DNS rebinding. Patched version not listed. |
| 47wq-cj9q-wpmp | Critical 9.9 | 2026-04-16 | Cross-tenant agent token minting |
| 3xx2-mqjm-hg9x | Critical 9.9 | 2026-04-16 | Cross-tenant agent key IDOR |
| vr7g-88fq-vhq3 | Critical | 2026-04-16 | OS command injection via `cleanupCommand` |
| 265w-rf2w-cjh4 | High 8.8 | 2026-04-16 | CVE-2026-41208, `provisionCommand` privilege escalation |
| gqqj-85qm-8qhf | High 8.7 | 2026-04-16 | `codex_local` inherited host Gmail connector and sent real email |
| xfqj-r5qw-8g4j | High 8.3 | 2026-04-16 | Unauthenticated endpoints in authenticated mode |
| w8hx-hqjv-vjcq | High 7.3 | 2026-04-16 | Malicious skills exfiltrate and destroy data |
| 3pw3-v88x-xj24 | Medium 6.5 | 2026-04-16 | Arbitrary file read via `instructionsFilePath` |
| fpw4-p57j-hqmq | Medium 5.4 | 2026-04-16 | Stored XSS in MarkdownBody |
| p7mm-r948-4q3q | Medium 4.3 | 2026-04-16 | Approval attribution spoofing |

Third-party write-up: Oasis Security (2026-08-05). Press: The Hacker News, CSO, SecurityWeek.

### 5.3 Security risks in the current code (ranked)

1. **Critical, confirmed by reading.** Local agents inherit the server environment minus only `PAPERCLIP_*` (`packages/adapter-utils/src/server-utils.ts:3617-3634`, `:4831`). The shipped compose and ECS files set `DATABASE_URL` and `BETTER_AUTH_SECRET`, so a prompt-injected agent gets full database access for every company and can forge agent JWTs (`server/src/agent-auth-jwt.ts:41`, `:91`). Codex limits its env with an allowlist; the Claude path does not.
2. **Critical.** Agents run on the host with permission bypass on by default (`claude-local/src/server/execute.ts:449`, `codex-local/src/index.ts:20`). The `bwrap` sandbox is opt-in, Linux-only, and without a filesystem scope it binds `/`. `enableManagedSandboxOnly` defaults to false.
3. **High.** One instance-wide AES-256-GCM key, no per-company derivation, no AAD, stored in `<instance>/secrets/master.key` inside the home directory agents can read.
4. **High, suspected.** Agents may edit their own adapter config; the denylist misses `filesystemScope`, `networkScope`, `networkAllowlist`, `filesystemExtraPaths` and `dangerouslySkipPermissions`.
5. **High, role threshold inferred.** Board members can set host shell commands (workspace commands, process adapter), which is host RCE.
6. **Medium.** Plugins run as unsandboxed forked processes with the server's user; `plugin-runtime-sandbox.ts` is dead code.
7. **Medium.** Approval creation accepts a caller-supplied `requestedByAgentId` with no same-company check (`server/src/routes/approvals.ts:422-423`).
8. **Medium.** Audit log rows are deleted or updated (`agent-deletion.ts:73`, `recovery/assigned-work-sweep.ts:948`). Open signup with no email verification. No API-wide rate limit. Cloud-tenant headers trust one shared token to impersonate any user (`server/src/middleware/auth.ts:648-665`).

Route authz sample: 1,143 handlers scanned, about 45 reviewed by hand, none confirmed unscoped. About 140 flagged handlers remain unreviewed. Company scoping is per-route (`assertCompanyAccess`, `server/src/routes/authz.ts:211`), with no central middleware.

### 5.4 Business entity isolation

Verdict: one deployment is one trust domain. Never place more than one paying customer in a deployment.

- No RLS anywhere: zero `CREATE POLICY` across 499 migrations. One database role sees every row. 176 of 198 schema tables carry `company_id`; scoping is application code only.
- Instance-wide state with no `company_id`: `environments` (config and env vars), `instance_settings`, plugins, `plugin_state` instance scope, `board_api_keys` (bound to a user, so a key carries every company the user belongs to).
- Plugin `ctx.db.query` sends no company scope and runs on the main DB handle (`packages/plugins/sdk/src/host-client-factory.ts:590-605`, `server/src/services/plugin-database.ts:556-562`), so whitelisted core tables return every company's rows. Suspected: unqualified table names bypass the plugin SQL whitelist (`plugin-database.ts:127-155`).
- Shared `~/.claude` and `.claude.json` across companies unless a managed AI connection is used. Codex homes are per company but `auth.json` is symlinked from the operator home. Host git credentials are forwarded when `hostCredentials` is on.
- Filesystem: per-company paths and traversal guards exist for project workspaces, run logs, storage keys and skills, but agent workspaces are `workspaces/<agentId>` and everything runs as one OS user, so guards do not bind agent processes.
- Websocket live events are company-gated; no leak found.

### 5.5 Integration of business services, apps and SaaS

Verdict: strong fit.

- MCP both ways: `packages/mcp-server` exposes Paperclip itself; a governed MCP gateway proxies external MCP servers with policy, approval, rate limit and audit on every call (`doc/MCP-ACCESS-GOVERNANCE.md:97-126`). Per run, agents' own `mcpServers` are replaced with gateway URLs and short-lived tokens.
- App catalog: 113 definitions (`packages/shared/src/app-definitions/`), including Gmail, Google Docs/Drive/Calendar, Slack, Notion, Stripe, Xero, Shopify, Supabase, Netlify, AWS, Zapier, Composio, generic OAuth and API key. Missing: HubSpot, QuickBooks, Microsoft 365 mail and files. Gmail has draft but no send.
- OAuth broker owned by Paperclip: PKCE, dynamic client registration, refresh under a database lease, credentials stored as company-scoped secret refs, grants to a user or an agent. Agents never hold provider tokens.
- Plug-in points: `http` adapter (fire-and-forget POST, success on any 2xx, callback via REST or MCP) for business-logic agents as services; `ServerAdapterModule` for deeper control; plugin SDK (`agent.tools.register`, `connectors.provide`, `webhooks.receive`, `api.routes.register`, `jobs.schedule`, `events.*`, `http.outbound`, `secrets.read-ref`, `environment.drivers.register`, own DB namespace, UI slots).
- Triggers: routines fire on schedule, webhook (bearer or HMAC signing), API or heartbeat.
- Computer use: Browser Use Cloud is a governed connection. Sandbox providers: e2b, daytona, modal, cloudflare, kubernetes (alpha), boxd, createos, novita, exe-dev. No Cua desktop driver, no AWS-native (ECS or Firecracker) provider.
- Gaps for small businesses: new connections enable every discovered action by default (`server/src/services/tool-access/catalog/default-access.ts:182-184`); Xero and other S4 apps require the customer to register their own OAuth app; Stripe, QuickBooks and HubSpot are deferred upstream; no provider event adapters (Gmail push, Stripe webhooks) beyond generic signed webhooks.

### 5.6 Porting business knowledge

Verdict: weak fit. Paperclip has containers, not a knowledge system.

- Present: per-company skills library (`SKILL.md`, agentskills.io compatible, pinned imports, scripts rejected), per-agent `AGENTS.md` instructions, company-scoped revisioned markdown documents, 10 MB attachments, connection instructions (2,000 chars each), PARA file-memory skill, experimental memory connectors (Mem0, Zep, Supermemory, Cognee, Honcho) off by default.
- `plugin-llm-wiki` ingests only Paperclip issues, comments and docs (opt-in). `wiki_search` is SQL `LIKE` on path and title, not body (`packages/plugins/plugin-llm-wiki/src/wiki/core.ts:4071-4082`).
- Missing: bulk import from Drive, Notion or email; continuous sync; embeddings or pgvector; full-text knowledge search; citations; role-based access to knowledge.
- Templates are software-company oriented: core-exec-team (CEO, CTO, QA), product-engineering, product-design, content-machine, phone receptionist. No sales, support, bookkeeping or marketing operations teams.

### 5.7 Self-improvement and self-optimisation

- Present and worth keeping: Reflection Coach (reads an agent's work history, proposes capped diffs to instructions, skills or tool text, never reflects on itself, propose and apply in separate runs), a replay gate of 3 to 5 past issues per proposal, a server consent gate, append-only instruction revisions with rollback, versioned company skills, promptfoo eval suites, per-run cost events (tokens, cached tokens, cents, links to issue, goal, project), budgets with warn at 80 percent and hard stop, per-run work assessment, board-curated decision training examples.
- Missing: the reflection routine ships paused and nothing runs it automatically; no automatic replay evals from production; model routing is only round robin or quota-aware, never by quality or cost; decision training data is captured but unused; agent friction reports go to the vendor, not into the customer's improvement.

### 5.8 Knowledge storage, vector search and speed

- pgvector HNSW on `halfvec(1536)`, 1M vectors: p95 about 8 to 25 ms for top-10 (secondary source). `halfvec` halves index size and cuts build time about 3x. pgvector 0.8 `iterative_scan` helps filtered queries.
- Embedding the query is the slow step (one test: `text-embedding-3-small` batch of 100, p50 1.1 s, p95 2.4 s). Cache query embeddings and answers.
- RLS cost: wrap `(select auth.uid())`, index policy columns, add `TO authenticated`, and filter explicitly in queries (Supabase reports 11 s to 7 ms).
- Move to a dedicated vector store (Qdrant, Apache-2.0) only when a single tenant reaches tens of millions of vectors or p95 exceeds budget.

## 6. Target architecture: one cell per customer

| Component | Placement |
|---|---|
| Paperclip server (one process) | AWS ECS task or VM per customer, own OS user |
| Paperclip database | Development: one shared Supabase project on the existing plan. Production: per-business database, option A or C (see 6.1), same AWS region as the cell, direct connection, separate migration role |
| Knowledge tables (`documents`, `chunks`, `memories`, `eval_cases`, `run_events`) | Same customer Supabase project, RLS on `business_id` as defence in depth |
| Raw files | S3, per-customer bucket or prefix, SSE-KMS, own IAM role |
| Document parsing | Docling (MIT) or Unstructured OSS (Apache) on AWS Lambda or ECS |
| Embeddings refresh | Supabase Automatic Embeddings (trigger, pgmq, pg_cron, edge function) |
| Secrets | Infisical project per customer: master key, `BETTER_AUTH_SECRET`, `PAPERCLIP_AGENT_JWT_SECRET`, OAuth tokens, BYOK keys |
| Agent execution | Remote sandbox only (E2B self-hosted or new AWS sandbox provider), default-deny egress with allowlist |
| Computer use | Cua desktop sandboxes on EC2, Browser Use for web |
| Workflows | Activepieces per cell, exposed to agents as an MCP connection |
| Customer portal, marketing | Netlify |
| Audit log | Shipped to an append-only external store per customer |

Shared multi-company deployments are allowed only for internal or demo use with no real customer data.

### 6.1 Database per business: options

Paperclip needs only Postgres (it uses better-auth, its own API and S3, not Supabase Auth, PostgREST or Realtime). The knowledge layer needs Postgres with pgvector, pgmq and pg_cron.

| Option | Isolation | Ops burden | Cost per business (Supabase list prices, 2026) | Notes |
|---|---|---|---|---|
| **A. Managed Supabase project per business (recommended)** | Own database, own credentials, own backups. Lives in Supabase's account, not the cell's AWS account. | Low: Supabase runs upgrades, backups, monitoring. Provision by Management API. | Small compute $15/month (Micro $10 has 60 direct connections; Paperclip can open up to 73 per process, so use Small with 90 or tune the pool). Org plan fee once: Pro $25 (includes $10 credit). | Pick the same AWS region as the cell. PrivateLink to the cell's VPC needs the Team or Enterprise plan, which removes the public database path. Customer data spans two vendors (AWS and Supabase); disclose in the DPA. |
| B. Self-hosted Supabase on AWS per business | Everything inside the customer's AWS cell | High: about ten containers per cell (Postgres, Auth, PostgREST, Realtime, Storage, Kong, Studio, edge runtime, pooler); we own patching, backups, HA and security fixes. No Supabase support. | AWS compute and storage only, plus our ops time | Pays for Supabase services Paperclip does not use. Only worth it for a premium tier that demands everything in the customer's own AWS account. |
| C. Plain Postgres in the cell (Amazon RDS or Aurora with pgvector) | Everything inside the cell, private subnet, IAM and KMS | Medium: AWS-managed, fewer moving parts than B | AWS pricing | Strongest single-boundary story and lighter than B. Steps outside the fixed Supabase platform, so needs owner approval. |

**Owner decisions 2026-11-20:**
- Development: one Supabase project on the owner's existing plan for the whole project while we build. It holds no real customer data, so a shared multi-company deployment is acceptable here.
- Production per business: AWS server (cell) per business is approved. The RDS premium tier (all data in the customer's own AWS account) is approved as a roadmap option.
- Demo account: to be set up later; it needs paid AWS server fees, so it waits for an owner go-ahead on spend.
- Per-business managed Supabase projects (option A) are not yet approved; revisit before the first paying business.

Original recommendation: A for every tier at launch, with PrivateLink once the plan supports it. If a customer requires all data inside their own AWS account, prefer C over B. Our own control plane (accounts, billing, cell registry) runs on one separate Supabase project that never holds customer business data.

## 7. Hardening patch set (required before any customer data)

1. Replace agent env inheritance with an explicit allowlist. Never pass `DATABASE_URL`, `BETTER_AUTH_SECRET`, JWT secrets, cloud credentials or model keys from the server env.
2. Set `enableManagedSandboxOnly` and run every agent in a remote sandbox. Disable local and process adapters and workspace shell commands for non-admins.
3. Force `dangerouslySkipPermissions` and the Codex bypass to false by default.
4. Change agent self-config from a denylist to an allowlist of editable fields.
5. Add an Infisical secret provider (or AWS Secrets Manager with per-customer KMS as an interim). Move `master.key` out of the agent-readable home.
6. Same-company check on `requestedByAgentId` in approvals.
7. Plugins: only our vetted plugins; disable npm plugin installs; review plugin SQL scoping before enabling any plugin that queries core tables.
8. Disable public signup; require email verification; add API-wide rate limiting and a WAF.
9. Append-only external audit log.
10. Ask-first defaults for outbound money and messaging actions (send email, pay invoice, refund, post publicly).
11. CI: CodeQL, secret scanning, dependency review on our fork.
12. Remove the UX labs and perf routes mounted without login (`ui/src/App.tsx:816-819`) and all dev-only screens.
13. Remove forced bypass flags in adapters we keep (claude-local permission skip; cursor-local and hermes `--yolo`; opencode-local `sudo -n`).
14. Replace the 20 `paperclip_cloud_connector` OAuth methods (Google apps, GitHub, Asana) with our own OAuth apps.
15. Replace all 40 upstream workflows with routed jobs on the runner router; drop `pull_request_target` with secrets.

Full screen and feature list with dispositions: `docs/research/2026-11-20-paperclip-feature-inventory.md`.

Offer each fix upstream as a PR; keep our patch set small so upgrades stay cheap.

## 8. Build plan (phases)

| Phase | Scope | Exit criteria |
|---|---|---|
| Design (before kickoff) | Owner-led full product design: user flows, end-to-end screen flow and UI design for both models (cloud and self-run), built on the business lifecycle and the Paperclip feature inventory. No build starts until this is done. | Owner signs off the design |
| 0. Verify | Throwaway instance, no real data: confirm the env leak, test whether remote sandbox providers inherit server secrets, test Supabase direct connection and migrations, measure upgrade cost across two releases. Needs owner go-ahead. | Written results; patch set confirmed or reduced |
| 1. Secure cell | Fork, rebrand, hardening patch set, Infisical provider, AWS sandbox provider, one-cell deployment on AWS + Supabase + S3 | Independent security review passes; cell rebuilt from scratch by script |
| 2. Integrations | Curated SMB connector set (Google Workspace send, Microsoft 365 mail and files, HubSpot, QuickBooks, Xero via our hosted OAuth apps), provider event adapters, ask-first presets, Activepieces bridge, HTTP agent-as-a-service template, Cua driver | A pilot business connects its core apps without registering OAuth apps |
| 3. Knowledge | Ingestion plugin (Drive, Notion, Gmail, M365, uploads), Docling parsing, pgvector hybrid search with RRF, cited retrieval tools, freshness (`valid_to`), onboarding that turns SOPs and brand voice into skills, business team templates (sales, support, bookkeeping, marketing) | Agents answer from the business's own documents with citations; p95 retrieval under 100 ms excluding embedding |
| 4. Self-improvement | Enable reflection routine, nightly production-to-eval-case mining, promptfoo replay gate, quality and cost model routing from `cost_events`, owner approval for customer-facing changes, DSPy/GEPA offline on high-traffic prompts | Measured quality up or cost down with no replay regressions |
| 5. Hosted cloud | Cell provisioning, billing, BYOK onboarding, backups and restore drills, upgrade pipeline, monitoring, incident runbook | First paying cell live |

## 9. Fork and upstream strategy

- Keep a full mirror with history of every upstream release we base on; MIT grants on released commits cannot be revoked.
- Track upstream releases, rebase the patch set, run our gates, and upgrade cells in waves. Plan for heavy migrations every release.
- Watch for relicensing, the Paperclip Cloud launch, and trademark policy changes.

## 10. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Upstream ships critical vulnerabilities often | Customer compromise | Cell isolation, sandbox-only agents, our own security review on every upgrade |
| Paperclip Cloud competes directly | Pricing and positioning pressure | Differentiate on security, small-business templates, knowledge layer, BYOK |
| Single maintainer, very fast churn | Upgrade cost, abandoned direction | Small patch set, mirror, option to freeze on a release |
| Relicensing of future versions | Lose upgrades | Pinned mirror; MIT on past commits stands |
| Agent prompt injection via business data and skills | Data exfiltration, unwanted actions | Egress allowlist, ask-first actions, vetted skills only |
| Per-customer Supabase and AWS costs | Margin | Price tiers; shared projects only for non-customer use |

## 11. Owner gates and open questions

- Phase 0 runs Paperclip on a throwaway instance; needs owner approval.
- Development database: decided, one Supabase project on the existing plan.
- Demo account on AWS: approved in principle; spend go-ahead needed before it is created.
- Paid services still to approve: per-business production databases, AWS cells, Infisical seats, E2B or EC2 sandboxes.
- Legal: license and trademark review of the fork and rebrand (see section 12 once complete); customer terms and DPA for hosted cells.
- Whether to use Paperclip's cloud-tenant mode at all (shared-token impersonation design) or keep pure single-company cells.

## 12. License and IP (audit 2026-11-20, not legal advice)

Items marked **[LAWYER]** need counsel before release.

### 12.0 Our own license

Owner decision 2026-11-20: automate.business's own code is open source under Apache-2.0 (`LICENSE`, `NOTICE`). Paperclip's MIT notices are kept alongside it.

### 12.1 What the license is

- Root `LICENSE` is standard MIT, "Copyright (c) 2025 Paperclip AI". `README.md:582` says "MIT (c) 2026 Paperclip Labs, Inc". Keep both lines.
- 41 package.json files declare MIT; the 10 without a field are `"private": true`. The Rust runner is MIT. No GPL, AGPL, SSPL, BSL or non-commercial code in Paperclip's own source, and no `ee/` directory.
- No CLA, DCO or rights assignment for contributions. Upstream PRs go in under MIT.
- MIT on already-released commits cannot be revoked. Future versions can be relicensed. The UI links to a closed "Paperclip EE" (`ui/src/components/TrustPresetSection.tsx:198-212`), an open-core signal.

### 12.2 CAN, CANNOT, MUST

| | Model A: self-run distribution | Model B: hosted cells |
|---|---|---|
| CAN | Fork, modify, rebrand, sell, sublicense, keep our patches closed | Same. MIT has no network-use clause |
| CANNOT | Redistribute Claude Code, the Claude Agent SDK, GitHub Copilot or Cursor binaries without those vendors' terms. Use the Paperclip name or logo as ours. Copy `paperclip-docs` (CC BY-NC-ND 4.0) or unlicensed repos (`clipmart`, `agent-plugins`, `northstar`, `pr-reviewer`) | Same, plus resell Paperclip's hosted services such as the `my.paperclip.app` OAuth broker (ToS section 4). Let customer data reach Paperclip telemetry or feedback endpoints |
| MUST | Keep MIT, OFL (fonts), LGPL (libvips via `sharp`) and BSD notices in the repo, the image and an in-app open source licenses page. Keep libvips replaceable. Offer Debian base-image source | Same notices. Disable or repoint telemetry, announcements and the cloud connector. Run our own OAuth apps. Disclose sub-processors in the DPA |

### 12.3 Ranked license and IP risks

1. Proprietary agent CLIs and SDKs bundled: `@anthropic-ai/claude-code@latest` in `Dockerfile:383`; `@anthropic-ai/claude-agent-sdk` in server `bundleDependencies`; `@github/copilot-*` and `@cursor/sdk` as dependencies. **[LAWYER]**
2. Phone-home on by default: usage telemetry to `telemetry.paperclip.ing` (`packages/shared/src/telemetry/client.ts:14-17`), announcements feed from `pages.paperclip.ing`, feedback-trace export (prompted, includes companyId), CLI update check. Paperclip ToS section 8 allows using detailed telemetry to train models. GDPR and customer-contract exposure.
3. Future relicensing or features held back for EE. Our local clone is shallow (one commit).
4. Trademark: MIT grants no trademark rights. Other parties hold PAPERCLIP marks. Clear "automate.business" as our mark. **[LAWYER]**
5. Vendor OAuth broker and the ToS resale ban.
6. LGPL native code: `heif2jpeg` linking (static or dynamic) unverified. **[LAWYER]**
7. Accidentally copying from non-MIT sibling repos or docs.
8. Copyright holder name mismatch (Paperclip AI vs Paperclip Labs, Inc.).

### 12.4 Compliance checklist

- [ ] Full `--mirror` clone with history and tags; pin `19a7381`; archive source tarball and lockfile.
- [ ] Generate THIRD_PARTY_NOTICES from the lockfile; ship in repo, image and UI licenses page.
- [ ] Remove Claude Code, Claude Agent SDK bundle, Copilot and Cursor from our images; agent CLIs installed per cell under the customer's or our own vendor terms.
- [ ] Rebrand: npm scope `@paperclipai/*`, CLI `paperclipai`, favicons and logos in `ui/public/`, about 6,700 "Paperclip" UI strings, `paperclip.ing` and `paperclip.app` links. Keep copyright lines.
- [ ] Set `PAPERCLIP_TELEMETRY_DISABLED=1`, `PAPERCLIP_ANNOUNCEMENTS_ENABLED=false`, `PAPERCLIP_UPDATE_CHECK=0`; point the feedback backend to our own or hide the setting; set `PAPERCLIP_CLOUD_CONNECTOR_BASE_URL` to ours or disable.
- [ ] Remove the Paperclip notebook app definition and EE upsell links.
- [ ] Own OAuth broker and OAuth apps.
- [ ] LGPL replaceability notes; Debian source offer.
- [ ] Trademark clearance for automate.business.
- [ ] Before each upstream sync, diff `LICENSE` and every package.json `license` field.

## Sources

- Paperclip: https://github.com/paperclipai/paperclip, https://paperclip.ing/docs, https://github.com/paperclipai/paperclip/security/advisories
- Oasis Security write-up: https://www.oasis.security/blog/paperclip-agent-vulnerabilities
- NVD CVE-2026-41679: https://nvd.nist.gov/vuln/detail/cve-2026-41679
- OpenClaw Enterprise: https://openclaw.ai/blog/openclaw-enterprise
- OpenClaw skills risk: https://labs.cloudsecurityalliance.org/wp-content/uploads/2026/07/CSA_research_note_openclaw_clawhub_supply_chain_risk_20260707-csa-styled.pdf
- Activepieces license: https://www.activepieces.com/blog/activepieces-open-source-license-what-is-free-2026.md
- Dify license: https://raw.githubusercontent.com/langgenius/dify/main/LICENSE
- E2B: https://e2b.dev/open-source
- LiteLLM incident: https://docs.litellm.ai/blog/security-update-march-2026
- Supabase AI: https://supabase.com/docs/guides/ai/automatic-embeddings, https://supabase.com/docs/guides/ai/hybrid-search, https://supabase.com/docs/guides/ai/vector-indexes/hnsw-indexes, https://supabase.com/blog/pgvector-0-7-0
- Supabase RLS performance: https://supabase.com/docs/guides/troubleshooting/rls-performance-and-best-practices-Z5Jjwv
- Qdrant multitenancy: https://qdrant.tech/documentation/guides/multiple-partitions/
- Docling: https://github.com/docling-project/docling
- DSPy and GEPA: https://github.com/stanfordnlp/dspy, https://arxiv.org/abs/2507.19457
- promptfoo: https://github.com/promptfoo/promptfoo
