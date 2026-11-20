# Paperclip design system review

Date: 2026-11-20. Upstream commit 19a7381. Read-only review; nothing rendered.

Paperclip keeps its colours, fonts and corner radius in one file, so a token swap is cheap. Its brand and wording, however, are written directly into the screens across many hundreds of lines, so a full rebrand is a large job. Paperclip paths below are relative to the Paperclip repo root.

## 1. How Paperclip's design system works

**`DESIGN.md`** (v0.3, 1247 lines) controls structure, not brand.

- Colour, type and icons are left open on purpose. The rule is "Nothing in `ui/` may hardcode them" (`DESIGN.md:3`).
- Product stance: an operator control panel. Every screen answers "what is happening, does it need me, what do I do" (`:11-13`).
- The 8 principles (`:27-36`):
  1. One component per job.
  2. Tokens are the only source of visual values. Tailwind palette classes count as hardcoded.
  3. Spacing scale still to be decided.
  4. Hierarchy through structure, not borders.
  5. One status vocabulary everywhere.
  6. Machine values (IDs, costs, timestamps) in monospace.
  7. "task" is the canonical term, never issue or ticket.
  8. Agents can change the look by editing tokens and running checks.

**Styling stack** (`ui/package.json`):

- Tailwind v4 (`:104`, `@tailwindcss/vite :98`). There is no tailwind config file (`components.json:7` has `"config": ""`).
- shadcn/ui, style `new-york`, base colour `neutral` (`ui/components.json:3,10`).
- Primitives: `radix-ui` (`:74`, used in `components/ui/alert-dialog.tsx:2`) plus `@base-ui/react` (`:33`).
- class-variance-authority (`:64`; `button.tsx:7`, `badge.tsx:7`), `tailwind-merge`, `clsx`, `cmdk`, `@tailwindcss/typography`.
- Icons: lucide (`components.json:20`, `lucide-react :70`).

**Tokens.** The single source is `ui/src/index.css` (5832 lines) plus `motion-tokens.css` (`index.css:1-2`).

- `@theme inline` block: `index.css:156-205`. `--font-mono` sits in a non-inline `@theme` at `:207`.
- Light values live in `:root` (`:211`, e.g. `--background: oklch(1 0 0)` at `:228`). Dark values live in `.dark` (`:675`).
- Three tiers (`DESIGN.md:21-25`): shadcn semantic, brand (agent gradients `--agent-1a..10b`, `--status-*`), and domain tokens.
- Theming is light or dark only (`ui/src/context/ThemeContext.tsx:11`), applied by toggling the `.dark` class (`:43`).
- There is also a scoped proposal theme, `.design-proposal-theme` (`index.css:4231`, 42 rules), used by Storybook and the sandbox.

**Radius and shadow.** One knob: `--radius: 0.5rem` (`:227`). sm, md, xl, 2xl, 3xl and 4xl are multiples of it (`:193-203`). Shadows are named one-off tokens (`--shadow-task-composer :552`, `--shadow-extract-1 :3234`). There is no shadow scale.

**Typography.**

- InterVariable and Paper Mono are self-hosted from `ui/public/fonts/` (`index.css:110-132`).
- Proposed ramp: 10, 12, 13, 14, 16, 18, 20, 24, 30px, with only weights 400 and 600 (`DESIGN.md` "Proposed typography foundation", after `:346`).
- The colour palette came from a Paper.design dark-mode frame (`:286`).

**Components.** `ui/src/components/ui/` holds 30 primitives (button, card, badge, dialog, sheet, select, tabs, tooltip, toggle-switch and others).

**Storybook.** Version 10.6.0, configured in `ui/storybook/.storybook/`. There are 369 `*.stories.tsx` files in 34 groups, including foundations and design-proposals. That is broad coverage.

**Enforcement.**

- `scripts/check-forbidden-tokens.mjs` and `check-token-gates.mjs` (root `package.json:46-47`) gate hex/rgb/oklch literals and arbitrary Tailwind values.
- The gate only covers `components/**` and `pages/**` (`check-token-gates.mjs:6-9`).
- There is an allowlist with written reasons (`index.css:3430-3460`).

**Brand assets.**

