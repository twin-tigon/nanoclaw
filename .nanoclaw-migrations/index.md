# NanoClaw Migration Guide

Generated: 2026-08-15T00:53:22Z
Last migrated: 2026-08-31T23:01:00Z
Base (pre-2026-08-31 migration): 1b08b58fcd80a2019b5cf012904798901b087fc8
Base (current, post-migration): 5ab77545
HEAD before this migration: d7bd82fc (27d15c64 with guide update)
HEAD after this migration: b9c534c0
Upstream at generation: 249cf983
Upstream as of this migration: 5ab77545 (qwibitai/nanoclaw, remote `upstream`, branch `main`)

## 2026-08-31 migration outcome (second pass)

Second migration pass, from the 249cf983 base (established by the first pass
above) to upstream `5ab77545` — 371 upstream commits, spanning a full
container-runtime rewrite (session driver seam: `src/drivers/`, replacing
`buildContainerArgs`), an agent-mailbox storage seam, async central-DB access,
a host lifecycle registry, and the CLAUDE.md→project-doc-compose rewrite. A
plain `git merge` was attempted first via `/update-nanoclaw`'s new
transactional flow and abandoned cleanly (no live mutation) once
`container-runner.ts`'s conflict showed the old `buildContainerArgs` function
had been structurally replaced — hand-resolving it wouldn't have been a merge,
it would have been re-deriving the new architecture from scratch.

What carried forward, reapplied fresh on the new base in `.upgrade-worktree/`:

- **`add-whatsapp` (v7) + `add-emacs`** — refetched from `channels` branch,
  unchanged mechanically from the first pass.
- **`add-ollama-tool`** — reapplied per its own current `SKILL.md`, which had
  *already* been updated upstream for the driver seam (`composeSessionSpec`
  instead of `buildContainerArgs`, `ollamaEnv()` instead of `ollamaEnvArgs()`,
  the stderr log-level branch now in `src/drivers/docker-driver.ts`'s
  `DockerHandle.start()` instead of `container-runner.ts`). No manual
  architecture translation needed — the skill's instructions already targeted
  the new shape.
- **`add-gmail-tool`** (custom, fork-authored skill — confirmed not present on
  either `upstream/main` or `upstream/channels`) — copied as-is, Phase 2
  reapplied via `container/cli-tools.json` (unchanged pattern from the first
  pass): `@gongrzhe/server-gmail-autoauth-mcp@1.1.11` +
  `zod-to-json-schema@3.22.5` as sibling entries alongside the carried-forward
  `@tobilu/qmd@2.5.3` entry. `TOOL_ALLOWLIST` still needs no edit —
  `mcpAllowPattern` still derives it from registered `mcpServers` keys.
- **WhatsApp PTT/voice-note + `lastSentId` return** (this fork's own
  customization, made 2026-08-31 — see the dedicated section in
  [customizations.md](customizations.md)) — reapplied on top of the freshly
  fetched v7 adapter; the surrounding code was unchanged from the version this
  was originally written against.
- **`.gitignore` package-lock guard**, **`setup/verify.ts` ONECLI_URL fix** —
  both still needed; upstream hadn't picked them up independently this time
  (unlike the Dockerfile pnpm pin and the RW session-DB handle from the first
  pass, which upstream had already absorbed).

Breaking-change detectors run clean against this fork's actual footprint:
`bun scripts/detect-driver-migration.ts` (via `pnpm exec tsx`, no bun on this
host) reported nothing; the mailbox-seam, central-DB-async, and
CLAUDE.md-compose detect greps from their respective `docs/*.md` guides found
no hits outside trunk's own already-migrated code. Full validation
(`pnpm run build`, `pnpm test` — 191 files / 2264 tests) passed in the
worktree before swap; the container image could not be built inside the
worktree itself (its tag is derived from the absolute project path via
`setup/lib/install-slug.sh`, so a worktree build would tag under a different,
useless name) — rebuild after swap, before restarting the service.

---

This fork diverged heavily from upstream (1063 upstream commits / 681 files vs.
216 local commits / 32 files since the last common base). Rather than merging,
this guide captures the fork's customizations as intent so they can be
replayed on a clean `upstream/main` checkout.

