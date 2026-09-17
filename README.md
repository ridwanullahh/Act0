Bismillah Ar-Rahman Ar-Raheem. Ash-hadu an laa ilaaha illa-Llah wahdaHu lasharikalaHu, wa ash-hadu anna Muhammadan Abduhu wa Rasooluh. Laa hawla wa laa quwwata illaa biLLAH. Hasbiyallaahu laa ilaaha illaa Huwa, 'alayhi tawakkaltu wa Huwa Rabbul-'Arshil-'Azeem. SubhaanALLAH wa bihamdih, SubhaanALLAHil-'azeem, AlhamduliLLAH, Laa ilaaha illa-ALLAH, wa ALLAHU AKBAR, walaa hawla walaa quwwata illaa biLLAH. Astaghfirullaaha wa atoobu ilayh.


# Act0 — Always-On Keep-Alive Pinger

Bismillah Ar-Rahman Ar-Raheem. This public repository hosts a fully secure,
always-running GitHub Actions pinger that keeps wake-on-traffic hosts 

## How it stays running (not just once)

1. **Self-rescheduling chain** — every run pings the targets, sleeps
   `PING_INTERVAL_SECONDS` (default 330 s ≈ 5.5 min), then dispatches the next
   run via the Actions API. The chain is independent of GitHub's scheduler
   jitter and keeps firing around the clock.
2. **Backstop crons** (`*/5` and `2-59/5`) — re-seed the chain automatically if
   a handoff ever fails.
3. **Repo Activity Keeper** — prevents GitHub's "60 days without repo activity
   disables scheduled workflows" rule from ever switching the pinger off.

A deduplication guard ensures exactly one ping run is ever active, so runs
never pile up. The repo is public, so Actions minutes are **free** — the
always-on chain consumes quota-free minutes only.

## Customization (no code edits required)

Configure everything in **Settings → Secrets and variables → Actions**:

| Name | Type | Purpose | Default |
| --- | --- | --- | --- |
| `PING_URLS` | Variable | Comma/space/newline-separated list of root URLs to ping | the URL hardcoded in the workflow |
| `PING_URLS` | Secret | Same, but hidden if the URL must stay private | — |
| `PING_INTERVAL_SECONDS` | Variable | Seconds between ping rounds (60–540, kept under 10 min) | `330` |

A one-off **manual run** can override URLs via the *Run workflow* button
(`override_urls` input) without touching the configuration.

## Security posture

- No personal access token is stored in the repo or workflow — runs use the
  built-in, least-privilege `GITHUB_TOKEN` (`actions: write` only).
- No third-party Actions, no checkout — only `curl` and the preinstalled `gh`.
- All inputs enter as environment variables (script-injection safe), URLs are
  validated to absolute http(s), curl runs with `--`, and query strings are
  never echoed to logs.

BismiLLAH — SubhaanALLAH wa bihamdih, SubhaanALLAHil-'Azeem.
