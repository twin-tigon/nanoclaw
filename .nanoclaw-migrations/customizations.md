# Customizations

Apply these after the skill merges (see index.md's Migration Plan) and after
dropping the orphaned WhatsApp-v6 / whisper / image modules. These are real,
standalone changes with no upstream equivalent — hand-copy them.

---

### .env.example — document optional env vars

**Intent:** Surface two optional env vars that were previously undocumented.

**Files:** `.env.example`

**How to apply:** Append:
```
ASSISTANT_HAS_OWN_NUMBER=
OLLAMA_HOST=
```

---

### .gitignore — block npm lockfile

**Intent:** This project is pnpm-only (see CLAUDE.md supply-chain rules); a
stray `package-lock.json` from someone running plain `npm install` shouldn't
get committed.

**Files:** `.gitignore`

**How to apply:** Append:
```
# npm artifacts — this project is pnpm-only (see CLAUDE.md supply-chain rules)
package-lock.json
```

---

### Delete fork-specific CI workflows

**Intent:** `bump-version.yml` and `update-tokens.yml` auto-bump the version
and token-count badge on the canonical upstream repo. This fork isn't the
canonical repo, so that automation doesn't apply — it was deliberately
deleted (commit `5d7f9d77 ci: remove the auto version-bump and token-count
workflows`).

**Files:** `.github/workflows/bump-version.yml`, `.github/workflows/update-tokens.yml`

**How to apply:** If the fresh worktree (from `upstream/main`) has these
files, delete them.

---

### container/entrypoint.sh + container-runner.ts uid mapping — SUPERSEDED, do not apply

~~This section originally planned to replace entrypoint.sh with a
root-then-setpriv privilege-drop script and add `-u node` +
`RUN_UID`/`RUN_GID` forwarding in container-runner.ts.~~ Verified during the
upgrade: upstream's current `container-runner.ts` (around line 515) already
does `args.push('--user', \`${hostUid}:${hostGid}\`)` for non-1000/non-root
hosts, and the Dockerfile still bakes `USER node` for the standard case —
functionally equivalent to what this fork's customization was solving,
implemented via Docker's native `--user` flag instead of a root-start +
`setpriv` dance. Additionally, the fork's `.env`-shadow-mount logic in
entrypoint.sh solves a problem that doesn't exist in the current
architecture — nothing mounts `.env` into the container at all anymore
(confirmed: no `.env` reference in `container-runner.ts`; OneCLI injects
secrets at request time, never via env vars or mounts). **Left
`container/entrypoint.sh` and `container-runner.ts`'s user-mapping block at
upstream's stock content — nothing to reapply.**

<details><summary>Original (superseded) instructions, kept for reference</summary>

### container/entrypoint.sh — privilege-drop for non-uid-1000 hosts

**Intent:** Docker's `--user node` (uid 1000) is what makes Claude Code's
`--dangerously-skip-permissions` flag work inside the container (it's
blocked for root). On hosts where the invoking user isn't uid 1000, the host
passes `RUN_UID`/`RUN_GID` and the entrypoint starts as root just long enough
to `chown` the stdin capture file and `setpriv` down to that uid before
exec'ing bun. This is paired with the `-u node` / `RUN_UID` logic in
`src/container-runner.ts` below — the entrypoint's setpriv path is a fallback
that's rarely exercised (most hosts are uid 1000), not dead code.

**Files:** `container/entrypoint.sh`

**How to apply:** Replace the file's contents entirely with:

```bash
#!/bin/bash
# NanoClaw agent container entrypoint.
#
# The host passes initial session parameters via stdin as a single JSON blob,
# then the agent-runner opens the session DBs at /workspace/{inbound,outbound}.db
# and enters its poll loop. All further IO flows through those DBs.
#
# Apple Container only supports directory mounts (VirtioFS) — file mounts like
# /dev/null are not supported. Instead we start as root, shadow .env via
# mount --bind inside the VM, then drop to the host user via setpriv.

set -e

# Shadow .env so the agent cannot read host secrets (requires root).
# Credentials are injected by the OneCLI gateway, never exposed to containers.
if [ "$(id -u)" = "0" ] && [ -f /workspace/.env ]; then
  mount --bind /dev/null /workspace/.env
fi

cat > /tmp/input.json

# Drop privileges if running as root.
# RUN_UID/RUN_GID are set by the host when the host user isn't node (uid 1000).
if [ "$(id -u)" = "0" ] && [ -n "$RUN_UID" ]; then
  chown "$RUN_UID:$RUN_GID" /tmp/input.json
  exec setpriv --reuid="$RUN_UID" --regid="$RUN_GID" --clear-groups -- bun run /app/src/index.ts < /tmp/input.json
fi

exec bun run /app/src/index.ts < /tmp/input.json
```

Note: the comment block still references Apple Container by name (that's how
this logic originated) even though the fork now runs Docker — the
`RUN_UID`/`RUN_GID` fallback path is still live and used by Docker for
non-1000-uid hosts. Consider rewording the comment during reapply since
Apple Container is no longer in play for this fork, but keep the mechanism.

---

### src/container-runner.ts — uid mapping for Docker

**Intent:** Force the container to run as the `node` user (uid 1000) so
Claude Code's `--dangerously-skip-permissions` is accepted. For hosts whose
invoking user isn't uid 1000, forward `RUN_UID`/`RUN_GID`/`HOME` so
`entrypoint.sh`'s setpriv fallback (above) can still drop to the right
identity.

**Files:** `src/container-runner.ts`

**How to apply:** In the container-spawn arg-building function, after the
host-gateway args and before the volume-mounts loop, add:

```typescript
// User mapping — run as the container's `node` user (uid 1000) so Claude
// Code's --dangerously-skip-permissions flag is accepted (it is blocked for
// root). Docker respects --user natively; we pass RUN_UID/RUN_GID only for
// non-1000 hosts in case the entrypoint.sh setpriv path is ever exercised.
const hostUid = process.getuid?.();
const hostGid = process.getgid?.();
args.push('-u', 'node');
if (hostUid != null && hostUid !== 0 && hostUid !== 1000) {
  args.push('-e', `RUN_UID=${hostUid}`);
  args.push('-e', `RUN_GID=${hostGid}`);
  args.push('-e', 'HOME=/home/node');
}
```

</details>

---

### src/container-runtime.ts — proxy bind host detection — SUPERSEDED, do not apply

Verified during the upgrade: `PROXY_BIND_HOST`/`CONTAINER_HOST_GATEWAY` are
**dead code even in the current fork** — exported from
`src/container-runtime.ts` but never imported or consumed anywhere else in
`src/*.ts`. They trace back to the abandoned Apple-Container-networking fix
commits (`c0b58bd7`, `9fc60f6a`), not the actually-active OneCLI credential
path. OneCLI's own SDK (`onecli.applyContainerConfig()` in
`container-runner.ts`) handles the container's proxy/cert wiring internally
as a black box — this repo doesn't implement that binding itself. **Skip —
nothing to reapply.**

