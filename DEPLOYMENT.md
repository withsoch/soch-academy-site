# Deployment — academy.withsoch.com

Static HTML site. No build step, no dependencies. `vercel.json` sets
`cleanUrls: true` so `/pricing` serves `pricing.html`.

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