See [customizations.md](customizations.md) for the detailed reapply steps.

## Migration Plan

Apply in this order on the upgrade worktree:

1. **Attempt merging the 7 upstream skill branches** (section below). ~~These
   are used as-is~~ — UPDATE: all 7 conflicted against current upstream in
   practice (none had been rebased in months). See the "Applied Skills —
   actual outcome" table below for what really happened with each one:
   2 merged with manual fixes (qmd, emacs), 1 was reapplied fresh via its own
   restructured skill instead of a branch merge (ollama-tool), and 4 were
   skipped as superseded or unwanted (setup-dynamic-context, compact,
   channel-formatting, native-credential-proxy).
2. **Do NOT merge `upstream/skill/apple-container`.** It was applied to this
   fork historically but fully reverted by the most recent local commit
   (`aaf46068 feat: migrate container runtime to Docker + native WhatsApp
   channel`). Docker is the deliberate, current runtime. Reintroducing that
   branch would regress the fork.
3. **Run `/add-whatsapp` fresh** — DONE, code-apply phase only (commit
   `9cf67555`). The fork's own WhatsApp adapter was pinned to Baileys v6 with
   a hand-written `getPlatformId` patch; the user chose to abandon that in
   favor of upstream's current native Baileys v7 implementation on the
   `channels` branch, gaining upstream's ongoing fixes rather than preserving
   the old adapter. This drops `src/channels/whatsapp.ts`, `src/whatsapp-auth.ts`,
   and the `whatsapp` git remote's lineage entirely. **Live device pairing
   (QR/pairing-code) still needs to happen** on the real install after the
   swap — it needs `store/auth/` (gitignored instance data, doesn't exist in
   a worktree) and the user's phone. The old v6 session in `store/auth/`
   almost certainly won't be compatible with v7 either way — expect to
   re-pair.

   Same trap hit **`emacs`** during this pass, worth flagging: the earlier
   `skill/emacs` branch merge (in the "Applied Skills" table above) looked
   clean — no git conflict — but the file it added was written against
   v1-era APIs (`../db.js`, `../logger.js`, `Channel`/`RegisteredGroup`
   types) that don't exist in current v2, and didn't compile. A git-clean
   merge is not the same as a working merge when the branch predates a large
   upstream rewrite. Fixed by replacing with the current `emacs.ts` /
   `emacs.test.ts` / `emacs-registration.test.ts` / `nanoclaw.el` from
   `upstream/channels`, which has its own current `/add-emacs` skill (commit
   `9cf67555`, bundled with the WhatsApp commit).
