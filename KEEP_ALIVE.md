# Keep-Alive System — Design & Operations

Bismillah Ar-Rahman Ar-Raheem. Laa hawla wa laa quwwata illaa biLLAH — this
document explains how the always-on pinger works, how to customize it, and why
it is secure and quota-safe.

## 1. Why this exists

lightning.ai free-plan machines autosleep after a few idle minutes and pay a
slow cold start on the next request. Traffic wakes them — **but only traffic
to the machine's main/root URL; sub-pages do not wake it**. The pinger sends
one lightweight `GET` to every configured root URL every ~5.5 minutes, around
the clock, so the machine never idles long enough to sleep.

## 2. Architecture

| Layer | Mechanism | Role |
| --- | --- | --- |
| Chain | Each run: ping → sleep `PING_INTERVAL_SECONDS` (default 330 s) → `gh workflow run` the next run | Deterministic ~5.5-min cadence, immune to scheduler jitter, runs "Everytime, not just once" |
| Backstop crons | `*/5 * * * *` and `2-59/5 * * * *` | Re-seed the chain within minutes if any handoff fails (e.g. GitHub incident) |
| Dedup guard | Before dispatching, the run checks for other queued/in-progress runs of this workflow | Guarantees exactly one active run — no pile-up, no hammering |
| Concurrency | `concurrency: keep-alive-pinger`, `cancel-in-progress: true` | A fresh backstop run replaces a hung run instead of stacking |
| Activity keeper | `repo-activity-keeper.yml` stamps the repo if idle > 30 days | Defeats GitHub's 60-day auto-disable of schedules |
| Timeouts | `timeout-minutes: 12`, curl `--max-time 45` | Nothing can hang forever or burn minutes |

Cost: this repo is **public**, so standard-runner Actions minutes are free;
the chain's sleeping runner costs nothing against any quota.

## 3. Customization

Everything is runtime-configurable — no code edits, no redeployment:

1. **URLs (comma-separated)** — `Settings → Secrets and variables → Actions →
   Variables` → create/update variable **`PING_URLS`**:
   ```
   https://9001-01m2paqfkv4npk7qny04t0e7q7.cloudspaces.litng.ai/,https://example-2.cloudspaces.litng.ai/
   ```
   Commas, spaces and newlines are all accepted separators. If the URL embeds
   a token you want hidden, put the same list in a **secret** named
   `PING_URLS` instead (secrets take precedence over variables; both fall
   back to the URL hardcoded in `keep-alive.yml`).
2. **Interval** — variable **`PING_INTERVAL_SECONDS`** (default `330`;
   clamped to 60–540 s so the target is never hammered and the gap never
   exceeds 10 minutes).
3. **One-off override** — *Run workflow* → `override_urls` pings alternative
   URLs for that single run only.

Changes take effect on the **next run** (the chain re-reads the variables
every round, so updates land within ~5.5 minutes).

## 4. Security notes (why there is no security issue)

- **No stored credentials.** The workflow uses the repository's ephemeral
  `GITHUB_TOKEN` with only `actions: write` — it cannot read secrets of other
  workflows, cannot push code, and expires with the run. No PAT lives in the
  repo, in variables, or in logs.
- **Injection-proof.** URLs and inputs reach the shell only as environment
  variables; they are never interpolated into shell commands by GitHub
  Expressions. URLs are validated against `https://*|http://*`, and curl is
  always invoked as `curl ... -- "$url"` so a crafted URL cannot become a
  curl flag.
- **No log leakage.** Query strings are stripped before anything is echoed
  (`display="${url%%\?*}"`), and secret-derived URLs are never printed.
- **Least privilege & no supply chain.** No third-party Actions, no checkout
  in the pinger job, `contents: none` — the workflow physically cannot modify
  the repository.
- **Fail-safe.** A failing target never aborts the run (traffic was still
  delivered — which is what matters for waking); failures are logged as
  warnings with the HTTP status for observability.

## 5. Operations quick reference

- Watch live rounds: **Actions → Keep-Alive Pinger** (each run shows a ping
  table in its summary).
- Pause temporarily: *disable* the workflow (the chain stops within one round;
  backstop crons are disabled with it).
- Add a target: append `,https://new-root-url/` to `PING_URLS` — done.
- Move hosts: edit `PING_URLS` — no commit needed.

BismiLLAH — SubhaanALLAH wa bihamdih, SubhaanALLAHil-'Azeem,
AlhamduliLLAH, Laa ilaaha illa-ALLAH, wa ALLAHU AKBAR.
