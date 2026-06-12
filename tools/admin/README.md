# Mobile Donation Admin — Deployment Guide

A one-file Cloudflare Worker that lets you log donations from your phone by uploading a
screenshot. Claude reads the screenshot, pre-fills a form, you tap "Add", and it commits
directly to `data/donations.json` on GitHub. The fundraiser site rebuilds within ~60s.

## What you need before deploying

- **A Cloudflare account** (free) — sign up at [cloudflare.com](https://cloudflare.com)
- **An Anthropic API key** with $5+ credit — from [console.anthropic.com](https://console.anthropic.com)
- **A GitHub fine-grained token** scoped to this repo — instructions below
- **A passcode** of your choosing (e.g. `cabin-pop-42`)
- ~15 minutes total

## Step 1 — Create the GitHub token

1. Go to [github.com/settings/personal-access-tokens](https://github.com/settings/personal-access-tokens) (you may need to sign in as `thesqueezeproject`)
2. Click **Generate new token → Fine-grained**
3. **Token name:** `squeeze-admin-worker`
4. **Expiration:** 1 year (or whatever you prefer)
5. **Resource owner:** `thesqueezeproject`
6. **Repository access:** Select **Only select repositories** → pick `squeeze-project-form`
7. **Permissions → Repository permissions → Contents:** set to **Read and write**
8. Click **Generate token** at the bottom
9. **Copy the token now** — it starts with `github_pat_...` — you can't see it again

## Step 2 — Get the Anthropic API key

1. Go to [console.anthropic.com](https://console.anthropic.com)
2. Sign in (or sign up). Under **Plans & Billing**, add at least $5 of credit.
3. Go to **API Keys** → **Create Key**
4. Name it `squeeze-admin-worker` → Create
5. **Copy the key** — it starts with `sk-ant-...`

## Step 3 — Deploy the Worker

1. Go to [dash.cloudflare.com](https://dash.cloudflare.com) → sidebar **Workers & Pages**
2. Click **Create application** → **Create Worker**
3. Name it `squeeze-admin` (or whatever you want — this becomes your URL)
4. Click **Deploy** (deploys a placeholder; we replace the code next)
5. Once deployed, click **Edit code** (top right)
6. Open `worker.js` in this folder, **copy ALL of its contents**, and paste it into the Cloudflare editor, **replacing the placeholder code**
7. Click **Save and deploy**

## Step 4 — Add the three secrets

Still in the Worker:

1. Click **← Back to worker** (above the editor)
2. Click **Settings** tab → **Variables and Secrets**
3. Add three encrypted variables (click **Add variable** for each, set Type to **Secret**):
   - `ANTHROPIC_API_KEY` = your `sk-ant-...` key from Step 2
   - `GITHUB_TOKEN` = your `github_pat_...` token from Step 1
   - `PASSCODE` = the passcode you chose (e.g. `cabin-pop-42`)
4. Click **Deploy** to apply

## Step 5 — Test it

1. At the top of the Worker page, find the URL (something like `https://squeeze-admin.YOURACCOUNT.workers.dev`)
2. Open it on your phone in any browser
3. Enter your passcode → tap **Unlock**
4. Tap **Choose Screenshot**, pick a recent Venmo notification
5. Wait ~3 seconds. The form should pre-fill with the donor, amount, ambassador, and platform.
6. Review, tap **Add Donation**
7. ~5 seconds later: success screen with the ambassador's new total

If anything's wrong, see **Troubleshooting** below.

## Step 6 — Bookmark it like an app

On iPhone:
1. Open the worker URL in Safari
2. Tap the **Share** button → **Add to Home Screen**
3. Name it "Add Donation" → Add

Now it lives on your home screen like a native app.

## How parsing works

When you upload a screenshot, the Worker sends it to Claude's vision model (currently
Sonnet 4.6) with instructions to extract `donor_name`, `amount`, `platform`, and
`ambassador_key`. Claude returns JSON, the Worker shows it to you for review. You always
get the final say before commit — never auto-submits.

## Cost

- **Cloudflare Workers free tier:** 100,000 requests/day. You'll use ~30/day. **$0.**
- **Anthropic API:** ~$0.005–0.02 per screenshot parsed. At 30 donations/day, **~$0.50/day.**
- **GitHub:** free.
- **Total:** ~$15/month at heavy use; could be pennies/month at light use.

## Security notes

- The Worker is publicly reachable — anyone who finds the URL sees the passcode prompt
- Bearer passcode check stops random visitors from logging fake donations
- The GitHub token is scoped to **just this repo** — if it ever leaked, the blast radius is limited to `data/donations.json` (which you'd notice and could revert)
- Don't share the worker URL publicly; treat it like a password
- To rotate the passcode: just update the `PASSCODE` secret in Cloudflare and re-deploy

## Troubleshooting

**"Wrong passcode" but you're sure it's right:**
- The Worker compares exactly — check for trailing spaces or wrong case
- Look in Cloudflare → Workers → Settings → Variables — is `PASSCODE` actually saved?

**"parse failed" / form not pre-filling:**
- Could be a Claude API error. Open the Worker logs (Cloudflare → Workers → your worker → Logs / Real-time logs) to see the actual error
- Most common: out of credit on Anthropic. Top up.
- Less common: image too big (>5MB). Most phone screenshots are well under this.
- You can still fill in the form by hand — parsing is a convenience, not required.

**"validation failed":**
- Means the resulting JSON wouldn't pass the validator. Usually an upstream race condition
  (you logged from two places at once). Refresh and try again.
- Or check for an inflight commit in GitHub — wait 30s and retry.

**GitHub commit fails:**
- Token may have expired. Generate a new one and update the `GITHUB_TOKEN` secret.
- Token may be scoped wrong. Confirm it has **Contents: Read and write** on the
  `squeeze-project-form` repo.

**Live site shows old numbers after commit:**
- GitHub Pages takes ~30-60s to rebuild. Hard refresh after a minute.
- Cloudflare may cache `/data/donations.json` for up to 10 min — hard refresh (Cmd-Shift-R on desktop, pull-to-refresh on mobile)

## Updating the Worker code

If `worker.js` is ever updated in this repo:
1. Copy the new contents
2. Cloudflare → Workers → your worker → **Edit code**
3. Paste over the existing code
4. **Save and deploy**

No automatic sync between this repo and the Worker — Cloudflare doesn't know about this
repo. (We could wire it up later, but for one file it's overkill.)
