# Inquiry: a process has been running unattended for 37 days on the Paisley EC2 host

**From:** DuelTech operator, via the Paisley session (Perplexity)
**Date:** 2026-09-04
**To:** whichever session owns the DuelTech Opportunity Engine worker
**Purpose:** identify who started this process and decide whether it should keep running. Nothing has been stopped or changed.

---

## Read this first

This document is a **question, not an instruction**. Nothing in the described setup has been altered. No process was signalled, no configuration was edited, no repository outside `goose-worker-paisley` was modified. The only actions taken were read-only inspection commands.

If you are the session that set this up, the useful reply is a plain confirmation plus whatever you know about intent. If you are not, saying so is equally useful — it narrows the search.

---

## What was found

While closing an unrelated round of work on Paisley, a long-running process was observed on Paisley's EC2 host. It has been running continuously for **37 days**, unattended, and is supervised so that it restarts on failure and returns after a reboot.

### Exact identifiers

| Property | Value |
|---|---|
| Host | EC2 `3.14.9.90`, internal `ip-172-31-42-146`, clock is **UTC** |
| Working directory | `/home/ubuntu/dueltech-opportunity-engine` |
| Git remote | `github.com/dgregory3700/dueltech-opportunity-engine` |
| Repo's last commit | `3c49e8c`, **2026-06-29**, "Add Pass 045 upstream input preview artifacts" |
| Command | `npm run worker:connected` → `tsx src/scheduler.ts --mode=connected-dry-run` |
| Supervisor | **PM2 v7.0.1**, God Daemon PID 283187, app name `dueltech-opportunity-engine-worker`, id `0` |
| PM2 app first registered | **2026-06-17T01:14:03Z** |
| Current daemon + process start | **2026-07-29 06:17:05 UTC** |
| PM2 restarts since then | **0** (it has never crashed) |
| Autorestart | enabled, `max_restarts: 5`, from `ecosystem.config.cjs` |
| Survives reboot | **Yes** — `pm2-ubuntu.service` systemd unit is *enabled* and `~/.pm2/dump.pm2` has this app saved |
| Run cadence | every **360 minutes** (`OE_WORKER_INTERVAL_MINUTES=360`), firing at **HH:17 UTC** |
| Reports produced | **324** `OE-CONNECTED-*.md`, first `2026-06-16T19:46:18Z`, latest `2026-09-04T12:17:19Z` |
| Resource use | 36 seconds of CPU total across 37 days; each run takes ~459 ms |

It is the **only** app registered under PM2 on this host.

### What it actually does

Very little, and nothing external. Across all 324 connected reports:

- **0** contain a real `http(s)` source URL
- **282** contain the literal string `Source URL: not available`
- source entries read `Placeholder official workflow documentation` and similar

Its `.env` defines only `OPPORTUNITY_ENGINE_MODE`, `SUPABASE_URL`, `SUPABASE_SECRET_KEY` — **no search-provider or LLM API credential of any kind**. Its own `src/scheduler.ts` says: *"Current mode should be connected-dry-run until live research adapters are approved."*

The reasonable reading is that it is looping over placeholder/sample sources every six hours and emitting a review artifact, without making external research calls. It is not burning API quota, and it uses a Supabase project distinct from Paisley's.

---

## Why we are asking you specifically

The operator's working hypothesis was that this belongs to **Chessa**, on the reasoning that Aven and Calla are both manual start/stop and Chessa is the one with a persistent "keep going" configuration.

**The repository evidence does not support the Chessa attribution.** A recursive case-insensitive search of the entire repo (excluding `node_modules` and `.git`) for `chessa`, `aven`, and `calla` returns **zero matches**. The remote is `dueltech-opportunity-engine`, which is the shared Opportunity Engine control plane rather than any single worker's repo. On this host only two project directories exist: `dueltech-opportunity-engine` and `goose-worker-paisley`.

So the "keep going" behaviour is real, but it comes from **PM2 plus an enabled systemd unit plus a saved process dump on this host** — not from anything worker-branded in the code.

That leaves the actual open question, which repo ownership doesn't answer:

> **Which session or person ran the PM2 setup on this host, and was 37 days of unattended operation the intent?**

The two timestamps that would pin it down are **2026-06-17T01:14Z** (PM2 app first registered) and **2026-07-29 06:17:05 UTC** (current daemon start, i.e. the last time someone started or resurrected it).

---

## Questions we would like answered

1. Did one of your sessions run `pm2 start ecosystem.config.cjs` (or `pm2 save` / `pm2 startup`) in `/home/ubuntu/dueltech-opportunity-engine` on host `3.14.9.90`? If so, roughly when — does **2026-06-17** or **2026-07-29** match your records?
2. Was **indefinite** unattended running the intent, or was this meant to be a bounded test that was never torn down? Note the repo's last commit is 2026-06-29, so it has been running for two months against code that has not changed.
3. Is anything **consuming** the 324 `OE-CONNECTED-*` reports and the `decision-ledger/drafts/*` files it produces, or are they accumulating unread?
4. Do you have a reason it should **keep** running? Paisley's operator is deciding whether to stop it and would rather not break something that another worker depends on.
5. Is `pm2-ubuntu.service` being enabled on this host expected? It means the worker returns automatically after any reboot, which is easy to forget.

---

## Practical notes if a decision is made to stop it

Recorded here because they are non-obvious and were initially got wrong on the Paisley side:

- **Killing PID 283234 will not stop it.** PM2 has `autorestart: true` and will bring it straight back. The correct stop is `pm2 stop dueltech-opportunity-engine-worker` (or `pm2 delete` to deregister).
- **A reboot will not clear it either**, because the systemd unit is enabled and the app is in the saved dump. Fully retiring it also means `pm2 save` after deleting, and possibly `pm2 unstartup systemd`.
- **`pm2` is not on the default non-interactive `PATH`** on this host — it lives under nvm at `~/.nvm/versions/node/v24.16.0/bin/pm2`. A plain `which pm2` returns nothing, which is misleading; the daemon is nevertheless running. This is exactly the check that produced a wrong conclusion on the Paisley side before the process table was inspected directly.
- Restarting later is `pm2 start dueltech-opportunity-engine-worker`. Do **not** use a bare `npm run worker:connected`, which would create a second uncontrolled instance alongside the PM2-managed one.

---

## Scope and honesty notes

- No process was signalled, stopped, or restarted. No file outside `goose-worker-paisley` was modified. No credential values were read or copied, and none appear in this document.
- The conclusion that it makes no external research calls is an **inference** from an absent credential plus zero observed URLs across 324 reports. It is **not** a packet-level observation; no `strace` or netstat capture was taken across a live HH:17 burst.
- 42 of the 324 reports do not contain the string `Placeholder` and were not individually characterized. Three read by hand were "park" decisions recording `source categories: none; evidence: none`, consistent with no real sources — but that is 3 of 42, not a survey.
- The next scheduled run at time of writing is **2026-09-04 18:17 UTC**, if you want to observe it live rather than take the above on trust.
