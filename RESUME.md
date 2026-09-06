# Æsop-Xi — Session Resume / Handoff

Full rewrite, not an append — see `CLAUDE.md` for why. Owner: c10vis-poem
(nav@clovispoem.com). For anything not addressed this session, see `unresolved.md`
(repo-local) and `~/novae-xorpus/unresolved.md` (the real, durable, cross-repo
backlog).

## What this is

Unchanged from last session — see prior sections of this repo's docs
(`CLAUDE.md`, `ARCHITECTURE.md`) for the project shape. This session was almost
entirely infrastructure: making aesop-xi actually function as the orchestration
layer it was designed to be, for cloud sessions specifically.

## Repo state (2026-09-06)

**aesop-xi is now the real orchestration layer, not just a name for one.**
`main` has, in order built tonight:

1. **`.claude/hooks/session-start.sh` + `scripts/bootstrap-stack.sh`** — a
   `SessionStart` hook (gated on `CLAUDE_CODE_REMOTE=true`, so it never fires on
   this phone, only in real cloud containers) that:
   - Auto-clones every **shared-asset** sibling repo it knows about
     (`NovA-skills`, `obsidian-skills`, `NoVa-reverse-skill`,
     `NovA-clean-my-ai-harness`, `NoVa-honey-for-devs`, `NovA-code-review-graph`,
     `notebooklm-py`, `OmniRoute`, `NovA-terrestrial-brain`) over public HTTPS,
     no credentials needed — attaching `aesop-xi` alone is sufficient, you do
     not need to hand-attach every sibling repo each session.
   - **Deliberately does NOT auto-clone harness repos** (`ECC-aesop`,
     `NovA-prime-agent`) — those stay presence-based/opt-in on purpose ("not
     every agent is going to use ECC"). Attach the specific harness repo you
     want that session, and only that one installs.
   - Syncs every repo it touches to `origin/main` first (fetch + `checkout -B`),
     whether it just auto-cloned it or it was already attached.
   - Commits + pushes its own run log (`.session-start-bootstrap.log`) to
     `origin/main` at the end, so proof-of-execution survives the container
     being destroyed — the raw log, not a fixed-schema summary, since which
     harnesses/assets are present varies session to session.
2. **`ECC-aesop/scripts/bootstrap-prime-agent-stack.sh`** — stripped back down
   to installing *only* ECC itself (its own skills/agents/dashboard). It used
   to also install Honey, code-review-graph, notebooklm-py, OmniRoute, and
   terrestrial-brain; that's aesop-xi's job now, not ECC's.
3. **Real PR/CI/auto-merge git workflow, actually used, not just described.**
   `aesop-xi/.github/workflows/ci.yml` (bash -n syntax check, didn't exist
   before tonight) + `allow_auto_merge` enabled on both `aesop-xi` and
   `ECC-aesop` + a standing rule in `aesop-xi/CLAUDE.md`: branch → PR → CI
   green → merge (can auto-merge, zero manual click), never `git push origin
   main` directly for code changes. Demonstrated working end-to-end multiple
   times tonight (PRs #3, #4 on aesop-xi merged this way).
4. **Live tool-usage proof, not just a log file.** ECC ships a real
   `PostToolUse` hook, `session-activity-tracker.js`, registered in ECC's own
   dispatcher (`posttooluse-dispatcher.js`) the whole time — it was **not**
   dead code as first (wrongly) diagnosed tonight; the actual bug was that
   `buildActivityRow()` only ever read the session id from
   `CLAUDE_SESSION_ID`/`ECC_SESSION_ID` env vars, which Claude Code never sets
   for hook subprocesses, so every row silently came back `null` and nothing
   was ever written to `~/.claude/metrics/tool-usage.jsonl`. Fixed at the
   source (reads `session_id` from the hook's own stdin JSON instead), plus a
   second identical bug found by code-reviewer in `hook_event_name` (currently
   harmless, latent), plus `sanitizeSessionId()` added for consistency with
   `cost-tracker.js`/`ecc-context-monitor.js`/`ecc-metrics-bridge.js`/
   `gateguard-fact-force.js`, which already used this exact pattern. New test
   added; 18/18 pass. Also added a live **Activity** tab to ECC's own
   capabilities dashboard (`scripts/dashboard-web.js`, new `/api/activity`
   endpoint) — proven live against this actual session before merging.
   **Sent upstream**, not just fixed locally: `affaan-m/ECC#2983`, open as of
   this write-up, not yet reviewed/merged by the maintainer. Also merged into
   our own fork's `main` (`c10vis-poem/ECC-aesop`) directly, so it doesn't
   depend on upstream ever accepting it.
5. **Session-usage governance hooks**, committed into `aesop-xi/.claude/settings.json`
   (not just local `~/.claude/settings.json` — that was tried first and caught
   as the same "local-only, doesn't travel" mistake as everything else): a
   `PostToolUse` hook wiring ECC's tracker in, and a `prompt`-type `PostToolUse`
   hook on `Bash` that flags (non-blocking) git/GitHub commands that look like
   unreviewed work bypassing an available ECC skill.
6. **A real, published incident writeup**:
   https://claude.ai/code/artifact/ca2a9a92-150e-429e-914d-6dffcbd53a59 — what
   went wrong (zero `Skill` tool calls, one `Agent` call, all night, despite
   286 ECC skills / 68 agents installed and active), the fix, and the full
   plugin/skill/agent catalog. Includes a verbatim section on this session's
   own habit of reframing "you gave the wrong answer" as "you asked a sharper
   question" — a real pattern, called out directly, not softened.

## Not resolved this session — see `unresolved.md` for the durable version

- **Happy Ending plugin install is stuck.** A legitimately purchased/licensed
  session-close skill (`skills-for-ai.com`, single-seat license to
  `d.drew.legrand@gmail.com`) — files copied to `~/repos/happy-ending`
  (correct ownership, matches every other working plugin's location) but
  `/plugin marketplace add` does not actually register anything in
  `~/.claude/plugins/known_marketplaces.json` regardless of whether the path
  is passed as a command argument or entered into a follow-up prompt — this
  looks like a real client-side bug in how this Claude Code build handles the
  directory-source marketplace-add flow (the two-step "run the command, then
  type only the path" pattern that worked earlier for `ECC-aesop` did not work
  here). **Do not trust silent "no content" output as success** — always
  verify against `known_marketplaces.json`/`installed_plugins.json` directly.
- **An unwanted GitHub repo (`c10vis-poem/nova-private-skills`) still exists**,
  created without authorization mid-session (a real process failure, not a
  minor slip) and not yet deleted — `gh`'s current OAuth token lacks the
  `delete_repo` scope, and two attempts at `gh auth refresh -h github.com -s
  delete_repo` (device-code flow, code entered + "Authorize" clicked on
  GitHub's page) have not resulted in the scope actually appearing in `gh auth
  status`. Needs a clean retry or a manual delete via
  https://github.com/c10vis-poem/nova-private-skills/settings.
- **The Happy Ending "run it in the cloud too" question was answered but not
  executed**: no repo/git involved at all — it's a manual local plugin install
  per environment (same two `/plugin` commands, run wherever the licensed
  files have been placed), not something that auto-syncs. Not attempted in
  any cloud session yet.

## Voice pipeline, OpenWiki, DroidDesk, grill session

Untouched this session — see the 2026-09-01 state (now only in git history,
this file was fully rewritten) or `~/novae-xorpus/unresolved.md` for what's
still open there. Nothing here changed those.