<details><summary>Original (superseded) instructions, kept for reference</summary>

**Intent:** The credential proxy needs a bind address containers can reach.
macOS/WSL (Docker Desktop) route `host.docker.internal` to loopback, so
`127.0.0.1` is correct and safe there. Bare-metal Linux Docker has no such
routing — binding to `0.0.0.0` would expose the proxy beyond the container,
so bind to the `docker0` bridge IP instead (only containers can reach that).
An explicit `CREDENTIAL_PROXY_HOST` env var always wins if set.

**Files:** `src/container-runtime.ts`

**How to apply:** Add near the top of the file, after the existing
`CONTAINER_RUNTIME_BIN` export:

```typescript
import fs from 'fs';

/** Hostname containers use to reach the host machine. */
export const CONTAINER_HOST_GATEWAY = 'host.docker.internal';

/**
 * Address the credential proxy binds to.
 * Docker Desktop (macOS): 127.0.0.1 — the VM routes host.docker.internal to loopback.
 * Docker (Linux): bind to the docker0 bridge IP so only containers can reach it,
 *   falling back to 0.0.0.0 if the interface isn't found.
 */
export const PROXY_BIND_HOST = process.env.CREDENTIAL_PROXY_HOST || detectProxyBindHost();

function detectProxyBindHost(): string {
  if (os.platform() === 'darwin') return '127.0.0.1';

  // WSL uses Docker Desktop (same VM routing as macOS) — loopback is correct.
  // Check /proc filesystem, not env vars — WSL_DISTRO_NAME isn't set under systemd.
  if (fs.existsSync('/proc/sys/fs/binfmt_misc/WSLInterop')) return '127.0.0.1';

  // Bare-metal Linux: bind to the docker0 bridge IP instead of 0.0.0.0
  const ifaces = os.networkInterfaces();
  const docker0 = ifaces['docker0'];
  if (docker0) {
    const ipv4 = docker0.find((a) => a.family === 'IPv4');
    if (ipv4) return ipv4.address;
  }
  return '0.0.0.0';
}
```

