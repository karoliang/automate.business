# Security policy

Security is the core of automate.business. This file covers how to report a vulnerability and the rules everyone follows when committing or sharing code in this open source repository.

## Reporting a vulnerability

Do not open a public issue for a vulnerability.

Report it privately through GitHub: **Security > Report a vulnerability** on this repository (private vulnerability reporting is enabled). Include what is affected, how to reproduce it, and the impact. We will acknowledge reports as quickly as we can.

If the issue is in upstream Paperclip rather than our code, we will also report it privately to the Paperclip maintainers through their GitHub security advisories.

## Supported versions

The project is in the research and planning stage. No release is supported yet. This section will list supported versions once the first release ships.

## Rules for committing and sharing code

1. **No secrets in the repository, ever.** API keys, tokens, passwords, private keys, OAuth client secrets, database URLs and model provider keys live in Infisical and are injected at runtime. Commit only `.env.example` files with placeholder values.
2. **Scan before every commit.** Run a staged secret scan (for example `gitleaks protect --staged`) and do not commit if it reports anything. CI repeats the scan on every pull request.
3. **No customer or business data.** Never commit real business documents, emails, customer records, exports, database dumps or logs from a customer cell. Use synthetic fixtures for tests.
4. **No vendor binaries.** Do not commit or bake proprietary software (for example Claude Code, Copilot or Cursor binaries) into images or the repository.
5. **Pull requests only.** All changes reach `main` through a reviewed pull request. No force-pushes and no direct pushes to `main`.
6. **Pin what you depend on.** Lockfiles are committed. New dependencies need a reason in the pull request and a license that allows hosted resale (MIT, Apache-2.0, BSD, ISC). Flag AGPL, SSPL, BSL, ELv2 or non-commercial licenses for review.
7. **Security-sensitive changes get an independent review.** This covers authentication, tenant isolation, secrets handling, agent sandboxing, payments, migrations and CI workflows. The author does not review their own change.
8. **If a secret leaks**, revoke it at the provider first, then remove it from the code and history, then tell the maintainers. Removing it from the code alone is not enough.

## Design commitments

- One isolated cell per business: its own compute, database, storage, secrets and network. No two paying businesses share a deployment.
- Agents run only in sandboxes with default-deny network egress and never receive the server's own credentials.
- Outbound money and messaging actions (send email, pay, refund, post publicly) ask the owner first by default.
- Every agent action is written to an append-only audit log outside the cell.

See `docs/plans/2026-11-20-paperclip-foundation-plan.md` for the full threat findings and hardening plan.
