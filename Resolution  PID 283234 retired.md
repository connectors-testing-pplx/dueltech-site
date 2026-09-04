# Resolution: PID 283234 retired — 2026-09-04

**Outcome:** the 37-day unattended opportunity-engine worker on `ubuntu@3.14.9.90` was stopped, deregistered from pm2, and removed from the boot dump, with the operator's explicit go-ahead. Verified gone and reboot-safe.

**Paisley impact: none.** Paisley is not registered under pm2. Its code, gates, thresholds, and corpus were not touched. `goose-worker-paisley` remained at `fd0d52c` with `?? logs/` as the only working-tree difference throughout.

---

## How the ownership question was settled

Three sessions were consulted; the answer came from the third.

| Session | Verdict |
|---|---|
| Paisley (Perplexity) | Found it, characterized it, and **wrongly denied** the Chessa link on a stale-checkout grep |
| Calla (Claude, `goose01`) | Not mine; correctly identified a shared pre-fork ancestor; **wrongly inferred** the Jul 29 event was a reboot |
| Chessa (ChatGPT/Codex) | Deliberately configured for unattended pm2 operation in June, **later retired and no longer authorized**; could not attribute the Paisley-host start to a specific session |

Chessa's account was verified against the repository rather than taken on trust:

- **pm2 config added `1f04f507`, 2026-06-16T20:43:59Z** ("Add PM2 ecosystem config for worker"). The pm2 log shows the app first starting at **20:48:47** — five minutes later. Deliberate setup, corroborated to the minute.
- **Retirement cluster, 2026-07-31 02:02–03:42 UTC**: `618b20b9` and `502b568c` "Retire connected dry-run execution path", `6de8c221` "Retire legacy connected dry-run path", `e36ccec0` "Remove retired connected dry-run script validator", `68d4566c` "Record persisted PM2 stop and live-only run policy", `e189ece7` "Record real-live-run policy and persisted PM2 pause", `5b91208a` "Record connected dry-run retirement handoff". Chessa said "July 30"; 02:02 UTC Jul 31 is 22:02 EDT Jul 30 — same event, timezone difference only.
- Chessa also correctly noted the host distinction: its historical runtime was `ip-172-31-44-79`, not this host.

**The gap this exposes:** pm2 was deliberately restarted **2026-07-29 06:17:06 UTC** (no reboot — host had been up 30 days). The persisted-stop policy was recorded **~44 hours later**. On this host the policy was never applied, so the worker ran against its own retirement policy for 36 of its 37 days. Nobody was watching the box the policy was supposed to govern.

## Ancestry, for the record

All four repos contain root commit `f86e5f1d` (2026-06-14T14:10:12Z): `dueltech-opportunity-engine`, `goose01`, `goose-worker-paisley`, `goose-worker-aven`. Creation dates: 2026-06-14, 2026-06-26, 2026-06-28, 2026-08-21. `3c49e8c` — the commit the stale checkout sits on — is **not** in `goose01` or `goose-worker-paisley`; both were created before it, so it is a parent-only commit made after the children forked.

---

## What was run

```
export PATH="$HOME/.nvm/versions/node/v24.16.0/bin:$PATH"
pm2 stop   dueltech-opportunity-engine-worker
pm2 delete dueltech-opportunity-engine-worker
pm2 save            # << SILENTLY REFUSED, see below
pm2 save --force    # << what actually wrote the empty dump
```

### The `pm2 save` trap — worth remembering

`pm2 save` **declined to write** and exited without error:

```
[PM2][WARN] PM2 is not managing any process, skipping save...
[PM2][WARN] To force saving use: pm2 save --force
```

Because the app had just been deleted, pm2 treated an empty list as nothing worth saving — leaving `~/.pm2/dump.pm2` still holding **1 entry: `dueltech-opportunity-engine-worker`**. `pm2-ubuntu.service` is enabled, so the next reboot would have resurrected the worker and the retirement would have silently undone itself, possibly months later.

Caught only because the verification step read `jq length ~/.pm2/dump.pm2` instead of trusting the exit code. `pm2 save --force` then wrote `0` entries.

**Rule:** after `pm2 delete`, always `pm2 save --force`, and verify the dump length rather than the command's exit status.

### Verified end state

| Check | Result |
|---|---|
| `pm2 list` | empty |
| `~/.pm2/dump.pm2` entries | **0** (was 1 after the failed save) |
| `ps -p 283234` | gone |
| `pgrep -af worker:connected` | none |
| `pgrep -af dueltech-opportunity-engine` | none |
| `goose-worker-paisley` | `fd0d52c`, `?? logs/` only |

Final uptime at stop: **37 days, 11:13:39**. Lifetime output: 324 `OE-CONNECTED-*` reports, zero real source URLs.

### Deliberately NOT done

`pm2 unstartup systemd` was not run. `pm2-ubuntu.service` remains enabled with an empty app list, so nothing restarts; removing the boot hook is a broader host change left to the operator.

---

## Carried forward as an open hazard

The stale checkout at `/home/ubuntu/dueltech-opportunity-engine` is still on disk, and the **current HEAD** of that repo rewired pm2 to a **live** worker: `6509f59c` "Wire PM2 to Chessa live worker" (2026-07-09) and `2c8b9d5e` "Increase Chessa live search cadence and caps" (2026-07-10).

The retired worker was harmless only because its checkout was frozen 1,494 commits back. Anyone who updates that checkout and runs `pm2 start ecosystem.config.cjs` on this host starts a **billable live search worker** on Paisley's box, unprompted. Not Paisley's to remove — worth raising with Chessa.

Recovery of the retired config, if ever needed, must come from `cross-worker/PM2-SNAPSHOT-20260904-pre-retirement.txt`, **not** from the repo's current config.

---

## Corrections this investigation forced on the Paisley side

Recorded because the pattern matters more than the individual errors: in all three, a conclusion was reported that the underlying check did not support.

1. **"No supervisor — no systemd, no cron, no PM2."** Based on `which pm2` returning nothing; pm2 lives on an nvm path absent from the non-interactive `PATH`. The God Daemon (PID 283187) was visible as an unresolved ancestor in my own earlier process-tree output. Fixed in `1b58026`.
2. **"The Chessa attribution is not supported."** Based on a grep of a checkout **1,494 commits stale**. At live HEAD there are 82 code matches for `chessa` and a 2026-09-01 commit clarifying Chessa spending authority. The operator's original hypothesis was right and my denial was wrong. Fixed in `fd0d52c`.
3. **A malformed grep** passed both `-L` and `-l`; `-l` won and its count of 282 was discarded. 42 of 324 reports remain uncharacterized (3 sampled by hand were park decisions recording `source categories: none; evidence: none`).

## Honesty notes

- The "makes no external calls" finding remains an inference from an absent search credential plus zero real URLs — no packet capture or `strace` was taken. It applied to the stale checkout only.
- The 2026-07-29 restart is established as **not a reboot** (boot time plus systemd `ActiveEnterTimestamp`). Who or what triggered it is still unknown, and Chessa could not attribute it.
- No credential values were read, copied, or recorded at any point. The snapshot captures env var names only.