(`os` is already imported in this file for other purposes — just add the
`fs` import.) Check for existing callers of `CONTAINER_HOST_GATEWAY` /
`PROXY_BIND_HOST` elsewhere (e.g. wherever the credential proxy server is
started) and wire them to use these exports instead of hardcoded values, if
that wiring doesn't already exist on the fresh upstream checkout.

</details>

---

### src/db/session-db.ts / session-manager.ts / host-sweep.ts — RW handle — SUPERSEDED, already upstream

Verified during the upgrade: upstream independently built the identical fix.
`src/db/session-db.ts` already has `openOutboundDbRw()` (functionally
identical — same pragmas, same doc comment intent), `session-manager.ts`
already wraps it and already uses it in `writeOutboundDirect`, and
`host-sweep.ts` already uses it for orphan `processing_ack` cleanup. **All
three files below: skip, nothing to reapply — upstream already has this.**

<details><summary>Original (superseded) instructions, kept for reference</summary>

### src/db/session-db.ts — RW handle for post-crash cleanup

**Intent:** `openOutboundDb()` opens the outbound DB read-only (the
container is the sole normal writer, per the two-DB session split). The host
sweep needs a *write* handle, but only for one narrow case: deleting orphan
`processing_ack` rows after a container crash, when no container is running
to own that write.

**Files:** `src/db/session-db.ts`

**How to apply:** Add immediately after the existing `openOutboundDb`
function:

```typescript
/**
 * Open the outbound DB for writing. Only used by the host sweep to delete
 * orphan processing_ack rows after a container crash — the container is the
 * normal writer; the host only writes here during post-crash cleanup when no
 * container is running.
 */
export function openOutboundDbRw(dbPath: string): Database.Database {
  const db = new Database(dbPath);
  db.pragma('journal_mode = DELETE');
  db.pragma('busy_timeout = 5000');
  return db;
}
```

---

### src/session-manager.ts — expose the RW wrapper

**Intent:** Wrap `openOutboundDbRw` the same way the existing readonly
`openOutboundDb` is wrapped (resolving `agentGroupId`/`sessionId` to a path).
Also switch `writeOutboundDirect` (used by the command gate to write denial
responses) to use the RW handle — it was previously using the readonly one,
which is a latent bug independent of this migration.

**Files:** `src/session-manager.ts`

**How to apply:**

1. Add `openOutboundDbRw as openOutboundDbRwRaw` to the import from
   `./db/session-db.js`.
2. Add, immediately after the existing `openOutboundDb` export:
   ```typescript
   /** Open the outbound DB for writing (post-crash orphan cleanup only). */
   export function openOutboundDbRw(agentGroupId: string, sessionId: string): Database.Database {
     return openOutboundDbRwRaw(outboundDbPath(agentGroupId, sessionId));
   }
   ```
3. In `writeOutboundDirect`, change `const db = openOutboundDb(agentGroupId, sessionId);` to
   `const db = openOutboundDbRw(agentGroupId, sessionId);`.

---

### src/host-sweep.ts — use the RW handle for orphan cleanup

**Intent:** `resetStuckProcessingRows` was calling `deleteOrphanProcessingClaims(outDb)`
using the caller's readonly `outDb` handle, which would fail once
`openOutboundDb` is strictly readonly. Open a dedicated RW connection for
just this write, scoped tightly and closed immediately after.

**Files:** `src/host-sweep.ts`

**How to apply:**

1. Add `openOutboundDbRw` to the import from `./session-manager.js`.
2. In `resetStuckProcessingRows`, replace:
   ```typescript
   const cleared = deleteOrphanProcessingClaims(outDb);
   ```
   with:
   ```typescript
   // Must open a separate RW connection — the caller's outDb is readonly.
   const outDbRw = openOutboundDbRw(session.agent_group_id, session.id);
   let cleared = 0;
   try {
     cleared = deleteOrphanProcessingClaims(outDbRw);
   } finally {
     outDbRw.close();
   }
   ```