- Favicons, PWA icons and `worktree-favicon*` in `ui/public/`.
- `paperclip-thinking.svg`, used in `ui/src/pages/BoardChat.tsx:895`. `AnimatedPaperclipIcon.tsx:9` draws the same mark inline.
- `ui/src/components/PaperclipLockup.tsx:21` is an inline SVG wordmark.
- Onboarding mascot "pill guy" colours sit in tokens (`index.css:320-327`).
- Adapter logos live in `ui/public/brands/` (claude, codex, opencode, `adapters/`, `apps/`).

**Copy and voice.**

- Principle 7, plus action-named buttons and errors that say what to do.
- i18n is scaffolded (40 locale files) but empty: `en.json` is 9 lines. In practice almost all copy is written inline in the TSX files.

**Accessibility** is strong:

- 3403 `aria-*` attributes and 133 `sr-only` uses.
- 27 reduced-motion blocks in CSS, 23 files handling reduced motion.
- Touch-target floor citing WCAG 2.5.5 (`index.css:2875`).
- Status colours tuned to WCAG AA (`:360`), contrast helpers in `ui/src/lib/color-contrast.ts:91`.
- Two-level focus ring tokens (`--focus-ring-*`, `:242-245`).

## 2. How tightly the brand is coupled

Colour **values** are well tokenised. Colour **choices** and brand **words** are not.

| Item | Approx count |
|---|---|
| Tailwind palette classes (`bg-blue-600` etc.) in non-test tsx | **3397**. Hotspots: `IssueThreadInteractionCard.tsx` (168), `RunTranscriptView.tsx` (126), `AgentDetail.tsx` and `.production.tsx` (90 each) |
| Hex values in non-test ts/tsx | 106. Real colours are in about 12 files, mostly `lib/status-colors.ts` (13) |
| rgb/oklch literals in ts/tsx | 43 |
| Hex / oklch values in `index.css` | 199 / 111 (expected, this is the token file) |
| "Paperclip" in UI ts/tsx lines | 1089, of which about 484 are user-facing strings |
| "Paperclip" in `server/src` | 3862, mostly internal. Some are user-facing, e.g. `routes/tool-access.ts:255,281` ("Returning to Paperclip…") and `services/authorization.ts:219` |
| `.paperclip-*` CSS class names | 54 (internal, can stay) |

Where the brand surfaces:

- `ui/index.html:6-8` (theme-color `#18181b`, title) and the startup error page (`:52,85,100`).
- `ui/public/site.webmanifest` (name, description).
- The favicon set, `PaperclipLockup`, the thinking icon and the mascot.

I found no HTML email templates. The server's HTML strings are OAuth return pages.

**Is there a single theme file to swap?** Partly. Overriding `:root` and `.dark` in `index.css` changes every semantic colour, the radius and the fonts at once. The 3397 palette classes and `status-colors.ts` do not follow it.

## 3. Our current automate.business brand

Sources are the previous automate.business site (repo `ab`), the brand note v1 draft (2026-10-31) and the Story A note (2026-11-09). Paths in this section are relative to the `ab` repo.

**Tokens** (`app/design-system.css:2-27`):

- Dark-first: bg `#0b0d11`, panel `#0f1217`, card `#12151b`, raised `#1c212b`, border `#232833`.
- Text `#eef0f4`, body `#d5d9e1`, muted `#8c93a1`.
- One blue `#3b5bdb` (hover `#496be6`, dot `#6f8bf0`, link `#9db0f5`), amber `#f0b35a`, green `#50a88b`.
- The light tokens are an escape hatch only.
- Radius: card 12px, pill 999px, **button 3px**.

**Type.** Geist 400/500/600/700, self-hosted (`app/fonts/fonts.css`). Body is 15px/1.55 with tabular numbers (`app/design-system.css:33-40`).

**Logo.** A typeset wordmark, "automate**.**business", with only the dot in blue (`marketing/marketing.css:40-49`). No logo file or favicon exists (brand note v1 draft (2026-10-31), `:27,63`).

**Voice** (brand note v1 draft (2026-10-31), `:47-68`):

- Short declarative sentences. No exclamation marks, no em-dashes.
- The human approval gate is stated plainly.
- Honest "Private preview" labels.
- Motif: the "approval block" with draft, awaiting (amber edge) and done (green dot) states (`:57`).
- This brand note is still a DRAFT.

**Positioning.** Story A, confirmed by Karo 2026-11-09: "a business workspace you run by conversation." The site headline is "Set the outcome." (`marketing/index.html:11,27`; Story A note (2026-11-09)).

