# automate.business feature list and scope

Date: 2026-11-20. Status: scope draft for owner review, no code built.
Companion to `2026-11-20-paperclip-foundation-plan.md` (findings, architecture, phases).

## How to read this

Source of each feature:

- **Inherit**: Paperclip already has it; we use it as is (after the hardening patch set).
- **Harden**: Paperclip has it, but we must patch or lock it down before customer use.
- **Build**: we write it.
- **Integrate**: we adopt an existing open-source or SaaS component (license must allow hosted resale).

Priority:

- **MVP**: needed for the first pilot business running on its own cell.
- **V1**: needed before selling the automate.business cloud to paying customers.
- **Later**: valuable, not needed for V1.

Initial target customer (from the existing go-to-market work): owner-led agencies and small service businesses with 5 to 25 staff.

## 1. Business workspace (run the business by conversation)

| Feature | Source | Priority |
|---|---|---|
| Company workspace with goals, projects, issues (work items), comments, documents | Inherit | MVP |
| Agent org chart: roles, reporting lines, per-agent instructions | Inherit | MVP |
| Routines: scheduled, webhook, API and heartbeat triggers | Inherit | MVP |
| Approvals inbox for the owner (approve, reject, comment) | Inherit | MVP |
| Budgets per agent and per company, warn at 80 percent, hard stop, circuit breaker | Inherit | MVP |
| Conversation-first front door: owner talks to the business in plain language, work becomes issues and routines | Build (on Paperclip chat channels and MCP) | MVP |
| Email and chat channels in (Slack, email; WhatsApp later) | Inherit + Build | V1 |
| Simplified small-business UI mode (hide software-dev concepts: repos, branches, worktrees) | Build | V1 |
| Mobile-friendly approvals and daily digest | Build | V1 |
| Voice receptionist | Inherit template, Build hardening | Later |

## 2. Agents as services

| Feature | Source | Priority |
|---|---|---|
| Built-in agent runtimes via adapters (Claude Code, Codex, Hermes, others) | Harden (sandbox only, no host credentials) | MVP |
| HTTP agent-as-a-service template: a business-logic agent running as its own service, called by Paperclip, reporting back via API or MCP | Build (on Paperclip `http` adapter) | MVP |
| Agent SDK and starter kit for custom agents (LangGraph, Mastra or OpenAI Agents SDK) | Build | V1 |
| Business team templates: operations manager, sales and lead follow-up, customer support, bookkeeping and invoicing, marketing and content, client project coordinator | Build | MVP (2 to 3 roles), V1 (full set) |
| Agency starter company template (client intake, proposals, project status, invoicing, content calendar) | Build | MVP |
| Skill library per business (SOP skills, brand voice skill, policy skills), vetted only | Inherit + Harden | MVP |
| Public skill marketplace installs | Out of scope (security) | n/a |

## 3. Integration of business services, apps and SaaS

| Feature | Source | Priority |
|---|---|---|
| Governed tool gateway: every agent tool call checked for policy, approval, rate limit, audit | Inherit | MVP |
| OAuth broker with token refresh, connections scoped to the business, grants per agent or user | Inherit | MVP |
| Connection secrets stored in Infisical | Build (Infisical secret provider) | MVP |
| Google Workspace: Gmail (read, draft, send with approval), Calendar, Drive, Docs, Sheets | Inherit + Build (send) | MVP |
| Slack, Notion, Stripe, Shopify, Xero | Inherit | V1 |
| Microsoft 365: Outlook mail, calendar, OneDrive, SharePoint | Build | V1 |
| HubSpot CRM, QuickBooks | Build | V1 |
| Hosted OAuth apps owned by automate.business (customer never registers developer apps) | Build | V1 |
| Provider event adapters (Gmail push, Stripe webhooks, CRM changes) feeding routines | Build | V1 |
| Internal systems connector kit: customer's own REST APIs, databases (read-only by default), via generic MCP or HTTP | Build (on generic MCP and API-key definitions) | MVP |
| Visual workflow builder for deterministic automations, exposed to agents as MCP | Integrate Activepieces (MIT) | V1 |
| Computer use for apps with no API: Cua desktop sandboxes on AWS, Browser Use for web | Build (Cua sandbox driver) + Inherit (Browser Use) | V1 |
| Zapier, Composio, Arcade bridges | Inherit | Later |

