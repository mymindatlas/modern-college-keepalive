# modern-college-keepalive

Keeps the Modern College & School Supabase project from going to sleep.

## Why this exists

Supabase pauses free-tier projects after 7 days with no activity. A paused
project means the site's database stops responding until someone logs into the
Supabase dashboard and manually resumes it.

This repo runs a GitHub Actions workflow every 3 days that makes one small
read request to the database. That counts as activity, so the project never
reaches the 7-day mark.

## What it actually does

The workflow sends a single `GET` request to the Supabase REST API asking for
one row from the public `settings` table:

```
GET ${SUPABASE_URL}/rest/v1/settings?select=key&limit=1
```

The `settings` table already has a row-level security (RLS) allowlist policy
that permits anonymous `SELECT` on a fixed set of public keys. Nothing private
is exposed and nothing is written — this is a read of data that is already
public.

The Supabase URL and anon key are stored as repository secrets
(`SUPABASE_URL` and `SUPABASE_ANON_KEY`). No values are hardcoded in this repo.

If the request returns a 2xx status the run passes. Anything else fails the
run, so a broken keepalive shows up as a red X in the Actions tab instead of
silently doing nothing.

## Why it also commits to `log.txt`

GitHub automatically disables scheduled workflows in any repo that has gone
60 days without a commit. Since nobody ever pushes to this repo by hand, the
schedule would quietly switch itself off after about two months.

So after a *successful* ping, the workflow appends one line to `log.txt` and
commits it as `github-actions[bot]`. That commit keeps the repo active and the
schedule alive, and doubles as a plain-text history of pings:

```
2026-08-11T06:00:12Z - ping succeeded (HTTP 200)
```

A failed ping commits nothing — the run just fails, so failures stay visible
as red X's rather than being papered over by a log entry. (These commits don't
retrigger the workflow: pushes made with the built-in `GITHUB_TOKEN` don't
start new workflow runs, so there's no loop.)

## How to check it's working

Go to the **Actions** tab in this repo and look at the run history for the
"Supabase Keepalive" workflow. You should see a green run roughly every
3 days. Click any run to see the HTTP status code and response body it logged.

`log.txt` in this repo is the same history in one place — the last line tells
you when the most recent successful ping happened.

If you see failed runs, the most likely causes are an expired/rotated anon key,
a renamed `settings` table, or a changed RLS policy.

## How to run it manually

Actions tab → select **Supabase Keepalive** in the left sidebar → **Run
workflow** button on the right → **Run workflow**.

Useful after rotating secrets, or to confirm things still work without waiting
for the next scheduled run.

## Note on where this lives

This repo is deliberately separate from the main Modern College & School site
repo. The keepalive has nothing to do with the site's code and shouldn't be
coupled to its deploys, branches, or release cycle — it just needs to keep
running on its own schedule.

**Main site repo:**
https://github.com/modernschool301-create/modern_college_school_site

If you're reading this and don't know what this repo is: it is safe to leave
running. If it is ever deleted or disabled, the Supabase project will pause
after 7 days of no site traffic and someone will need to resume it manually
from the Supabase dashboard.