4. **Run `/add-gmail-tool` fresh** — DONE (commit `8d0947f8` in the upgrade
   worktree). The skill only exists on `upstream/channels`, not `upstream/main`
   — copied its `SKILL.md` in and applied Phase 2 (code changes) manually: its
   own Dockerfile instructions are stale (predate the `cli-tools.json`
   refactor, same issue as qmd), so applied via `cli-tools.json` instead —
   `@gongrzhe/server-gmail-autoauth-mcp@1.1.11` + `zod-to-json-schema@3.22.5`
   as sibling entries (both land in the same `pnpm install -g` call, which is
   what makes the zod pin actually take). No `TOOL_ALLOWLIST` edit needed —
   confirmed the allowlist is now fully derived from registered `mcpServers`
   keys (`mcpAllowPattern` in `providers/claude.ts`), not a static list.
   **Still needed post-migration, on the real install:** Phase 1 (OneCLI
   Gmail connect — host machine, needs the user's Google account) and Phase 3
   (wire the `gmail` MCP server + `.gmail-mcp` mount into specific groups'
   `container.json` — gitignored instance data, out of scope for this guide).
5. **`/add-vercel` — do NOT run during migration.** Initially applied per the
   skill's own Phase 4 (add `vercel` to `cli-tools.json`), then reverted:
   `container/cli-tools.test.ts` has an explicit regression guard (`bakes in
   nothing that a skill is meant to add on request`) asserting `'vercel'`
   specifically must never be in the base manifest — it's meant to be added
   only by a live `/add-vercel` run with real OneCLI credentials (Phase 3 of
   that skill needs a pasted Vercel token), not pre-applied here. If Vercel
   deploy capability is wanted, run `/add-vercel` yourself after the migrated
   install is up and running.
6. **Drop orphaned modules** — already satisfied by construction. `src/transcription.ts`,
   `src/image.ts`, `src/image.test.ts` (local whisper.cpp / sharp image
   handling) never existed on `upstream/main`, only on top of this fork's own
   history — starting the worktree from clean upstream means they were never
   introduced. Nothing to delete.
7. **Skip the Dockerfile pnpm pin.** The fork manually pinned
   `pnpm@10.33.4` (with the comment "pnpm 11 breaks optional deps in global
   installs"). Verified: `upstream/main`'s current Dockerfile already pins
   `ARG PNPM_VERSION=10.33.0` with the identical rationale in its comment.
   This fix landed upstream independently — no action needed, don't
   reintroduce a manual pin.
8. **Apply the remaining source/config customizations** — see
   [customizations.md](customizations.md). These are genuine additions on
   top of upstream, unrelated to WhatsApp-v6 or Apple Container, and don't
   have a "just rerun a skill" shortcut.
9. ~~Restore the Ollama admin-tools wiring~~ — superseded, already covered
   by applying `add-ollama-tool` fresh in step 1 (that skill wires
   `OLLAMA_ADMIN_TOOLS` end-to-end as part of its own instructions).
10. Validate: `pnpm install && pnpm run build && pnpm test`.
11. **Apply the WhatsApp PTT/voice-note customization** — made after this
    guide's original generation, on top of the freshly reapplied `/add-whatsapp`
    v7 adapter. See [customizations.md](customizations.md#whatsapp-ptt-voice-note-and-lastsentid-return).

Note: `groups/` is entirely gitignored local instance data (not tracked in
git at all — confirmed via `git ls-tree`) and is untouched by this
migration, per the standard rule that data directories (`groups/`, `store/`,
`data/`, `.env`) are never part of a code migration. Nothing to do for it.

## Applied Skills — actual outcome (updated during upgrade execution)

Every one of the 7 skill branches conflicted against current `upstream/main`
when actually attempted (none had been rebased in months, and upstream
rewrote the setup engine, Dockerfile CLI-install pattern, agent-runner
entrypoint, MCP server registration, and channel-formatting approach in the
interim). Blind merges were unsafe. Actual resolution per skill:

| Skill | Outcome | Why |
|---|---|---|
| setup-dynamic-context | **Skipped** | Upstream's `setup/SKILL.md` was completely rewritten (the "channel install skills are single source of truth" breaking change). The file this patch targeted no longer exists in a compatible form. |
| qmd | **Merged + manually fixed** | Dockerfile conflict: upstream moved global CLI installs from ad hoc `RUN npm install -g` to `container/cli-tools.json` + `install-cli-tools.sh`. Resolved by keeping upstream's Bun-install Dockerfile block and adding `{"name": "@tobilu/qmd", "version": "2.5.3"}` to `cli-tools.json` instead. Also resolved `container/agent-runner/src/index.ts`: dropped the branch's hardcoded `qmd: { type: 'http', ... }` MCP server entry — upstream's current architecture derives MCP servers (and their tool allow-patterns) from per-group `container.json`/DB config, not hardcoded source. Wiring qmd for a given group is now a `ncl groups config add-mcp-server` / container.json concern (see the qmd container skill for the URL: `http://host.docker.internal:8182/mcp`). |
| native-credential-proxy | **Skipped (user's choice)** | Merged into the fork historically but never activated — no env var, no source references beyond the skill doc itself. OneCLI is what's actually running. User confirmed skip; `/use-native-credential-proxy` remains available to run later if desired. |
| emacs | **Merged cleanly** | Only conflict was the one-line self-registration import in `src/channels/index.ts` (mechanical barrel-file conflict, upstream's barrel just has different content around it now). |
| compact | **Skipped — superseded** | `/compact` is already a native admin-gated command in current upstream (`src/command-gate.ts` `ADMIN_COMMANDS` set, passed straight through to the agent SDK which handles context compaction natively). The skill branch's bespoke v1-era `session-commands.ts` machinery has no purpose anymore. |
| channel-formatting | **Skipped — superseded, and reapplying it would be actively wrong** | The skill's `text-styles.ts` was a unified cross-channel Markdown→native transformer invoked from `router.ts`. Upstream has since decentralized this: WhatsApp's adapter (the v7 one being freshly adopted) converts Markdown to native formatting automatically inside the adapter itself; Slack instead teaches the agent to write mrkdwn directly via the `slack-formatting` container skill (installed with `/add-slack`). Reapplying a blanket router-level transformer on top of either would risk double-transforming or corrupting already-channel-native text. |
| ollama-tool | **Applied fresh via the skill itself, not a branch merge** | `container/agent-runner/src/ollama-mcp-stdio.ts` no longer exists on `upstream/main` at all — `ollama-tool` was restructured into a proper opt-in utility skill (`.claude/skills/add-ollama-tool/`) that ships its own source + tests and installs on demand, same pattern as `add-vercel`. Applied by following that skill's own `SKILL.md` instructions directly (copy files, register in `index.ts`'s `mcpServers`, wire `ollamaEnvArgs()` into `container-runner.ts`, `.env.example` stub). This already includes the `OLLAMA_ADMIN_TOOLS`-gated management tools (pull/delete/show/list-running) this fork wanted restored — see commit `05483c9e` in the upgrade worktree. **The dedicated "Ollama Admin Tools" restoration section in customizations.md is now obsolete/superseded by this — skip it.** |

**apple-container** (`upstream/skill/apple-container`) was applied to this
fork historically but is **not** in the above table — it's explicitly
excluded per plan step 2 (reverted by the fork's own most recent commit;
Docker is the deliberate current runtime).

All other installed skills under `.claude/skills/` (add-discord, add-slack,
add-telegram, etc.) are stock trunk/channels-branch content already, not fork
customizations — nothing to migrate for those beyond what a normal clean
checkout already ships.

## Skill Interactions

Two of the applied skills absorbed pre-existing custom code the user had
built before the official skill existed. Both turned out to be non-issues
after merging, but are worth knowing about if the worktree merge produces
conflicts in these files:

- **channel-formatting vs. custom `src/text-styles.ts`**: the user had
  originally hand-built channel-aware Markdown formatting
  (`e87d15db feat: channel-aware text formatting for WhatsApp, Telegram,
  Slack, Signal`) before upstream shipped the official `channel-formatting`
  skill. After merging the skill, `git diff <skill-merge-hash>..HEAD --
  src/text-styles.ts` shows **only prettier reformatting**, zero functional
  difference. The skill's version is a strict superset — nothing to reapply
  here beyond the skill merge itself.

- **ollama-tool vs. custom "gated model management tools"**: the user
  originally added admin-gated Ollama MCP tools
  (`5743a48f feat: add gated model management tools to Ollama MCP server`)
  before the official `ollama-tool` skill existed. The skill merge (`80616cef`)
  brought in an equivalent (or the same) feature, including the
  `OLLAMA_ADMIN_TOOLS` env var plumbing all the way through
  `src/container-runner.ts`. **A later, unrelated refactor removed the
  env-var-forwarding half of that wiring from `container-runner.ts`**,
  leaving the feature present in `ollama-mcp-stdio.ts` but permanently dead
  (the container never receives the flag). This needs restoring — see
  customizations.md.

## Verify After Merging /add-whatsapp

`setup/groups.ts` (229 lines, new in this fork) replaced the previous
`05-sync-groups.sh` / `05b-list-groups.sh` shell scripts with a
Baileys-based WhatsApp group-metadata fetch for interactive setup. It was
written against this fork's own WhatsApp integration. After running
`/add-whatsapp` fresh in the worktree, check whether upstream's setup flow
already provides equivalent group-sync functionality for its native adapter.
If it does, `setup/groups.ts` is superseded — drop it. If it doesn't, the
group-fetch logic needs porting to call upstream's actual WhatsApp adapter
API (whatever that turns out to be after the fresh skill install) rather
than being copied verbatim, since it currently assumes this fork's own
Baileys wiring.