## 4. Business knowledge

| Feature | Source | Priority |
|---|---|---|
| Onboarding wizard: interview the owner, capture services, prices, policies, tone; generate SOP and brand voice skills | Build | MVP |
| Document upload (PDF, DOCX, spreadsheets) with parsing to Markdown | Build (Docling on AWS) | MVP |
| Knowledge base in the business's Supabase: documents, chunks, versions, sources | Build | MVP |
| Hybrid search (keyword plus vector, pgvector HNSW `halfvec`, RRF) as an agent tool | Build | MVP |
| Answers cite their source document and section | Build | MVP |
| Structured business facts (price list, services, staff, clients) as SQL tables agents query directly | Build | MVP |
| Connector sync: Google Drive, Gmail history, Notion, Microsoft 365 | Build | V1 |
| Automatic re-embedding on change (Supabase Automatic Embeddings, pgmq, pg_cron) | Build | V1 |
| Freshness: facts carry valid-from and valid-to; stale content demoted, not deleted | Build | V1 |
| Knowledge access by role (which agents and staff can see which sources) | Build | V1 |
| Per-customer and per-contact memory | Integrate Mem0 (Apache) or Build | Later |
| Graph retrieval across SOPs and customer history | Integrate Cognee or LightRAG | Later |
| Query embedding and answer cache | Build | V1 |

## 5. Self-improvement and optimisation

