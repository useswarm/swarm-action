# Swarm UX Test — GitHub Action

Runs a Swarm AI UX test on each pull request and posts ranked findings back to the PR,
with a deep link to the saved run in your Swarm dashboard.

## Setup
1. Create a `cli:run` API key in the Swarm dashboard and add it as the repo secret `SWARM_API_KEY`.
2. Commit `swarm.config.yml` (see `examples/`) describing the goal, personas, build/serve commands, and gating.
3. Add the workflow in `examples/swarm.yml` to `.github/workflows/`.

## Auth options
- **API key (Phase 1):** create a `cli:run` key in the Swarm dashboard, store it as `SWARM_API_KEY`, pass `apiKey: ${{ secrets.SWARM_API_KEY }}`. With this path you pin personas/goal in `swarm.config.yml` (a `generatePersonas` or `swarmId` block).
- **GitHub App (Phase 2, no secret):** install the Swarm GitHub App and link it to your org (Settings → Integrations → Connect GitHub). Then omit `apiKey` and grant the job `id-token: write`; the Action exchanges a GitHub OIDC token for a short-lived key automatically. With this path **personas and the task are configured conversationally** — see below.

## Conversational setup (GitHub App path)
When the App opens the setup PR, `swarm.config.yml` ships **without** personas, so the Action does **not** auto-run on the first PR. Instead the Swarm bot @-mentions you in a PR comment and asks for:
- the **persona type** (audience), the **number of personas**, and the **task** (goal).

Reply with the filled-in `@swarm-ci` template the bot posts; Swarm generates the personas, saves them as the repo's default, and reuses the task on future runs. Re-configure anytime with `@swarm-ci reconfigure`. (The legacy `/swarm …` commands still work.)

### When does it run?
Each connected repo has a **run-mode** you control in **Swarm → Integrations**:
- **Manual** (default): only runs when someone comments `@swarm-ci run` on a PR.
- **Automatic:** runs on every PR open once personas are configured.
- **Off:** never runs — not even on `@swarm-ci run`.

Changing the toggle takes effect immediately — no workflow edit needed.

```yaml
permissions:
  contents: read
  pull-requests: write
  checks: write
  id-token: write   # required for the App (OIDC) auth path
steps:
  - uses: actions/checkout@v4
  - uses: useswarm/swarm-action@v1   # no `apiKey` needed when the App is installed
```

## PR reporting & identity
The Action posts a sticky results comment and a **"Swarm UX"** check run on the PR:
- The check is created **in progress** as soon as the run starts (so `@swarm-ci run`
  comment-triggered runs are visible from the PR immediately) and is updated to the
  final conclusion when the run completes — one check, not two.
- With the **GitHub App (OIDC) auth path**, both the comment and the check are posted
  via the Swarm App identity (the same bot that handles `@swarm-ci`), through the Swarm
  API. If that call ever fails, the Action falls back to posting with the workflow's
  `GITHUB_TOKEN` ("github-actions" identity) so reports are never lost.
- With the **API key path**, posting always uses the workflow's `GITHUB_TOKEN`.

## Rendering
By default the Action builds + starts your app in the runner (the `build` block in
`swarm.config.yml`) and exposes it via a Cloudflare tunnel. To test an existing preview
deployment instead, pass its URL via the `preview-url` action input or set `preview.url`
in the config — there is no automatic preview-deployment detection.

## Config
See `examples/swarm.config.yml` for the full schema.

Under `gate.failOn`, `judgeVerdict` is a list of tokens matched case-insensitively as
whole words of the judge's plain-English verdict (which is prose, e.g. `"No — checkout is
broken."`). List failure-signalling words such as `"no"` or `"broken"`; any match fails the
check (`"no"` matches `"No — …"` but not `"notable"`).
