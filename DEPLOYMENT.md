# Deployment — academy.withsoch.com

Static HTML site. No build step, no dependencies. `vercel.json` does clean URLs with
an explicit rewrite (`/:path` -> `/:path.html`) plus `.html` -> clean redirects.

**Do not replace that with `cleanUrls: true`.** Under `cleanUrls`, `vercel build`
renames `index.html` to the path `index` and emits a 308 sending `/index` back to `/` —
which then has nothing to serve. Every subpage works and the site root 404s. A root
rewrite does not beat the rename; doing clean URLs explicitly is what stops it.

## How it deploys

Via **GitHub Actions**, not Vercel's native Git integration — see
`.github/workflows/deploy.yml`.

The reason: Vercel's **Hobby plan cannot connect to a private repository owned by a
GitHub organization**. Private repos under a *personal* GitHub account are fine, which
is why this worked before the repo moved into the `withsoch` org. Rather than pay for
Pro, the workflow builds the site on GitHub's runner and uploads the finished artifact
with `vercel deploy --prebuilt`. Vercel never reads the private repo, so the
restriction never applies.

| Trigger | Result |
|---|---|
| Push to `main` | Production → academy.withsoch.com |
| Push to a feedback branch, or any PR | Preview URL only |
| Manual `workflow_dispatch` | Preview URL only |

## Required GitHub secrets

Set at **Settings → Secrets and variables → Actions** on this repo.

| Secret | Where to get it |
|---|---|
| `VERCEL_TOKEN` | vercel.com/account/tokens, logged in as info@withsoch.com. Scope it to the **With Soch's projects** team, not "personal account". |
| `VERCEL_ORG_ID` | `.vercel/project.json` after `vercel link`, or Vercel → Team Settings → General → Team ID |
| `VERCEL_PROJECT_ID` | `.vercel/project.json` after `vercel link`, or Vercel → Project Settings → General → Project ID |

## One-time setup (already done unless the project is recreated)

```bash
vercel login                      # as info@withsoch.com
vercel project add soch-academy-site --scope with-sochs-projects
vercel link --yes --project soch-academy-site --team with-sochs-projects
cat .vercel/project.json          # -> orgId + projectId for the secrets above
```

Do **not** connect this project to Git in the Vercel dashboard. If you do, Vercel will
try to read the private org repo and the Hobby restriction returns. The project must
stay Git-disconnected and receive deploys only from this workflow.

## Gotcha: deploys must run OUTSIDE the Git work tree

The workflow copies `.vercel` into `RUNNER_TEMP` and deploys from there. That is load
bearing, not tidiness.

The Vercel CLI stamps Git metadata onto any deploy it runs inside a Git work tree —
including a prebuilt deploy with no repo connection. Vercel then requires the **commit
author** to hold a seat on the team. A Hobby team has exactly one seat
(`info@withsoch.com`), so a push authored by anyone else returns:

```
readyState: BLOCKED
seatBlock:  { blockCode: "TEAM_ACCESS_REQUIRED" }
readyStateReason: "Git author <email> must have access to the team ... to create deployments."
```

Two traps when this happens:

- **It hangs, it does not fail.** Vercel marks the deploy BLOCKED immediately but the
  CLI waits ~13 minutes before giving up. A deploy step running far past its usual ~40s
  is this, until proven otherwise.
- **It is invisible in the UI and the Actions log.** The reason only appears in the API:
  `GET https://api.vercel.com/v13/deployments/<url>?teamId=<team>` → `readyStateReason`.

Deploying from a directory with no `.git` stamps no author, so the deploy is attributed
to the token owner — which is what it actually is.

If Soch ever moves to Pro, add every committer to the Vercel team and this whole section
becomes unnecessary.

## Gotcha: preview URLs are SSO-protected

Preview deployments 302 to `vercel.com/sso-api`, so `curl` cannot smoke-test them.
Verify against the production alias `soch-academy-site.vercel.app` instead.

## Rotating the token

Vercel tokens expire. When a deploy fails with a 403 from the CLI, mint a new token and
replace the `VERCEL_TOKEN` secret — nothing else changes.

## DNS

`withsoch.com` nameservers are at **Hostinger** (`ns1/ns2.dns-parking.com`), not Vercel.
The `academy` A-record points at `216.198.79.1`, Vercel's shared anycast IP — identical
for every Vercel account, so moving this project between Vercel accounts needs no
A-record change, only the `_vercel` TXT verification.

## Related

The lead form posts to `https://sochconsulting.app.n8n.cloud/webhook/academy-lead`
(n8n workflow definition kept in `academy-lead-workflow.json`). Test it after any
deploy that touches form markup.