| Feature | Source | Priority |
|---|---|---|
| Per-run cost and token tracking, per agent, goal, project | Inherit | MVP |
| Owner feedback on agent work (thumbs up or down with reason) kept inside the business, not exported | Harden (disable vendor export) | MVP |
| Reflection loop: coach agent proposes instruction and skill changes, replay check against past work, owner approves, versioned with rollback | Inherit (enable and configure) | V1 |
| Production to eval-case mining: failures and best successes become test cases nightly | Build | V1 |
| Replay gate run with promptfoo before any change applies | Build (extend Paperclip) | V1 |
| Model routing by quality and cost (cheapest model that passes the business's evals) | Build | V1 |
| Monthly business performance report (work done, cost, time saved, approvals) | Build | V1 |
| Offline prompt optimisation (DSPy, GEPA) on high-traffic prompts | Integrate | Later |

## 6. Governance, safety and trust

| Feature | Source | Priority |
|---|---|---|
| Ask-first defaults for outbound money and messaging actions (send email, pay, refund, post publicly, delete) | Build | MVP |
| Per-tool approval policies editable by the owner | Inherit | MVP |
| Append-only audit log shipped outside the cell | Build | MVP |
| Kill switch: pause one agent or the whole business | Inherit | MVP |
| Daily activity digest to the owner | Build | V1 |
| Data export and full deletion for a business | Build | V1 |
| Support access for automate.business staff only with customer-granted, time-limited, logged break-glass | Build | V1 |

## 7. Security and isolation (hardening patch set)

All MVP. Details and code references in the foundation plan, section 7.

| Feature | Source |
|---|---|
| One isolated cell per business: own AWS compute, OS user, S3 prefix, IAM role, Supabase project, Infisical project | Build |
| Agents run only in remote sandboxes with default-deny network egress and an allowlist | Harden + Build (AWS sandbox provider) |
| Agent environment allowlist (no server database URL, auth secrets, cloud or model keys) | Harden |
| Permission-skip and Codex bypass off by default | Harden |
| Agent self-config limited to an allowlist of fields | Harden |
| Local and process adapters and host shell commands disabled for non-admins | Harden |
| Per-cell secrets from Infisical; no master key in agent-readable paths | Build |
| Plugins: vetted only, npm installs disabled | Harden |
| Public signup off, email verification on, API rate limiting, WAF | Harden |
| Same-company check on approval requesters | Harden |
| CI security: CodeQL, secret scanning, dependency review on our fork | Build |
| Independent security review before first customer and on every upstream upgrade | Process |

## 8. Models and BYOK

| Feature | Source | Priority |
|---|---|---|
| Business brings its own model keys or subscriptions per company | Inherit (AI connections) | MVP |
| BYOK keys stored in the business's Infisical project, injected only into its own sandboxes | Build | MVP |
| Spend view per model and per agent | Inherit | MVP |
| Managed models (backlog): models through Amazon Bedrock in each cell, priced as included credits per tier plus overage at a published rate; margin (about 10 percent) set after legal review; pursue the AWS Bedrock partner route and Anthropic approval first. BYOK stays required for any Claude Code agent path. | Build | Later (backlog) |
| Optional model gateway (LiteLLM, pinned by digest) | Integrate | Later |

## 9. automate.business cloud platform (Model B)

| Feature | Source | Priority |
|---|---|---|
| Cell provisioning by script: AWS (compute, network, IAM, S3, sandboxes), Supabase project via Management API, Infisical project | Build | MVP (scripted), V1 (self-serve) |
| Control plane on its own Supabase project: accounts, cell registry, plans, no customer business data | Build | V1 |
| Customer portal: sign up, plan, billing, cell status, BYOK setup, connections | Build (front end on Netlify) | V1 |
| Billing with Stripe | Build | V1 |
| Upgrade pipeline: track upstream Paperclip, rebase patch set, gate, roll cells in waves | Build | V1 |
| Backups and tested restore per cell | Build | V1 |
| Monitoring, alerting, incident runbook | Build | V1 |
| Data residency choice (region per cell) | Build | Later |
| Premium tier with all data inside the customer's own AWS account (RDS in the cell) | Build | Later, owner approval needed |

## 10. Self-run distribution (Model A)

| Feature | Source | Priority |
|---|---|---|
| Installer and infrastructure-as-code for a single business on AWS | Build | V1 |
| Admin and operations guide, upgrade guide | Build | V1 |
| Update channel from our fork with security patches | Build | V1 |
| Other clouds or on-premises | Out of scope | n/a |

## 11. Brand, legal and marketing

| Feature | Source | Priority |
|---|---|---|
| Rebrand of the fork (npm scope, CLI name, UI strings, logos, links); keep copyright lines | Build | MVP |
| Brand overlay: one `brand.css` token overlay (colors, Geist, radius, remapped Tailwind palette), brand-name and terms module, replacement assets, Streamlined shell only. See `docs/research/2026-11-20-paperclip-design-system.md` | Build | MVP |
| Logo mark and favicon (none exists yet) | Design | MVP |
| Vendor telemetry and phone-home disabled (usage telemetry, feedback export, announcements, update check, cloud connector) | Harden | MVP |
| Third-party notices file and in-app open source licenses page | Build | MVP |
| Images ship without proprietary agent CLIs or SDKs (Claude Code, Copilot, Cursor) | Harden | MVP |
| Full upstream mirror with history, pinned release, license diff before each sync | Process | MVP |
| Trademark clearance for automate.business | Owner and legal | V1 |
| Terms of service, privacy policy, DPA naming AWS, Supabase, Infisical and model providers | Owner and legal | V1 |
| Marketing site on Netlify | Build | MVP |

## Out of scope for V1

- Several paying businesses in one shared deployment.
- Public or community skill and plugin marketplaces.
- SSO, SAML and enterprise identity (planned after V1).
- Native mobile apps (mobile web only).
- Hosting our own models.
- Clouds other than AWS, on-premises or air-gapped installs.
- Formal certifications (SOC 2, HIPAA, PCI); design toward SOC 2 controls.
- Multi-region high availability per cell.
- Software-development workflows as a product focus (Paperclip's git and PR features stay hidden for small businesses).

## Owner decisions

Decided 2026-11-20:
- Development database: one Supabase project on the existing plan.
- Production: an AWS server (cell) per business; RDS premium tier approved as a roadmap option.
- Demo account: later, once paid AWS server fees are approved.

- Design before kickoff: the owner does the full design, user flows and end-to-end screen flow once the plan is complete. No build starts before sign-off.

Still open:
1. Production database per business (managed Supabase project or RDS in the cell).
2. Which 2 or 3 agent roles ship in the MVP pilot.
3. Pilot business choice.
4. Paid services for the demo account and pilot cell (AWS, Infisical, sandboxes).
5. Lawyer review of the [LAWYER] items in the foundation plan, section 12.
