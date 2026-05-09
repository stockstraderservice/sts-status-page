# sts-status-page

A standalone, **off-Vercel** status page for Stock Trader Service. Hosted on GitHub Pages, snapshotted every 5 minutes by GitHub Actions. Deliberately the simplest thing that works: when Vercel is down, this page is still up.

## What it does

1. A scheduled GitHub Action (`*/5 * * * *`) hits `https://www.stockstraderservice.com/api/health/deep`.
2. The JSON response is wrapped with a `_meta.fetchedAt` timestamp and committed to `data/status.json`.
3. GitHub Pages serves `index.html` + `data/status.json` from the `main` branch.
4. The page UI fetches `data/status.json` on load and renders traffic-light status per component.
5. If the snapshot is older than 15 min, the UI shows a stale-data warning — strong signal that either the cron is failing or main infra is down.

## One-time setup

These files were drafted inside `STS-Web-v1/sts-web/tools/status-page/`. To deploy:

### 1. Create the new repo

```powershell
# Create an empty GitHub repo named sts-status-page (public, no README, no .gitignore).
# Then locally:
$src = "C:\Users\sanja\Documents\STS-Web-v1\sts-web\tools\status-page"
$dst = "C:\Users\sanja\Documents\sts-status-page"
robocopy $src $dst /E
cd $dst
git init -b main
git add .
git commit -m "initial scaffold"
git remote add origin https://github.com/stockstraderservice/sts-status-page.git
git push -u origin main
```

### 2. Enable GitHub Pages

In the new repo: **Settings → Pages**.

- Source: `Deploy from a branch`
- Branch: `main` / `/` (root)
- Save.

GitHub will publish to `https://stockstraderservice.github.io/sts-status-page/` within a minute.

### 3. Add the custom domain

The repo already contains a `CNAME` file with `status.stockstraderservice.com`. In **Settings → Pages → Custom domain**, enter the same value and tick "Enforce HTTPS" once the cert is issued (~5-10 min).

### 4. Add the DNS record

On the DNS provider that hosts `stockstraderservice.com`:

```
Type:  CNAME
Name:  status
Value: stockstraderservice.github.io
TTL:   300
```

### 5. Verify

- Visit `https://status.stockstraderservice.com` — should show "No status snapshot available yet" (the placeholder JSON).
- Run the workflow manually: **Actions → Snapshot status → Run workflow**.
- Refresh the status page within 1 min — should now show real data.

## Why GitHub Actions and not a Cloudflare Worker?

- **GitHub Actions free tier:** unlimited minutes for public repos. Cron triggers run forever.
- **No new account or CLI to install.** You already have GitHub.
- **One downside:** scheduled runs can be delayed 5-15 min under platform load. The 15-min stale threshold in the UI accounts for this.

If the delays become a problem, swap to a Cloudflare Worker with a `cron` trigger writing to the same JSON file via the GitHub API (or to Cloudflare KV with a different fetch path).

## What it does NOT do

- Doesn't authenticate. The probe target (`/api/health/deep`) is already public.
- Doesn't store history. Each snapshot overwrites `data/status.json`. Git history of that file is the only retention; if you want trend charts, write to a separate `history/` directory append-only.
- Doesn't page anyone. This is a passive page — it tells users what's up, it doesn't notify the team.
- Doesn't run on Cloudflare Pages. GitHub Pages is enough for our scale.

## Operational checks

- **Action failing:** Actions → Snapshot status → most recent run. Common cause: GitHub revoked write permission on the repo. Verify `Settings → Actions → General → Workflow permissions` is set to "Read and write."
- **Snapshot is empty/null but action succeeds:** `/api/health/deep` returned non-JSON or 5xx. Check Vercel logs.
- **Custom domain unverified:** GitHub takes a few minutes to issue the Let's Encrypt cert after CNAME propagation. If it stalls >30 min, remove and re-add the custom domain in **Settings → Pages**.