</details>

---

### setup/verify.ts — ONECLI_URL isn't a credential

**Intent:** The credentials-configured check was treating `ONECLI_URL` (just
a gateway endpoint, not a secret) as evidence of configured credentials.
Removed from the regex so verify doesn't falsely report "configured" based
on a non-credential var.

**Files:** `setup/verify.ts`

**How to apply:** Change:
```typescript
if (/^(CLAUDE_CODE_OAUTH_TOKEN|ANTHROPIC_API_KEY|ONECLI_URL)=/m.test(envContent)) {
```
to:
```typescript
if (/^(CLAUDE_CODE_OAUTH_TOKEN|ANTHROPIC_API_KEY)=/m.test(envContent)) {
```

---

### WhatsApp PTT voice-note and lastSentId return

**Intent:** Two small fixes on top of the freshly reapplied `/add-whatsapp`
(native Baileys v7) adapter, made after this guide's original generation
(commit `d7bd82fc fix(whatsapp): send .ogg as PTT voice note, return message
id for caption-less file sends`). Apply these to `src/channels/whatsapp.ts`
**after** `/add-whatsapp` has been re-run against the new upstream base — they
build on the current adapter, not the pre-migration one.

**Files:** `src/channels/whatsapp.ts`

**How to apply:**

1. In the file-attachment mimetype/message-builder function (`buildMediaMessage`
   as of this writing — locate by its `audioExts`/`videoExts` branching that
   maps a file extension to a Baileys media message shape), add a dedicated
   `.ogg` branch **before** the generic audio-extension branch:
   ```typescript
   if (ext === '.ogg') {
     // WhatsApp only renders the playable "voice note" bubble (round record
     // icon + waveform) for ptt:true with this exact codec-qualified
     // mimetype. Without both, WhatsApp treats it as a generic audio file —
     // or on some clients, silently drops it instead of showing a document
     // fallback.
     return { audio: data, ptt: true, mimetype: 'audio/ogg; codecs=opus' };
   }
   ```
   Without this, `.ogg` falls through to the generic audio branch and sends
   as a regular audio/document attachment instead of a voice bubble.

2. In the outbound send handler's file-attachment loop (iterates
   `message.files!`, calls `sock.sendMessage` per file, tracks whether a
   caption was consumed), add tracking for the last successfully sent
   message ID and return it when there's no text left to send:
   ```typescript
   let lastSentId: string | undefined;
   for (const file of message.files!) {
     // ... existing per-file send logic ...
     const sent = await sock.sendMessage(platformId, mediaMsg);
     if (sent?.key?.id && sent.message) {
       sentMessageCache.set(sent.key.id, sent.message);
       lastSentId = sent.key.id;   // <-- add this line
     }
     if (caption) captionUsed = true;
     // ...
   }
   if (captionUsed || !text) return lastSentId;   // <-- was: if (captionUsed) return;
   ```
   Previously, a files-only send with no caption and no trailing text fell
   through and returned `undefined` implicitly. This makes it return the
   last file's message ID in that case too, so callers get a usable ID for
   any caption-less/text-less file send, not just when a caption was
   consumed.

If the reapplied v7 adapter has restructured either function since this was
written, apply the same intent (the `.ogg` → `ptt:true` codec-qualified
mimetype branch, and returning the last sent file's message ID when no text
follows) rather than a literal patch.

---

## Ollama Admin Tools — SUPERSEDED, do not apply

~~This section originally planned to restore `OLLAMA_ADMIN_TOOLS` forwarding
in `container-runner.ts` after merging `upstream/skill/ollama-tool`.~~ That
plan is obsolete: `ollama-tool` was restructured upstream into the
`add-ollama-tool` utility skill (ships its own source, no branch merge), and
applying it fresh (done — see index.md's Applied Skills table, commit
`05483c9e` in the upgrade worktree) already wires `OLLAMA_ADMIN_TOOLS`
end-to-end via `ollamaEnvArgs()` in `src/ollama-env.ts`, imported and called
in `src/container-runner.ts`. Nothing left to do here.
