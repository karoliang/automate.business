# Paperclip feature inventory

Date: 2026-11-20. Upstream: `paperclipai/paperclip` commit `19a738139854718bf9d8c70d9e631fbb28895a9a`. Method: three read-only reviews of the source (UI, server, extensions and tooling). Nothing was installed, built or run. Dispositions (Keep / Hide / Harden / Remove / Decide) are suggestions for the small-business product and feed the design phase (issue #24).

## Summary

- **Size.** server about 1.81M lines, packages 1.34M (paperclip-runner 542k), ui 811k, tests 370k. 50 packages, 132 route files, about 1,400 service files, 198 schema files (about 342 tables), 499 migrations (128 in October 2026 alone), releases about weekly, 13 patched dependencies.
- **UI.** 251 route declarations, about 95 distinct screens, two UI shells (Streamlined, default on, and Classic `*.production.tsx`), 37 instance-wide experimental flags.
- **Cheapest way to hide screens for small businesses:** `PAPERCLIP_HIDDEN_SETTINGS` plus the experimental flags, no code change.
- **Keep (core):** companies, agents, tasks (issues), projects, approvals, decisions inbox, routines and cron, heartbeats, budgets and costs, AI connections, documents, search, S3 storage, skills, plugin SDK (allow-list), tool gateway, connectors catalog, email and Slack/Teams/Telegram channels, audit hub, import and export.
- **Decide (strong fits still flagged):** Areas and Responsibilities dashboard, Conference Room chat, Cases (CRM-like), voice via Speko, Paperclip Runner, memory connectors, plugin-llm-wiki.
- **Remove:** dev-only screens (workspaces, file viewer, plan decomposition, Smoke Lab, GitHub review bots), personal agents (OpenAI Dot, Muse, Grok Bot), nine adapters (cursor-local, cursor-cloud, grok, kimi, pi, hermes, hermes-gateway, openclaw, opencode unless kept), eight of nine sandbox providers, Tailscale broker, all 40 upstream workflows (they break the runner-router rule; `commitperclip-review.yml` runs `pull_request_target` with a secret). About 700k lines can go before touching server or ui.
- **Security items found here, add to the hardening patch set:**
  - UX labs and perf routes at `ui/src/App.tsx:816-819` are mounted with no login and no dev-build gate.
  - Bypass flags forced by adapters: claude-local `dangerouslySkipPermissions` true, cursor-local and hermes `--yolo`, opencode-local tries `sudo -n`.
  - 20 connector methods use `oauthStrategy: "paperclip_cloud_connector"` (all Google apps, GitHub, Asana), which depend on Paperclip Cloud; we need our own OAuth apps.
  - Phone-home: telemetry, feedback upload, announcements, Sentry and PostHog gates, customer-success inspection.
- **Connector gaps for small businesses:** no first-party HubSpot, QuickBooks, Outlook / Microsoft 365 mail and calendar, SharePoint / OneDrive, Salesforce, Square, PayPal, WhatsApp, Mailchimp, WooCommerce.

## 1. UI and user-facing features

Paperclip's UI is very large. `ui/src/App.tsx` has 251 route declarations, and about 150 lines of it are redirects or legacy aliases, so there are roughly 95 distinct screens. There are 89 top-level page files in `ui/src/pages/`. Two whole interface shells exist side by side: Streamlined (`components/Layout`) and Classic (`Layout.production` plus the `*.production.tsx` pages). The `enableStreamlinedUi` flag picks one and defaults to on (`InstanceExperimentalSettings.tsx:225`). About 37 instance-wide experimental flags gate the rest. I read the code only; nothing was installed, built or run.

### Feature inventory

All paths are relative to `ui/src/`. "Flag" means the screen is gated by an experimental flag.

#### Home, attention and chat
| Feature | What it does | Where | Maturity | Dev-only? | Disposition |
|---|---|---|---|---|---|
| Dashboard | Company overview and live run count | `pages/Dashboard.tsx`, `DashboardLive.tsx` | stable | no | Keep, owner's daily home |
| Areas & Responsibilities | Dashboard built around areas you own, agent interviews you | `pages/Responsibilities.tsx`, `ResponsibilitiesGate` | flag `enableResponsibilities` | no | Decide, strong small-business fit |
| Decisions / What needs me | Tasks waiting for your input | `pages/WhatNeedsMe.tsx`, `DecisionQueuePage.tsx` | flag `enableDecisions` (sidebar only, route always on) | no | Keep, core owner inbox |
| Inbox | Mine, recent, unread and blocked views, plus join requests | `pages/Inbox.tsx`, `JoinRequestQueue.tsx` | stable; `enableCombinedInboxTasks` folds it into Tasks | no | Keep, merge with Decisions |
| Approvals | Pending and all approvals, with a detail page | `pages/Approvals.tsx`, `ApprovalDetail.tsx` | stable | no | Harden, money/hire gates |
| Agent chat | One-to-one chats with agents | `pages/AgentChats.tsx`, `AgentChat.tsx` | hook `useAgentChatEnabled` | no | Keep |
| Conference Room | Team-wide chat, live feed, new onboarding | `pages/BoardChat.tsx`, `ConferenceRoomChatGate` | flag `enableConferenceRoomChat` | no | Decide, good for agencies |
| Status Cards | Shared status board with summarizer | `pages/StatusCards/` | flag `enableStatusCards`, "beta" badge | no | Hide, needs Summaries |
| Search | Search across the company | `pages/Search.tsx` | stable | no | Keep |
| Timeline | Gantt-style activity view | `pages/Timeline.tsx` | stable in Classic; redirects in Streamlined | no | Hide, duplicates Audit |

#### Work
| Feature | What it does | Where | Maturity | Dev-only? | Disposition |
|---|---|---|---|---|---|
| Tasks (Issues) list and detail | Task list, chat-style thread, properties | `pages/Issues.tsx`, `IssueDetail.tsx`, `MyIssues.tsx` | stable; `enableClassicTaskInterface` brings back the old layout | partly | Keep, rename for small businesses |
| Projects | Overview, issues, workspaces, configuration, budget tabs | `pages/Projects.tsx`, `ProjectDetail.tsx` | stable | no | Keep |
| Routines | Scheduled or recurring agent work | `pages/Routines.tsx`, `RoutineDetail.tsx` | stable | no | Keep, core automation |
| Goals | Goal tree | `pages/Goals.tsx`, `GoalDetail.tsx` | sidebar link behind `enableGoalsSidebarLink` (legacy) | no | Hide, being evaluated upstream |
| Artifacts | Work outputs | `pages/Artifacts.tsx` | stable; board mount behind Conference Room flag | no | Keep |
| Cases | Long-lived work products such as blog posts | `pages/Cases.tsx`, `CaseDetail.tsx` | flag `enableCases`, "beta" | no | Decide, agency content fit |
| Workspaces / execution workspaces | Per-task worktrees, services, runtime logs | `pages/Workspaces.tsx`, `ExecutionWorkspaceDetail.tsx`, `ProjectWorkspaceDetail.tsx` | flag `enableIsolatedWorkspaces` | yes (git worktree) | Remove, dev-only |
| File viewer | Browse workspace files from a task | flag `enableExperimentalFileViewer` | experimental | yes | Remove |
| External Objects | Shows PR and ticket status from URLs in tasks | flag `enableExternalObjects` | experimental | yes (PRs) | Hide |
| Plan decomposition | Debug history of how plans were split into subtasks | flag `enableIssuePlanDecompositions` | debug | yes | Remove |
| Coordinator tasks / task views | Coordinator workspace views | `pages/CoordinatorTasks.tsx`, `hooks/useCoordinatorWorkspaceEnabled.ts` | flag | no | Decide |

#### Org and agents
| Feature | What it does | Where | Maturity | Dev-only? | Disposition |
|---|---|---|---|---|---|
| Agents list | All, active, paused, error, built-in filters | `pages/Agents.tsx`, `lib/agent-filter-tabs.ts` | stable; built-in tab behind `enableBuiltInAgents` | no | Keep |
| Agent detail | Overview, instructions, skills, runtime, secrets, tools, channels, permissions, API keys, revisions, advanced, audit, runs | `pages/AgentDetail.tsx`, `agent-detail-navigation.ts` | stable | partly (runtime, API keys) | Harden, hide runtime and API keys |
| New agent / hire | Create an agent and pick an adapter | `pages/NewAgent.tsx`, `adapters/*` (claude, codex, cursor, gemini, opencode, pi, kimi, grok, hermes, openclaw, http, process, paperclip-runner) | stable | yes (coding CLIs) | Harden, approved models only |
| Org chart | Reporting tree | `pages/OrgChart.production.tsx` | Classic only; Streamlined redirects to agents | no | Keep, sells "AI staff" |
| Agent budgets and costs | Spend panels per agent | `pages/AgentBudgetsPanel.tsx`, `AgentCostsPanel.tsx` | stable | no | Keep |
| Runner upgrade | Upgrade an agent to Paperclip Runner, offers GitHub Copilot | `components/AgentConfigForm.tsx`, flag `enableNativeRunner` | experimental | yes | Remove, upstream product |
| Personal agents (OpenAI Dot, Meta Muse, Grok Bot) | Pair an outside assistant as an agent | flags `enableOpenAiDot`, `enableMuse`, `enableGrokBot`; `/dot-connect/:id` | experimental | no | Hide |

#### Activity, audit and money
| Feature | What it does | Where | Maturity | Dev-only? | Disposition |
|---|---|---|---|---|---|
| Activity / Audit hub | Commentary, runs, costs, budgets, timeline | `pages/audit/AuditHub.tsx`, `CompanyActivity.tsx` | stable | no | Keep, compliance trail |
| Agent audit "Enterprise" view | Gated by `audit:view_agent_actions` permission | `pages/audit/AuditFeed.tsx:289` | stable, labelled Enterprise | no | Harden, relabel and keep |
| Costs | Provider spend, subscriptions, finance timeline | `pages/Costs.tsx`, `components/FinanceTimelineCard.tsx` | stable | no | Keep, owner cares |

#### Skills, connectors and plugins
| Feature | What it does | Where | Maturity | Dev-only? | Disposition |
|---|---|---|---|---|---|
| Skills library, sources, Studio | Browse, import and author skills | `pages/CompanySkills.tsx`, `SkillSources.tsx`, `SkillStudio.tsx` | stable; `enableBetaSkills` pins beta core skill | no | Keep, harden skill sources |
| Connectors catalog | Browse and connect apps, review queue, permissions | `pages/apps/Browse.tsx`, `AppDetail.tsx`, `AppsReview.tsx` | stable | no | Keep, core value |
| Composio and Arcade aggregators | Third-party tool brokers | `pages/apps/ComposioAppManager.tsx`, `ArcadeDiscoverySetup.tsx` | stable | no | Decide, vendor and data path |
| Google Workspace connector | Gmail, Drive and Calendar | flag `enableGoogleWorkspaceConnector` (Google verification pending) | experimental | no | Harden, high small-business value |
| AWS connector | AWS hosted MCP with short-lived credentials | flag `enableAwsConnector` | experimental | yes | Hide |
| Chat connectors | Slack, Discord, Teams, Telegram, email | `pages/apps/chat/*`, flag `enableChatConnectors` | experimental | no | Harden, Slack and email key |
| GitHub review bots | GitHub App identity that reviews PRs | flag `enableGitHubReviewBots` | experimental | yes | Remove |
| Memory connectors | Mem0, Zep, Supermemory, Cognee, Honcho | flag `enableMemoryConnectors` | experimental | no | Decide |
| MCP gateways and profiles | Self-hosted MCP gateways, tokens, paste-a-config | `pages/apps/gateways/*`, `pages/tools/*` | stable | yes | Hide, power-user only |
| Smoke Lab | Exercises every integration against local fixtures | `pages/tools/SmokeLabTab.tsx`, flag `enableSmokeLab` | debug | yes | Remove |
| Durable app actions (MCP Tasks) | Long-running app actions | flag `enableMcpTasks` | experimental | partly | Decide |
| Fast responses | Short acknowledgements while agents start | flag `enableFastResponses` | experimental | no | Keep |
| Tailscale connector | Private network to the host | `pages/apps/TailscaleConnector.tsx` | stable | yes | Remove, cell handles network |
| Assistant connections / MCP pairing / CLI auth | Pair Claude, ChatGPT or the CLI to an instance | `pages/McpConnect.tsx`, `CliAuth.tsx`, `pages/apps/AssistantConnection.tsx` | stable | partly | Harden, auth surface |
| Plugins | Plugin manager and settings; plugins mount their own pages at `:pluginRoutePath/*` and in settings slots | `pages/PluginManager.tsx`, `PluginPage.tsx`, `CompanySettingsPluginPage.tsx` | stable, plugins can be marked experimental (`api/plugins.ts:142`) | no | Harden, arbitrary UI code |

#### Settings and admin
| Feature | What it does | Where | Maturity | Dev-only? | Disposition |
|---|---|---|---|---|---|
| General settings | Name, logo, board approval for new hires, decision owner, budget owner | `pages/CompanySettings.tsx` | stable | no | Keep |
| Agent observability export | Braintrust or custom OTLP export | `components/observability/AgentObservabilitySettings.tsx` | stable | no | Hide |
| Members, invites, access | People, invites, roles | `pages/CompanyAccess.tsx`, `InviteLanding.tsx`, `InstanceAccess.tsx` | stable | no | Harden, auth and roles |
| Secrets | Company and user secrets, proposals, import from AWS SM, GCP or Vault | `pages/Secrets.tsx`, `pages/secrets/*` | stable | no | Harden, back with Infisical |
| Environments | Where agents run | `pages/CompanyEnvironments.tsx`, flags `enableEnvironments`, `enableManagedSandboxOnly` | experimental | yes | Hide, run managed-only |
| Connections settings | Sign-in method and Paperclip Cloud enrollment | `pages/CompanyConnections.tsx`, `pages/apps/Connections.tsx` | stable | no | Harden, remove Cloud broker |
| Company export and import | Zip export and import | `pages/CompanyExport.tsx`, `CompanyImport.tsx` | stable; import blocked on cloud-managed | no | Harden, data portability |
| Experimental settings | Every flag toggle | `pages/InstanceExperimentalSettings.tsx` | stable | no | Hide, operator only |
| Adapters manager | Turn agent runtimes on and off | `pages/AdapterManager.tsx` | stable | yes | Hide, operator sets |
| Profile settings and user profile | Personal settings | `pages/ProfileSettings.tsx`, `UserProfile.tsx` | stable | no | Keep |
| Companies switcher | Multi-company | `pages/Companies.tsx` | stable | no | Hide, one business per cell |

#### Onboarding and auth
| Feature | What it does | Where | Maturity | Dev-only? | Disposition |
|---|---|---|---|---|---|
| Onboarding wizard | Name the organization, first agent, connect a model, review | `components/OnboardingWizard.tsx`, `onboarding/Stepper.tsx:37` | stable | no | Harden, small-business copy |
| Onboarding chat and setup wizard | Chat-led setup | `components/OnboardingChat.tsx`, `SetupWizard.tsx` | stable | no | Decide |
| First task plan proposal | First single task gets a plan document | flag `enableFirstTaskPlanProposal` | experimental | no | Keep |
| Simplified English | Agents write in ASD-STE100 | flag `enableSimplifiedEnglishInteractions` | experimental | no | Keep, good for owners |
| Auth, board claim, Cloud sign-in, OAuth handoff | Login, claiming the first admin, Paperclip Cloud single sign-on | `pages/Auth.tsx`, `BoardClaim.tsx`, `components/CloudSignIn.tsx`, `pages/apps/PaperclipCloudOAuthHandoff.tsx` | stable | no | Harden; remove the Cloud parts |

#### Developer and labs screens
| Feature | Where | Gate | Disposition |
|---|---|---|---|
| UX labs: bootstrap-setup, responsible-user-denial, cross-issue-collaboration | `App.tsx:817-819` | **none, and mounted outside `CloudAccessGate`** | Remove, open without login |
| `tests/perf/long-thread` (top level) | `App.tsx:816` | **none** (the board-level copy at :347 is DEV-only) | Remove |
| `dev/task-chat-lab` | `App.tsx:425` | `import.meta.env.DEV` | Remove |
| `design-guide` | `App.tsx:456` | none | Remove |
| Developer mode (Honeycomb trace links), server info view, auto-restart dev server | flags `enablePaperclipDeveloperMode`, `enableServerInfoDebugView`, `autoRestartDevServerWhenIdle` | flag | Remove |
| Preview entry points | `wizard-preview-main.tsx`, `connect-*-preview-main.tsx` | build entries | Remove |

### Feature flags and how to toggle them
- **In the UI.** Settings > Experimental (`/company/settings/instance/experimental`) calls `PATCH /api/instance/settings/experimental` (`api/instanceSettings.ts:23`).
- **The 37 keys.** `enableResponsibilities`, `enableAwsConnector`, `enableBetaSkills`, `enableBuiltInAgents`, `enableCases`, `enableChatConnectors`, `enableCombinedInboxTasks`, `enableConferenceRoomChat`, `enableLegacyComposerSubmit`, `enableDecisions`, `enableMcpTasks`, `enableEnvironments`, `enableExternalObjects`, `enableIsolatedWorkspaces`, `enableIsolatedWorkspacesByDefault`, `enableFastResponses`, `enableExperimentalFileViewer`, `enableFirstTaskPlanProposal`, `enableGitHubReviewBots`, `enableGoogleWorkspaceConnector`, `enableGrokBot`, `enableMemoryConnectors`, `enableMuse`, `enableOpenAiDot`, `enableNativeRunner`, `enableSimplifiedEnglishInteractions`, `enableStatusCards`, `enableStreamlinedUi` (default on), `enableSummaries`, `autoRestartDevServerWhenIdle`, `enableManagedSandboxOnly`, `enablePaperclipDeveloperMode`, `enableServerInfoDebugView`, `enableSmokeLab`, `enableIssuePlanDecompositions`, `enableClassicTaskInterface`, `enableGoalsSidebarLink`.
- **Hiding as the operator.** The `PAPERCLIP_HIDDEN_SETTINGS` env var takes a comma list such as `instance.plugins,instance.adapters,company.import,instance.experimental.enableSmokeLab` (`server/src/services/settings-visibility.ts:11`). The UI reads it from `/health` `hiddenSettings` through `HiddenSettingsPageGate`. This is the cheapest way to hide things for small businesses without changing code.
- **Locking values.** A cloud-managed overlay sends `managedKeys`, which locks a toggle (`InstanceExperimentalSettings.tsx:206`).
- **Cloud-managed instances.** If the health response has a `cloud` block (from `PAPERCLIP_CLOUD_API_ORIGIN`), `CloudManagedPageGate` blocks company import.
- **Build-time.** `import.meta.env.DEV` gates only two dev routes.

### Phone-home and upsell points
1. **PostHog.** `components/ProductAnalyticsGate.tsx` loads it only when the server session sends `browserAnalytics` (cloud-managed), and only with consent.
2. **Sentry.** `components/SentryGate.tsx` starts browser error reporting whenever the session carries `sentryDsn`.
3. **Paperclip Cloud broker.** Managed Google sign-in defaults to `https://my.paperclip.app` (`server/src/services/paperclip-cloud-connector.ts:260`). The UI asks the owner to "Enable Paperclip-managed sign-in" (`pages/apps/Connections.tsx:623`), and Cloud then returns encrypted Google credentials to the instance.
4. **Cloud sign-in and stacks.** `CloudSignIn.tsx`, the `/oauth-handoff` route, and `api/cloud.ts` `/cloud/stacks`. `lib/cloudLinks.ts` sends company creation to Cloud's `/stacks/new` instead of the in-app wizard.
5. **Feedback link.** The account menu links to `https://paperclip.ing/feedback` (`components/SidebarAccountMenu.tsx:34`).
6. **Upsell wording.** "Agent audit is a Paperclip Enterprise view" (`pages/audit/AuditFeed.tsx:295`), although it is really a permission check. "Upgrade to Paperclip Runner" with GitHub Copilot (`components/AgentConfigForm.tsx`).
7. **Upstream skill releases.** Beta skills pin releases of the Paperclip core skill. Not traced to where they are fetched from.
8. **Honeycomb.** Trace links appear only in developer mode.

The labs and perf routes at `App.tsx:816-819` need no login and no dev build. Remove them before any public cell goes live.

I did not open every page, so the one-line descriptions come from route names, flag descriptions and headings. I did not check how the server enforces each flag, how Sentry or PostHog are configured on the server, or where the beta skills come from.

## 2. Server capabilities

Paths are relative to the repo root. `svc/` = `server/src/services/`, `rt/` = `server/src/routes/`. Route counts are `router.*` calls. "Flag" means an `INSTANCE_FEATURE_CATALOG` key in `packages/shared/src/feature-catalog.ts` (default off unless noted). "Exp" means listed in `rt/experimental-api-paths.ts`. Scale: 132 route files, about 1,400 service files, 198 schema files, about 342 tables.

### Capabilities

| Capability | What it does | Key files | Route file | Maturity | Dev-only | Disposition |
|---|---|---|---|---|---|---|
| **Org and admin** | | | | | | |
| Companies, members, invites | Tenant root, roles, invites, join requests | svc/companies, company-member-roles, invite-grants | rt/companies (28), access (44) | stable | no | Keep. One company per cell, hide create-company |
| Board auth | better-auth sessions, board API keys, CLI device login, first-admin claim, setup tokens | server/src/auth/, first-admin-claim.ts, svc/device-login-* | rt/auth, access | stable | no | Harden (already reviewed) |
| Instance admin | Settings and flags, DB backups, server metrics | svc/instance-settings, database-backup-* | rt/instance-settings (12), instance-database-backups, instance-server-metrics | stable. `enableOwnerInstanceAdmin` is on for cloud | no | Hide. Operator only |
| Cloud tenant mode | Managed config tiers, tenant sync, warm standby, runtime identity, Paperclip Cloud connector | middleware/cloud-tenant-sync.ts, cloud-control.ts, svc/cloud-*, paperclip-cloud-connector* | rt/cloud | stable, tied to Paperclip Cloud | no | Decide. Reuse the `PAPERCLIP_MANAGED_CONFIG` tier model and drop the vendor wiring |
| Customer-success inspection | Lets the vendor broker read run authority and liveness | svc/customer-success-*, doc/CUSTOMER-SUCCESS-INSPECTION.md | rt/customer-success | gated by env, off | no | Remove. Shaped like vendor access |
| **Agents** | | | | | | |
| Agents and hiring | CRUD, config and instruction revisions, hire approval, permission grants, org chart, avatars | svc/agents, agent-hiring, agent-instruction-*, agent-configuration | rt/agents (87), agent-permission-grants | stable | no | Keep |
| Adapters | claude/codex/gemini/grok/hermes/kimi/opencode/pi/acpx local CLIs, plus http and process | server/src/adapters/, svc/adapter-* | rt/adapters | stable | local CLIs suit dev | Harden. Allow only sandboxed or http adapters in a cell |
| Paperclip Runner (native runtime) | Sandboxed run execution, checkpoints, finalization | svc/native-runtime/, packages/paperclip-runner | rt/native-run-mcp | flag `enableNativeRunner` (on for self-hosted, off for cloud) | no | Decide. Candidate for the core execution path |
| Built-in agents | `briefs`, `learning`, `reflection-coach`, `summarizer`, with bundled routines | svc/built-in-agents.ts, server/src/built-ins/agents/, docs/built-in-agents.md | rt/built-in-agents | flag `enableBuiltInAgents` | no | Hide now. Learning and reflection are good later |
| CEO agent and onboarding | Seeds the CEO agent, onboarding chat, first-task admission | svc/primary-agent, onboarding-*, server/src/onboarding-assets/ceo | rt/primary-agent, onboarding-chat, onboarding-seed | stable | no | Keep. Rewrite copy for small businesses |
| Observability and commentary | Run telemetry, live commentary | svc/agent-observability, agent-commentary | rt/agent-observability, agent-commentary | stable | no | Keep |
| External agents (Muse, OpenAI Dot, Grok Bot) | Personal agent and external runner brokers using mailboxes | svc/muse-*, dot-*, grok-bot/, server/src/modules/external-agents | rt/muse, dot-runner, grok-bot | flags `enableMuse`, `enableOpenAiDot`, `enableGrokBot` | no | Remove |
| **Work** | | | | | | |
| Issues (tasks) | Comments, docs, attachments, relations, labels, questions and forms, plan decomposition, tree holds, watchdogs, privacy | svc/issue*, task-* | rt/issues (117), issue-tree-control | stable. Plan decomposition is flagged | no | Keep |
| Projects and goals | Projects, goals, memberships, project workspaces | svc/projects, goals, project-* | rt/projects (21), goals | stable | no | Keep. Hide the git and repo parts |
| Approvals | Board, hire and issue approvals, with retention | svc/approvals, approval-* | rt/approvals | stable | no | Keep |
| Decisions and queues | Decision inbox, queues seeded by rules, expiry, signing, archive | svc/decision* | rt/decisions, decision-queues | flag `enableDecisions` | no | Decide. Strong fit for owner sign-off |
| Decision model and fast responses | Company LLM connection for background decisions, quick replies | svc/decision-models, fast-response*, doc/decision-models.md | rt/decision-models, fast-responses | fast responses flagged | no | Keep the decision model, hide fast responses |
| Decision training | Captures decision examples, exports JSONL | svc/decision-training.ts | rt/decision-training | unflagged, new | no | Hide |
| Attention and inbox | Attention cards, dismissals, combined inbox | svc/attention*, inbox-* | rt/attention, inbox-* | stable. Combined inbox is flagged | no | Keep |
| Responsibilities dashboard | Areas, bars, metrics, status reports, automation proposals | svc/responsibility-* | rt/responsibility-* (10 files) | Exp, plus flag `enableResponsibilities` | no | Decide. Candidate KPI dashboard |
| Status cards and summaries | Evaluated status cards, summary slots | svc/status-card-*, summary-slots | rt/status-cards, summary-slots | flags `enableStatusCards`, `enableSummaries` | no | Hide |
| Cases, pipelines, workflows | Case facts, case docs, handoffs, pipelines with stages and transitions | svc/case*, workflows.ts, cases-disabled.ts | rt/cases (21), case-handoffs, workflows | Exp, plus flag `enableCases` | no | Decide. CRM-like, compare against building our own |
| Work assessments | Internal reassessment of run status and completion reviews | svc/native-runtime/work-assessments.ts, automatic-completion-reviews.ts | internal only | stable | no | Keep |
| **Scheduling** | | | | | | |
| Heartbeats | Agent wake timers, run queue, dispatch, liveness, watchdogs | svc/heartbeat.ts, heartbeat/, server/src/modules/{wake-queue,run-dispatch,active-run-watchdog} | rt/agents, liveness | stable | no | Keep |
| Routines and triggers | Cron and webhook triggers, routine suggestions, standing checks | svc/routines.ts, cron.ts, routine-* | rt/routines (18), routine-suggestions | stable. Suggestions are Exp | no | Keep |
| Recovery sweeps | Startup passes, issue graph liveness, assigned-work sweep, workspace recovery | svc/recovery/, system-recovery, workspace-recovery* | none | stable | no | Keep |
| Idle sleep | Process sleeps when idle and wakes on deadlines | svc/idle-*, doc/idle-*.md (about 45 docs) | none | stable | no | Keep. Saves cost per cell |
| Deployment notifications | Postgres LISTEN/NOTIFY hints for delivery queues | svc/deployment-notifications.ts | none | stable | no | Keep |
| **Money** | | | | | | |
| Budgets and costs | Policies, reservations, enforcement, incidents, cost events, billing import, invoices | svc/budget*, costs, accounting-*, billing-reconciliation | rt/costs (38) | stable | no | Keep and harden (money) |
| AI connections and routing | Company AI accounts, defaults, subscriptions, OpenRouter, routers | svc/ai-connection*, ai-provider-routing, local-ai-* | rt/ai-connections (17) | stable. `enableAiConnectionRouters` is flagged | no | Keep. Remove local and subscription logins |
| **Communications** | | | | | | |
| Chat channels | Slack, Teams, Discord, Telegram, GitHub, iMessage (Photon). Inbound and outbound, question forms, progress | svc/chat-*, chat-channels/, photon/ | rt/chat-channels (58), board-chat, slack-tools | flag `enableChatConnectors`. `enableAgentChat` is on | no | Keep Slack, Teams and Telegram. Remove GitHub review bots. Decide on iMessage |
| Conference room chat | Multi-agent room | not traced | not traced | flag `enableConferenceRoomChat` | no | Hide |
| Email in and out | AgentMail inboxes, inbound webhook, sends | svc/email-channels.ts, email-connections, agentmail-api | rt/email (15) | unflagged | no | Keep. Harden the vendor choice |
| Voice and phone | Speko voice sessions, inbound calls, reports, replies | svc/voice/ | rt/voice-sessions (17) | unflagged | no | Decide. High value, but locks in one vendor |
| **Content** | | | | | | |
| Documents | Revisions, annotation threads, locks | svc/documents, document-annotations | inside rt/issues and rt/cases | stable | no | Keep |
| Attachments and storage | Local disk or S3, with encryption pool | server/src/storage/, svc/assets | rt/assets, file-resources | stable | no | Keep, on S3 |
| Search | Faceted company search, saved search, extract | svc/company-search*, task-search, doc/SEARCH.md | rt/issues (`/companies/:id/search`) | stable | no | Keep |
| Link previews and external objects | URL unfurling, tracked GitHub objects | svc/link-preview*, external-objects | rt/link-previews | external objects flagged | no | Keep previews, hide external objects |
| Folders | Organizes work into folders | svc/folders | rt/folders | stable | no | Keep |
| **Extensibility** | | | | | | |
| Skills | Company skills, git sources, packages, versions, policy, catalogs | svc/skill-*, company-skills, packages/skills-catalog | rt/company-skills (60), company-skill-policy, mcp-skills, teams-catalog | stable. Beta skills flagged | no | Keep. Harden the supply chain |
| Plugins | Loader, workers, sandbox, jobs, state, plugin DB, webhooks, UI | svc/plugin-* (51), packages/plugins/ | rt/plugins (31), plugin-ui-static, realtime/plugin-api-ws.ts | stable. `plugin-dev-watcher` is dev-only | partly | Harden. Allow-list only |
| Tool access and MCP gateway | Connections, OAuth, profiles, policies, gateway tokens, reviews, MCP apps, tasks, elicitation | svc/tool-access/, tool-*, mcp-* | rt/tool-access (104), tool-gateway (20), mcp-apps, connection-intents | stable. MCP tasks and aggregators flagged | no | Keep (already reviewed) |
| Connectors | Slack, AWS, Supabase, Vercel, Tailscale, 1Password, Braintrust, Browser-Use, Unikraft, Composio | svc/connectors/, doc/connections/ | through tool-access | one flag per connector | no | Decide per connector. Remove dev-infra ones (AWS, Vercel, Railway, Unikraft) |
| Public MCP server | Exposes Paperclip to outside assistants, with OAuth and device flow | svc/public-mcp/ | rt/public-mcp (22) | `enablePublicMcp` is on | no | Harden |
| Memory connectors and Notebook | Honcho, Cognee, Paperclip Notebook | svc/honcho-connection, cognee-connection | not traced | flagged | no | Decide (knowledge already reviewed) |
| Browser use | Cloud browser sessions for agents | svc/browser-use* | rt/browser-use | unflagged | no | Decide |
| **Infra and platform** | | | | | | |
| Environments and workspaces | Execution workspaces, git worktrees, Daytona, Railway and Tailscale envs, custom images, restore points | svc/environment*, execution-workspace*, remote-workspace*, git-*, sandbox-* | rt/environments (21), execution-workspaces, task-workspaces, tailscale-connections | flags `enableEnvironments` and isolated workspaces | built for coding work | Remove the git and dev parts. Keep the sandbox lease core |
| Import and export | Company zip portability, transfer runs | svc/company-portability, company-import-transfers | rt/companies | stable | no | Keep for backup and migration. Harden |
| Secrets | Local encrypted store, AWS Secrets Manager, secret proposals | server/src/secrets/, svc/secret-proposals | rt/secrets (38) | stable | no | Replace with an Infisical provider |
| Workload identity, prompt-injection screening | Per-company JWT keys, input screening | svc/workload-identity, prompt-injection-* | rt/workload-identity, prompt-injection-prevention | stable | no | Keep |
| Telemetry | Product telemetry to telemetry.paperclip.ing, PostHog proxy, Sentry | server/src/telemetry.ts, packages/shared/src/telemetry/, svc/posthog-direct/ | rt/posthog-proxy | on unless `PAPERCLIP_TELEMETRY_DISABLED=1` | no | Remove, or point at our own endpoint |
| Feedback | Thumbs up/down votes and traces. Share upload goes to telemetry.paperclip.ing | svc/feedback*, feedback-share-client.ts | rt/issues (`feedback-votes`, `feedback-traces`) | stable | no | Keep votes. Remove the share upload |
| Announcements | Fetches a card from pages.paperclip.ing | svc/announcement* | rt/announcements | on by default | no | Remove or replace |
| Smoke Lab | Test OAuth server, fixtures, smoke runs | svc/smoke-lab | rt/smoke-lab (15) | Exp, plus flag | yes | Remove |
| Cliplab | SVG avatar animation renderer | packages/shared/src/cliplab/, svc/agent-avatar-worker | rt/agent-avatars | stable | no | Keep (cosmetic) |
| Dev tooling | Dev server status, hot restart, worktree seeding | server/src/dev-*.ts, worktree-*.ts, svc/hot-restart | none | flag `enablePaperclipDeveloperMode` | yes | Remove |
| API discovery | OpenAPI spec, llms.txt | rt/openapi, rt/llms | rt/openapi, rt/llms | stable | no | Keep |
| User preferences | Sidebar, workspace and user preferences, profiles | svc/sidebar-*, workspace-preferences | rt/sidebar-*, workspace-preferences, user-profiles | stable | no | Keep |

Not found: no "capsules" server feature exists (only UI motion CSS and the Composio catalog mention the word). "Learning agent" exists only as the `learning` built-in key. "Reflection" is the `reflection-coach` built-in.

### Schema domain map (`packages/db/src/schema`, about 342 tables)

| Domain | Tables | Examples |
|---|---|---|
| Tool access, MCP, public MCP | ~50 | tool_connections, tool_mcp_gateways, tool_invocations, mcp_oauth_tokens |
| Issues and tasks | ~33 | issues, issue_comments, issue_thread_interactions, issue_watchdogs, labels |
| Chat, voice, email | ~32 | chat_channels (12 in one file), chat_voice_sessions, email_messages |
| Runs, heartbeat, native | ~22 | heartbeat_runs, execution_admissions, native_run_results, work_assessments |
| Core org and auth | ~22 | companies, company_memberships, auth (4), invites, instance_settings |
| Agents | ~19 | agents, agent_api_keys, agent_instruction_revisions, managed_agent_profiles |
| Cases, pipelines, workflows | 19 | cases (8), pipelines, pipeline_stages, pipeline_cases (6) |
| Budgets, costs, accounting | ~17 | budget_policies, cost_events, billing_invoices, ai_subscriptions |
| Workspaces and environments | ~15 | execution_workspaces, execution_workspace_restore_points, environment_leases |
| Approvals and decisions | ~15 | approvals, decisions, decision_queues (6), decision_training_examples |
| Skills | 15 | company_skills (10), company_skill_sources |
| External agents | 15 | muse_* (9), dot_* (4), grok_bot_* (2) |
| Plugins | ~14 | plugins, plugin_jobs, plugin_state, plugin_webhooks |
| Secrets and identity | ~11 | company_secrets, company_secret_versions, secret_access_events |
| Responsibilities, status, summaries | ~11 | responsibilities, status_cards, summary_slots |
| Documents and objects | ~10 | documents, document_annotation_threads, external_objects |
| AI connections and routing | 7 | ai_connection_routing, fast_responses |
| Routines | 6 | routines (5), routine_documents |
| Misc | ~10 | activity_log, feedback_votes, announcement_dismissals, browser_use, smoke_runs |

### Background jobs

The scheduler runs on deadlines, not fixed polling. `svc/work-scheduler.ts` arms a single timer for the next due task. About 75 `register*Work` queues start in `server/src/index.ts`. Examples: budget enforcement, cost accounting, approval and attention retention, decision expiry and continuation, tool review expiry, tool connection health, secret proposal expiry, skill version retention, workspace recovery, environment cleanup, run dispatch, routine execution status, database backup, and join-request retention.

| Job | Interval or retention |
|---|---|
| Heartbeat scheduler (agent timers, issue monitors, routine cron) | `HEARTBEAT_SCHEDULER_INTERVAL_MS`, default 30s, minimum 10s (server/src/config.ts:443) |
| Database backup | 60 min (`PAPERCLIP_DB_BACKUP_INTERVAL_MINUTES`) |
| Execution control reconciliation | 15s (svc/execution-control-deadline.ts) |
| GitHub webhook recovery | 60s (svc/chat-channels/github/webhook-recovery-state.ts) |
| Workspace compaction trash purge | 30 min (svc/execution-workspaces.ts) |
| Native harness backup expiration | 1h |
| Provider trace retention | 24h |
| Queued-run redrive | up to 15 min |
| Workspace recovery observer | 15s |
| Photon receiver poll | 15s |
| Git restore points | kept 90 days |
| Skill packages and runtime assets | kept 30 days |
| Idempotency keys and abandoned workspace files | kept 7 days |
| Leases | About 25 lease-renewal `setInterval` timers (chat, email, Slack, plugins) |

### Env-var feature switches

- **Mode:** `PAPERCLIP_DEPLOYMENT_MODE` (`local_trusted` or `authenticated`), `PAPERCLIP_DEPLOYMENT_EXPOSURE`, `PAPERCLIP_EXECUTION_MODE`, `PAPERCLIP_AUTHZ_MODE`, `PAPERCLIP_RESPONSIBLE_USER_AUTHZ_MODE`, `PAPERCLIP_ISSUE_PRIVACY_MODE`, `PAPERCLIP_API_BRIDGE_MODE`, `PAPERCLIP_GITHUB_AUTH_MODE`, `PAPERCLIP_K8S_EGRESS_MODE`.
- **Cloud:** `PAPERCLIP_MANAGED_CONFIG`, `PAPERCLIP_CLOUD_*` (stack ID, tenant token, warm standby, API origin), `PAPERCLIP_CLOUD_CONNECTOR_*`.
- **Toggles:** `HEARTBEAT_SCHEDULER_ENABLED`, `PAPERCLIP_DB_BACKUP_ENABLED`, `PAPERCLIP_ANNOUNCEMENTS_ENABLED`, `PAPERCLIP_TELEMETRY_DISABLED`, `PAPERCLIP_POSTHOG_BROWSER_ENABLED`, `PAPERCLIP_POSTHOG_REPLAY_ENABLED`, `PAPERCLIP_PUBLIC_MCP_ENABLED`, `PAPERCLIP_ENABLE_OPENAI_DOT`, `PAPERCLIP_ENABLE_COMPANY_DELETION`, `PAPERCLIP_AUTH_DISABLE_SIGN_UP`, `PAPERCLIP_AUTH_RATE_LIMIT_ENABLED`, `PAPERCLIP_SECRETS_STRICT_MODE`, `PAPERCLIP_SECRETS_PROVIDER`, `PAPERCLIP_STORAGE_PROVIDER`, `PAPERCLIP_CUSTOMER_SUCCESS_{AUTHORITY,INSPECTION}_ENABLED`, `PAPERCLIP_VERCEL_CONNECT_ENABLED`, `PAPERCLIP_SLACK_MANAGED_SETUP_ENABLED`, `PAPERCLIP_RUNNER_API_TOOLS_ENABLED`, `PAPERCLIP_SKILL_PACKAGE_STORAGE_ENABLED`, `PAPERCLIP_SKILL_VERSION_RETENTION_ENABLED`, `PAPERCLIP_WORKSPACE_COMPACTION_ENABLED`, `PAPERCLIP_GIT_RESTORE_POINTS_ENABLED`, `PAPERCLIP_GIT_{BASE,CACHE}_EXPIRATION_ENABLED`, `PAPERCLIP_TASK_DIRECTORY_RETENTION_ENABLED`, `PAPERCLIP_NATIVE_*_ENABLED`, `PAPERCLIP_DISABLE_PLUGIN_AUTOBUILD`, `PAPERCLIP_DISABLE_CWD_ENV_FILE`, `PAPERCLIP_AGENT_JWT_DISABLE_LEGACY_FALLBACK`.
- **Instance flags:** about 50 boolean flags in `packages/shared/src/feature-catalog.ts`, tiered `preference`, `managed` or `floor`. Only these are on by default: streamlined UI and nav, apps, public MCP, MCP aggregators, agent chat, artifacts, remote workspace storage, branch reconcile and dirty-quarantine repair, and owner instance admin (cloud only).

## 3. Adapters, plugins, catalogs and tooling

Repo root below is `paperclip/`. Read-only survey. The clone is shallow (1 commit), so commit churn is unavailable. Churn figures come from the migration journal and the release notes.

### Adapters (`packages/adapters/*`, all v0.3.1 unless noted)

Every `*-local` adapter spawns a vendor CLI as a child process on the host or in a sandbox. Each needs that CLI installed plus a provider key or subscription login.

| Item | What it does | Maturity | Creds / network | Disposition |
|---|---|---|---|---|
| claude-local | Claude Code CLI | Mature (16k LOC) | ANTHROPIC_API_KEY or Bedrock AWS keys | **Harden**: `dangerouslySkipPermissions` defaults to `true` (`claude-local/src/server/execute.ts:449`) |
| codex-local | Codex CLI | Mature (23k) | OPENAI/CODEX_API_KEY, writes into shared `~/.codex/skills` | Harden or Decide |
| cursor-local / cursor-cloud | Cursor CLI / Cursor cloud agents | Medium | CURSOR_API_KEY, GH_TOKEN | Remove (developer tool) |
| gemini-local | Gemini CLI | Medium | GEMINI/GOOGLE_API_KEY | Decide |
| grok-local, kimi-local, pi-local | xAI, Moonshot and Pi CLIs | Medium | Vendor keys | Remove |
| opencode-local | OpenCode CLI, multi-provider | Medium | OpenRouter/OpenAI/Anthropic keys; the install path tries `sudo -n` (`opencode-local/src/index.ts:37`) | Decide; harden if kept |
| hermes (local+gateway) | Nous Hermes CLI with `--yolo`, or HTTP/SSE gateway | Medium | Multiple keys, remote gateway | Remove |
| hermes-gateway (0.1.0) | Deprecated shim | Deprecated | none | Remove |
| openclaw-gateway | Remote OpenClaw over ws/wss | Medium | OPENCLAW_TOKEN, network | Remove |

### Plugins (`packages/plugins/*`)

| Item | What it does | Maturity | Creds / network | Disposition |
|---|---|---|---|---|
| sdk (1.0.0) | Worker plus UI SDK, JSON-RPC protocol, test harness, bundlers | Stable API, 22k LOC | none | Keep (the extension point) |
| create-paperclip-plugin | Scaffolder | Small | none | Hide |
| plugin-llm-wiki | Local-folder wiki: managed agent, routines, its own DB migrations | Alpha, 21k LOC | Local FS, LLM spend | Decide (possible SMB "company knowledge" feature, needs FS rework for cells) |
| plugin-workspace-diff | Changes tab for code workspaces | Small | none | Remove (dev-only) |
| paperclip-plugin-fake-sandbox | Deterministic test provider | Test | none | Keep for tests only, Hide |
| examples/* (5) | hello-world, kitchen-sink, file-browser, smoke plugins | Examples | none | Remove |
| sandbox: daytona (0.1.10, 23k LOC) | Remote sandbox | Most mature | DAYTONA_API_KEY | Decide |
| sandbox: kubernetes (0.1.0-alpha.1) | Pods via manifests | Alpha | kubeconfig, bootstrap token | Decide (closest fit for an AWS EKS cell) |
| sandbox: e2b, modal, boxd, createos, exe-dev, novita | Third-party sandboxes | 0.1.x, thin | Vendor API keys | Remove; keep at most one |
| sandbox: cloudflare | Deploys a bridge Worker | 0.1.0 | Cloudflare account | Remove |

Contract: `sandbox-providers/SANDBOX-REQUIREMENTS.md`. Sandbox providers sit outside the pnpm workspace (`pnpm-workspace.yaml`).

### Skills, teams, integrations

| Item | What it does | Maturity | Creds / network | Disposition |
|---|---|---|---|---|
| `skills/` (9) | Runtime agent skills: paperclip, paperclip-board, create-agent, plans-to-tasks, para-memory-files, agentmail, slack, complain, suggestion-box | Core | Paperclip API | Keep the core four. Harden `complain` and `suggestion-box`: they submit "proactively without permission" to `/api/companies/:id/agent-commentary` |
| `skills-releases/` | Versioned paperclip skill snapshots | Core | none | Keep |
| `.agents/skills` (38) | Maintainer skills: release, PR gardening, evals, storybook, honeycomb | Internal | Varies | Remove |
| skills-catalog (16) | Bundled: triage, planning, status, QA, wireframe, github-pr. Optional: ramp (finance), agent-browser, content | Medium | Some need apps | Keep and curate. Drop the software-development skills |
| teams-catalog (5) | core-exec-team, product-engineering, product-design, content-machine, speko phone agent | Medium | speko needs voice | Keep core-exec and content. Rewrite as SMB teams |
| `integrations/assistant-plugins` | Claude/Codex plugin with 3 skills plus `.mcp.json` | Small | Paperclip MCP | Decide |

### MCP servers, runner, misc packages

| Item | What it does | Maturity | Creds / network | Disposition |
|---|---|---|---|---|
| mcp-server | Thin MCP wrapper over the REST API | Small, 1.5k LOC | API key | Keep |
| google-sheets-mcp-server | Sheets v4 over stdio or HTTP | Small | Google service account only | Decide (the remote MCP may cover this) |
| kv-demo-mcp-server | Demo KV plus web UI | Demo | none | Remove |
| tailscale-https-broker | Root-ish host broker for tailnet HTTPS previews, includes C peercred code | Medium | Tailscale operator | Remove (not needed in an AWS cell) |
| paperclip-runner | Native runner protocol (PRP v1). Rust core in `runner/crates` plus TS drivers: codex, opencode, acpx, claude-managed, cursor-cloud, AWS AgentCore | Active, 1,765 files, 542k lines (about 220 fixtures) | Provider keys, WSS to core | Keep but Harden. This is the future execution path, so it is a big maintenance cost |
| paperclip-eval-kernel | Eval matrix orchestrator | 204 LOC | none | Remove with evals |
| `tools/agent-shim` | Go PID-1 shim that reads `/run/paperclip/runtime-command.json` | Small | none | Keep if we use the container images |
| `cli/` (`paperclipai`) | 25 commands: onboard, doctor, db-backup, worktree, env-lab, mcp, service, update | Mature, 46k lines | DB, phones home to `telemetry.paperclip.ing` unless `PAPERCLIP_TELEMETRY_DISABLED=1` | Harden (strip worktree and env-lab, repoint telemetry) |

### Repo tooling

| Item | What it does | Disposition |
|---|---|---|
| `docker/` | Compose, quadlet, `agent-runtime/Dockerfile.{claude,codex,gemini,hermes,opencode,pi}`, daytona-runner, untrusted-review, `ecs-task-definition.json`, `.env.aws.example` | Keep base, claude and ECS. Remove hermes, openclaw, daytona and untrusted-review |
| `evals/promptfoo` | Heartbeat, governance and MCP-gateway prompt evals | Decide (small, cheap to keep) |
| `tests/` (2,217 files) | runner-e2e (1,751), restart-chaos, e2e, storybook-visual, perf, speko | Keep e2e and restart-chaos. Remove the rest |
| `scripts/` (473 files) | CI lanes, release, smoke, scale-ramp, `app-definition-overrides/` (28) | Prune to build, migrate and smoke |
| `patches/` (13) | See below | Keep only those for deps we keep |
| `releases/` (35 notes) | Calendar-versioned notes, v0.2.7 through v2026.1117.0 | Remove (history only) |
| `report/` (1), `maps/` (74 docs, agent-access-map), `announcements/` (in-app banner JSON) | Docs and banners | Remove report. Keep `maps/agent-access-map` as reference for the auth review. Replace announcements with our own feed |

#### Workflows (`.github/workflows`, 40)

Groups: PR and merge-queue (pr, pr-trusted, merge-queue-checks, ci-incremental-qualification, ci-work-benchmark, ci-routing, ci-runner-loss-rerun, ci-profile, ci-test-durations); docker and images (docker, docker-app-check, docker-merge-group, docker-runner-check, runner-image, agent-runtime-images); release (release, release-verify, release-smoke, canary-finalize, cloud-migrator-artifacts, cloud-readiness, refresh-lockfile); runner evals (runner-live-evals, runner-protocol-live-evals, runner-chaos-evals, runner-full-stack-e2e, runner-e2e-testops-registration, runner-windows); chat TestOps (chat-catalog-testops, -case, chat-feature-testops-retained-verification); other (e2e, crash-points, sentry-contract, storybook-deploy, storybook-visual, backup-checkpoint-qualification, pr-review-api-access, pr-merge-review webhook, commitperclip-review).

All 40 use `ubuntu-latest` or Paperclip's AWS runner vars, which violates our runner-router rule. `commitperclip-review.yml` uses `pull_request_target` with a secret. Secrets referenced: ANTHROPIC, OPENAI, OPENROUTER, BRAINTRUST, ALLURE, HONEYCOMB, COMMITPERCLIP, LOCKFILE_BOT_PRIVATE_KEY. **Disposition: Remove all of them and write about 4 new jobs (pr, docker, release, e2e) on `karoliang/runner`.**

#### Patches (`patches/`, wired in `pnpm-workspace.yaml`)

| Patch | Change |
|---|---|
| postgres@3.4.9 | Pipeline limit, buffer and timer leak fixes in connection/queue |
| embedded-postgres@18.1.0-beta.16 | Forces `LC_MESSAGES=C` |
| acpx@0.22.0 | Elicitation request ids, inbound protocol enforcement and routing |
| claude-agent-acp@0.73.0 | Forwards hook lifecycle via `_meta` for audit |
| codex-acp@1.6.2 | Pins `@openai/codex` 0.170.0, MCP elicitation approvals |
| pi-acp@0.0.33 | Isolated context, tool identities (1.5k lines) |
| chat-adapter slack/discord/teams/telegram/github@4.39.0 | Slash commands, native modals (discord, 3.5k lines), attachment caps, error bodies, file links |
| @discordjs/ws@1.2.3 | Connection-epoch reconnect races |
| brace-expansion@5.0.9 | Depth and rewrite caps (ReDoS fix) |

Upstream-bump cost: about 9k patch lines to rebase.

### (1) App definitions (`packages/shared/src/app-definitions/*.json`, 113)

Format: auth types / risk tier (`riskTier` S1 to S4 per method; S4 is highest).

- **productivity (21)**: arcade, asana, browser-use-cloud, calendly, circleback, clickup, coda, composio, executor, fireflies, google-calendar, google-workspace (experimental), jira, linear, make, manufact, miro, monday, ticktick, todoist, zapier. Mostly oauth, S3; clickup, monday and google-workspace are S4.
- **ai (19)**: anthropic, bedrock, chat-completions-api, cognee, github-copilot, google, honcho, hugging-face, local, mem0, messages-api, openai, openrouter, responses-api, supermemory, telem, xai, youcom, zep. Mostly api_key, S3.
- **developer (17)**: 1password, aws (experimental), braintrust, cloudflare, context7, daytona, github-code-review-bot, github, kernel, netlify, pagerduty, postman, railway, sentry, superagent, unikraft, vercel. Many S4.
- **content (14)**: beehiiv, box, cloudinary, ditto, egnyte, google-docs, google-drive, google-slides, notion, oreilly, paperclip-notebook (experimental), sanity, webflow, wix. oauth, S3; the Google apps are S3/S4.
- **communication (11)**: agentmail, discord, gmail, google-chat, google-people, imessage-photon, microsoft-teams, resend, slack, speko (voice), telegram. Chat apps use api_key over chat_sdk; gmail is oauth S3/S4.
- **data (11)**: airtable, candid, clickhouse, exa, firecrawl, google-sheets, google-workspace-search, neon, parallel-search, planetscale, supabase. DB apps are S4.
- **analytics (10)**: bitly, enterpret, gauge, gsc-wizard, local-falcon, mixpanel, posthog, similarweb, tavily, windsor-ai.
- **commerce (8)**: brex, embat, klaviyo, razorpay, shopify (auth none), stripe, ticket-tailor, xero. All S4 except shopify and ticket-tailor (S3).
- **other (2)**: api-key-generic, oauth-generic.

Transports across all methods: 153 mcp_remote, 15 runtime_auth, 9 rest_api, 6 chat_sdk, 2 local_stdio, 1 voice.

**Small-business relevant (~25):** Google (gmail, calendar, drive, docs, sheets, slides, chat, people, workspace-search), microsoft-teams, slack, telegram, xero, stripe, shopify, klaviyo, brex, razorpay, calendly, notion, airtable, asana, monday, wix, webflow, posthog, plus the generic OAuth and API-key methods.

**Gaps:** no first-party HubSpot, QuickBooks, Outlook/M365 mail or calendar, SharePoint/OneDrive, Salesforce, Square, PayPal, WhatsApp, Mailchimp, Zendesk or WooCommerce. Some of these are reachable only via aggregators: `arcade-app-catalog.json` (133 apps, includes HubSpot and Microsoft) and `composio-search-catalog.json`.

**Hard dependency:** 20 methods (all Google apps, GitHub, Asana) use `oauthStrategy: paperclip_cloud_connector`, which brokers OAuth through Paperclip Cloud (`server/src/services/paperclip-cloud-connector.ts`, `PAPERCLIP_CLOUD_CONNECTOR_BASE_URL`). We need our own Google and Microsoft OAuth apps and must replace or disable that strategy.

**Disposition:** curate to about 30 definitions. Remove developer, AI-memory and S4 database apps from the SMB gallery. Add HubSpot, QuickBooks and M365.

### (2) Maintenance-cost signals

- **Packages:** 50 package.json outside fixtures. The workspace covers packages/*, adapters/*, plugins/*, examples, server, ui and cli; sandbox providers sit outside it.
- **Lines of git-tracked text (files / lines):**

| Directory | Files | Lines |
|---|---|---|
| server | 4,959 | 1.81M |
| packages | 4,276 | 1.34M |
| ui | 3,792 | 811k |
| tests | 2,217 | 370k |
| doc | 799 | 255k |
| scripts | 473 | 89k |
| cli | 178 | 46k |
| .github | 122 | 29k |
| .agents | 168 | 23k |
| maps | 74 | 17k |
| docs | 142 | 15k |
| patches | 13 | 9k |

- **Inside packages (lines):** paperclip-runner 542k, db 416k (503 SQL files, mostly migrations), plugins 101k, adapter-utils 93k, shared 89k, adapters 81k.
- **Migration churn:** 498 migrations from 2026-02 to 2026-11 (`packages/db/src/migrations/meta/_journal.json`). Per month: 24, 23, 29, 18, 33, 69, 38, 58, **128 (Oct)**, 78 (Nov to date). That is about 4 per working day. Tracking upstream means a schema merge almost daily, which conflicts with Supabase-managed migrations.
- **Release churn:** 35 release notes, roughly weekly (`releases/v2026.*`). There are 13 dependency patches.

### (3) Delete outright from a fork

Remove these: `packages/adapters/{cursor-local,cursor-cloud,grok-local,kimi-local,pi-local,hermes,hermes-gateway,openclaw-gateway}`, `packages/plugins/examples/*`, `plugin-workspace-diff`, sandbox providers except the one we pick, `kv-demo-mcp-server`, `tailscale-https-broker`, `paperclip-eval-kernel`, `.agents/skills`, `.devin`, `.greptile`, `.codex`, `report/`, `releases/`, `announcements/examples`, all 40 workflows, `tests/{runner-e2e,storybook-visual,perf,speko-*}`, `docker/{hermes-gateway-smoke,openclaw-smoke,daytona-runner,untrusted-review}`, and the matching Dockerfiles. Then drop the patches for removed deps (pi-acp, discord pair, github chat adapter if unused).

The adapters alone are about 30k lines, and `tests/runner-e2e` is the bulk of the 370k test lines. Dropping the matching paperclip-runner drivers and fixtures would cut far more. The server and UI references to the deleted adapters must be unwired too; that step was not checked.

---
Not checked: server/ui wiring of each adapter and plugin, runtime behavior (nothing was run), `.env.aws.example` contents (deliberately not read), or the per-tool risk tiers inside `reviewedTools`. Risk: the claude-local permission-skip default and the Paperclip Cloud OAuth broker are the two items that would break or expose a customer cell if shipped as-is.
