# BraiSEO handoff (read this first)

Last updated: 2026-10-06. Written so any agent (Codex, Claude, a person) can continue the work.
The owner (hans@brai.fi) prefers answers in Finnish, short, with the next action first.

## What BraiSEO is

- A fork of [every-app/open-seo](https://github.com/every-app/open-seo) (MIT, © Ben Senescu), rebranded as BraiSEO.
  Keep `LICENSE` unchanged.
- First version is for Brai's own team only. It is deployed to Cloudflare Workers
  with the upstream self-host flow and sits behind Cloudflare Access (only listed emails can sign in).
- A public SaaS (sign-up, billing) is a later phase.
- Domain to attach later: `seo.brai.build` (owner holds `brai.build`).

## Current state

| Item                                                   | State                                                                                       |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| Rebrand (visible UI name, icons, manifest)             | Done, merged in PR #2                                                                       |
| Deploy workflow `.github/workflows/deploy-braiseo.yml` | Merged in PR #2. Its fix (pipefail + fail when no URL) is in **PR #3, open**                |
| Cloudflare API token + GitHub secrets                  | Done                                                                                        |
| DataForSEO credentials                                 | Work via `DATAFORSEO_LOGIN` + `DATAFORSEO_PASSWORD` secrets (checked against the API in CI) |
| Live deployment                                        | **Not yet.** Last deploy stopped at "Access is not enabled"                                 |
| Upstream sync                                          | **Not started.** 112 upstream commits pending (see below)                                   |

## Next steps, in order

1. **Owner:** enable Cloudflare Access.
   - Open https://one.dash.cloudflare.com/, pick a team name (e.g. `brai`) and the Free plan, or press "Enable Access".
2. **Owner:** merge PR #3. That redeploys automatically. The job summary prints the URL
   (`https://open-seo-selfhost.<subdomain>.workers.dev`) and verifies the Access redirect.
3. **Agent:** upstream sync (see section below). Then close PR #1, which is GitHub's auto "sync fork" PR. Do not merge it as is.
4. Later: custom domain `seo.brai.build`, Finnish UI texts, shut down the old Fly/Supabase stack (see "Old stack").

## Deploy: how it works

- **Trigger:** push to `main` (non-docs) or manual run (Actions → "Deploy BraiSEO (Cloudflare)").
- **Steps:**
  1. Check secrets.
  2. Normalize and verify the DataForSEO key.
  3. `pnpm alchemy cloudflare bootstrap`.
  4. `pnpm deploy:selfhost --yes`.
  5. Verify the Cloudflare Access redirect.
- **Repository secrets:**
  - `CLOUDFLARE_API_TOKEN`: account permissions
    - Edit: Workers Scripts, Workers KV Storage, D1, Workers R2 Storage, Access: Apps and Policies, Access: Organizations…, Secrets Store
    - Read: Account Settings
    - There is no separate "Workflows" permission; Workers Scripts covers it.
  - `CLOUDFLARE_ACCOUNT_ID`
  - `ACCESS_ALLOWED_EMAILS`: comma-separated.
  - DataForSEO: `DATAFORSEO_LOGIN` + `DATAFORSEO_PASSWORD` (API password from https://app.dataforseo.com/api-access). Alternatively `DATAFORSEO_API_KEY` (base64 of `email:password`).
  - Optional: `OPENROUTER_API_KEY`, which enables the SAM AI agent.
- **Infra names:** infra keeps upstream names (`open-seo-*`, stage `selfhost`) on purpose, to keep upstream merges easy.

## Rebrand rules (needed again after every upstream sync)

- Visible text `OpenSEO` → `BraiSEO` in `src/client` and `src/routes` only.
- Skip lines that mention `MCP`, `skill`, `plugin`, `OpenSEO-Audit` (crawler user agent), `openseo.so` links, and code comments.
- Server code (`src/server`) is unchanged.
- The command used:

```bash
files=$(grep -rl "OpenSEO" src/client src/routes --include=*.tsx --include=*.ts | grep -v "\.test\.")
for f in $files; do
  sed -i -E '/MCP|skill|plugin|OpenSEO-Audit|openseo\.so|^\s*(\*|\/\/|\/\*)/!s/OpenSEO/BraiSEO/g' "$f"
done
sed -i '82s/your OpenSEO workspace/your BraiSEO workspace/' src/routes/_authenticated.oauth-consent.tsx  # check line first
sed -i 's/OpenSEO/BraiSEO/g' src/client/lib/error-messages.test.ts src/client/features/integrations/googleLinkError.test.ts
```

- Also: `public/site.webmanifest` name, BraiSEO icons in `public/` (keep ours), and the BraiSEO note at the top of `README.md`.

## Upstream sync procedure

As of 2026-10-06, a trial merge gave **22 conflicts**, all caused by the rebrand text. No conflicts touched the deploy workflow, the database or infra.

```bash
git remote add upstream https://github.com/every-app/open-seo
git fetch upstream main
git checkout -b sync/upstream-YYYY-MM-DD origin/main
git merge upstream/main
# For each conflicted file under src/client or src/routes:
git checkout --theirs <file>     # take upstream's version
# then re-run the rebrand command above, keep our public/ icons and README note
pnpm install --frozen-lockfile
pnpm exec tsc --noEmit
pnpm exec vitest run src/client src/routes
pnpm exec prettier --check <changed files>
```

- Open a PR to `main`. After it merges, close PR #1.
- Repeat about monthly so the gap stays small.

## Old stack (superseded, still running)

- Repo [BraiGAIP/openseo](https://github.com/BraiGAIP/openseo): self-built Next.js + Supabase + Python worker app.
  - Fly apps: `braiseo-web` (https://braiseo-web.fly.dev) and `braiseo-workers`.
  - Supabase project `itxtluifqbxxrealkpfj`.
- PR [BraiGAIP/openseo#5](https://github.com/BraiGAIP/openseo/pull/5) (magic-link redirect fix) is open. It is only needed if that stack stays.
- Repo BraiGAIP/akku-turvan-mestari, branch `claude/openseo-architecture-database-3ut3vj`: early architecture/schema work for the old stack.
- Shut these down once BraiSEO on Cloudflare works. Ask the owner first.

## Session report

A Finnish session report is in the owner's Google Drive: "BraiSEO – istuntoraportti 30.9.2026".
