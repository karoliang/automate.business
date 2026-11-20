# automate.business

An open source, agentic business operating system for small businesses. Agents run as services and work through a business's workflows end to end, while the owner runs the business by conversation and approves what matters.

> **Status: research and planning.** No product code yet. The plan, findings and feature scope are in [`docs/plans`](docs/plans). Open decisions and to-do items are tracked in [issue #22](https://github.com/karoliang/automate.business/issues/22).

## What it is

- **A business workspace you run by conversation.** Goals, projects, work items, routines and approvals, with an org chart of AI agents (operations, sales, support, bookkeeping, marketing) working alongside the owner.
- **Agents as services.** Built-in agent runtimes plus your own business-logic agents running as HTTP services, all coordinated by one control plane with budgets and approvals.
- **Connected to how the business already works.** Google Workspace, Microsoft 365, Slack, Stripe, Xero, HubSpot, QuickBooks, Shopify, your own APIs and databases, and computer use for apps with no API, all through a governed tool gateway.
- **Knows the business.** SOPs, policies, price lists, documents and brand voice loaded into a per-business knowledge base with cited answers.
- **Gets better over time.** Agents propose improvements to their own instructions and skills, re-checked against past work and approved by the owner.
- **Bring your own model keys.** Each business uses its own model provider keys or subscriptions.

## Two ways to use it

1. **Self-run.** A business runs automate.business in its own AWS account and connects its existing workflows, APIs and internal systems.
2. **automate.business cloud.** A small-business operator gets a secure business cloud from us: one isolated cell per business, with its own compute, database, storage and secrets.

## Security first

- One isolated cell per business. No two paying businesses share a deployment.
- Agents run only in sandboxes with default-deny network egress and never see the server's credentials.
- Money and messaging actions ask the owner first by default.
- Append-only audit log of every agent action.
- Secrets live in Infisical, never in code.

See [SECURITY.md](SECURITY.md) for vulnerability reporting and the rules for committing and sharing code.

## Architecture (planned)

| Layer | Choice |
|---|---|
| Control plane | Fork of [Paperclip](https://github.com/paperclipai/paperclip) (MIT), hardened and rebranded |
| Per-business cell | AWS: control plane server, agent sandboxes, computer-use VMs, S3 |
| Database and knowledge base | Supabase Postgres with pgvector hybrid search |
| Secrets and BYOK keys | Infisical |
| Marketing site | Netlify |
| Workflows | Activepieces (MIT) |
| Computer use | Cua (MIT), Browser Use (MIT) |

## Roadmap

| Phase | Scope |
|---|---|
| 0. Verify | Confirm the security findings on a throwaway instance |
| 1. Secure cell | Fork, rebrand, hardening patch set, Infisical, AWS sandbox, one-cell deployment |
| 2. Integrations | Small-business connector set, hosted OAuth apps, event triggers, ask-first actions, computer use |
| 3. Knowledge | Document ingestion, hybrid search, citations, onboarding that turns SOPs into skills |
| 4. Self-improvement | Reflection loop, evals from real work, model routing by quality and cost |
| 5. Hosted cloud | Cell provisioning, billing, portal, backups, upgrades, monitoring |

Details: [foundation plan](docs/plans/2026-11-20-paperclip-foundation-plan.md), [feature scope](docs/plans/2026-11-20-feature-scope.md) and [business lifecycle](docs/plans/2026-11-20-business-lifecycle.md).

## Contributing

Contributions are welcome once code lands. Every change goes through a pull request, and the rules in [SECURITY.md](SECURITY.md) apply: no secrets, no customer data, no vendor binaries, and a staged secret scan before every commit.

## License

Licensed under the [Apache License 2.0](LICENSE). See [NOTICE](NOTICE).

automate.business is planned to be based on Paperclip, which is MIT licensed; its notices will be kept when its code is brought in. automate.business is independent and not affiliated with or endorsed by Paperclip Labs, Inc.
