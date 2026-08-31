# Deployment — academy.withsoch.com

Static HTML site. No build step, no dependencies. `vercel.json` does clean URLs with
an explicit rewrite (`/:path` -> `/:path.html`) plus `.html` -> clean redirects.

**Do not replace that with `cleanUrls: true`.** Under `cleanUrls`, Vercel renames
`index.html` to the path `index` and emits a 308 sending `/index` back to `/` — which
then has nothing to serve. Every subpage works and the site root 404s. A root rewrite
does not beat the rename; doing clean URLs explicitly is what stops it.

## How it deploys

Vercel's native Git integration, on the **info@withsoch.com** account
(team `with-sochs-projects`, project `soch-academy-site`).

Push to `main` -> production. Pull requests -> preview URLs. Nothing else to run.

## History: why this was briefly a GitHub Actions workflow

This repo used to be **private**, and Vercel's Hobby plan refuses to connect to a
private repository owned by a GitHub organization. Private repos under a *personal*
account are fine, which is why the site deployed happily from Ahmad's account for
months and paywalled the instant it moved into `withsoch`.

The workaround was GitHub Actions building the site and uploading the artifact with
`vercel deploy --prebuilt`, so Vercel never read the private repo. **That workflow has
been removed** — the repo is public now, so native Git works and the workaround is
redundant. Keeping both would fire two competing deploys per push.

If this repo is ever made private again, the whole problem returns. The workflow is
recoverable from git history (`.github/workflows/deploy.yml`, removed 2026-08-10)
along with the two traps below, which cost hours to diagnose the first time.

### Trap 1 — deploys must run OUTSIDE the Git work tree

Only relevant if the Actions workflow is ever restored.

The Vercel CLI stamps Git metadata onto any deploy it runs inside a Git work tree —
including a prebuilt deploy with no repo connection. Vercel then requires the **commit
author** to hold a seat on the team. A Hobby team has exactly one seat, so a push
authored by anyone else returns:

```
readyState: BLOCKED
seatBlock:  { blockCode: "TEAM_ACCESS_REQUIRED" }
readyStateReason: "Git author <email> must have access to the team ... to create deployments."
```

Two things make this brutal to diagnose:

- **It hangs, it does not fail.** Vercel marks the deploy BLOCKED immediately but the
  CLI waits ~13 minutes before giving up. A deploy step running far past its usual
  runtime is this, until proven otherwise.
- **It is invisible in the UI and the Actions log.** The reason appears only in the API:
  `GET https://api.vercel.com/v13/deployments/<url>?teamId=<team>` -> `readyStateReason`.

### Trap 2 — preview URLs are SSO-protected

Preview deployments 302 to `vercel.com/sso-api`, so `curl` cannot smoke-test them.
Verify against the production domain instead.

## DNS

`withsoch.com` nameservers are at **Hostinger** (`ns1/ns2.dns-parking.com`), not Vercel.
The `academy` A-record points at `216.198.79.1`, Vercel's shared anycast IP — identical
for every Vercel account, so moving this project between Vercel accounts needs no
A-record change, only a `_vercel` TXT verification.

`_vercel.withsoch.com` holds one TXT record per Vercel-verified subdomain. They stack —
when adding one, **add**, never replace, or you break the other sites' verification.

## Link previews (Open Graph)

`<link rel="icon">` only feeds the browser tab. WhatsApp, Slack, LinkedIn, iMessage and
X ignore it entirely and read Open Graph / Twitter Card meta tags instead — so a page
with a working favicon can still paste as a blank grey card. Every real page now carries
`og:*` + `twitter:*` (added 2026-08-31, when previews were showing no image at all).

Two rules when editing them:

- **`og:image` and `twitter:image` must be absolute URLs** (`https://academy.withsoch.com/...`).
  Relative paths are silently dropped by most scrapers — this is the single most common
  way to break previews.
- **`og:image` must be opaque.** The preview card is `assets/og-image.png`: 1200x630,
  cream (`--cream`) background, `logo-lockup.png` top left, the headline in Poppins
  SemiBold, a line of Instrument Sans body copy, the domain in `--brand`, and
  `logo-mark.png` ghosted at 7% opacity bleeding off the right edge. Rendered at 2x and
  downsampled so the type stays crisp. Do not point `og:image` straight at
  `logo-mark.png`: it is a 512x512 RGBA file, and transparency renders as black on
  WhatsApp and Slack dark mode. Any replacement must be flattened onto a solid
  background, never just stripped of its alpha channel.

`favicon.ico` at the repo root is deliberate — some crawlers and older clients request
`/favicon.ico` blindly, ignoring the `<link>` tags. Vercel serves static files before
the `/:path` -> `/:path.html` rewrite in `vercel.json`, so it is not caught by that.

After changing any of this, previews stay stale until each platform's cache is cleared:
re-scrape at `developers.facebook.com/tools/debug` and `linkedin.com/post-inspector`.
WhatsApp and Slack have no manual purge; append `?v=2` to the pasted link to force a
fresh fetch when testing.

## Related

The lead form posts to `https://sochconsulting.app.n8n.cloud/webhook/academy-lead`
(n8n workflow definition kept in `academy-lead-workflow.json`). Test it after any
deploy that touches form markup.