**Governing rule.** The `ab` repo's `CLAUDE.md:44-46`, "Stock first", puts theme tokens on a low rung of the deviation ladder. Core patches go in `PATCHES.md`, which does not exist yet.

## 4. Adaptation plan

**(a) Token swap (S).** Add one `ui/src/brand.css`, imported last by `index.css`, which keeps a single token root as `DESIGN.md:17` requires. Override `:root` and `.dark` like this:

| Paperclip token | Our value |
|---|---|
| `--background` | `#0b0d11` |
| `--card`, `--popover` | `#12151b` |
| `--muted` | `#0f1217` |
| `--muted-highlight`, `--accent` | `#1c212b` |
| `--border`, `--input` | `#232833` |
| `--foreground` | `#eef0f4` |
| `--muted-foreground` | `#8c93a1` |
| `--primary` | `#3b5bdb` (foreground white) |
| `--ring` | `#9db0f5` |
| `--radius` | `0.75rem` (12px cards) |

Map the status tokens: awaiting approval becomes amber, done becomes green. Make dark the default, since `ThemeContext` falls back to light when nothing is stored.

**(b) Type and logo (S).** Set `--font-sans` to Geist (add an `@font-face` for it). Keep Paper Mono, or use Geist Mono, for machine values. Replace the favicons, manifest, `index.html` title and theme-color, `PaperclipLockup`, the thinking icon and the mascot tokens. We need to commission a favicon or mark first, since none exists.

**(c) Small-business mode (S-M).**

- Set `PAPERCLIP_HIDDEN_SETTINGS` (`server/src/services/settings-visibility.ts:11`). Valid keys are `instance.adapters`, `instance.plugins`, `instance.environments`, `instance.experimental`, `company.secrets.vaults`, `company.export`, `company.import` and others (`packages/shared/src/settings-visibility.ts:35-92`).
- Pin the Streamlined shell on. `ui/src/hooks/useStreamlinedUiEnabled.ts:27` reads instance experimental settings, so this is a config change.
- Hiding dev routes such as runs, transcripts, workspaces and UX labs needs a small route or nav patch in `ui/src/App.tsx`.

**(d) Copy (M-L).** About 484 user-facing "Paperclip" strings, plus terms. JSX text roughly counts "Board" 69, "Organization" 58 (`en.json` uses it), "Company" 19 and "Issue" 11. Upstream is already moving issue to task (Principle 7).

- Recommended: put a brand-name constant and a term map in one module, then patch strings into it, and offer that refactor upstream.
- Our words: Board becomes Owner or Workspace, Organization/Company becomes Business, and Tasks stays.
- The quick way: a build-time string replace over JSX text. It is fragile.

**(e) Restyle vs keep.**

- Restyle Button (the 3px radius conflicts with the one radius knob, so add a `--radius-button` token and a one-line change to `button.tsx:8,26-27`), Card (approval-block states), Badge/status chip and the sidebar.
- Keep dialog, sheet, select, tabs, tooltip, command, the editor and all the accessibility plumbing.

**(f) Staying rebase-friendly.**

- Overlay files only: `brand.css`, a replacement `public/` asset set, a strings module and env config.
- Never mass-edit the 3397 palette classes. Instead, remap Tailwind colour variables (`--color-blue-600` and so on) in `brand.css` to our palette. That re-themes them all from one file.
- Record each core edit in `PATCHES.md`.
- Run `check:tokens` and `check:token-gates` in CI.
- Visual-diff the Storybook foundations stories after each upstream rebase.

**(g) Effort.** (a) S, (b) S plus design time, (c) S-M, (d) M-L, (e) M, (f) S setup with ongoing cost per rebase.

**(h) Risks.**

- Two shells: 20 `*.production.tsx` duplicates (e.g. `AgentDetail.production.tsx`) double every component edit. Choose Streamlined only.
- Size: about 700k lines of ts/tsx in `ui/src`, including tests.
- Upstream churn: `DESIGN.md` says the brand redesign is coming ("will land as token values only"). Their token names may move, which could break our overlay, and the proposal theme is still unstable.
- The copy is hardcoded and i18n is unused, so upstream copy changes will conflict with our string patches.
- Paperclip is light-first and ours is dark-first. Some light-only surfaces, such as the MDXEditor and xterm themes, need checking.

No screen was rendered, so the counts are grep estimates, not visual checks. The token mapping is untested for contrast, so run `lib/color-contrast` or Storybook foundations on it before sign-off.
