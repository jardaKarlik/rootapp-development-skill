---
name: rootapp-development
description: Use when building, reviewing, or debugging a Root App (a full React/TypeScript mini-application that runs inside a channel of the Root community client, rootapp.com). Covers dev-machine setup, account/DEV_TOKEN flow, the client/server/networking stack, audio playback rules, persistence options, and known unresolved platform behaviors. Load this before writing any Root App code so you don't have to re-read docs.rootapp.com from scratch.
---

# Root App Development

Root ("Root: Built for Community", rootapp.com) is a Discord-adjacent community/chat client. Developers extend it with **Root Apps** (full UI mini-apps, what this skill covers) or **Root Bots** (headless, no UI — use the separate bot docs if that's the ask). Each Root App runs inside its own dedicated channel in a community; members jump in to interact with it.

## Stack

- **Client**: React + TypeScript. Runs in a sandboxed, "browser-like container" inside the Root desktop/mobile client. Responsive design required (phone screen sizes + desktop presentation modes). Theme (light/dark) and localization are provided by Root — detect and respond to both.
- **Server**: TypeScript. Root hosts and scales it, and hosts its database with automatic backups. You do not manage infrastructure for this part.
- **Client-server networking**: custom messages/services defined with **Protobuf**, generated into typed client/server code via Root SDK tooling. `RootServerException` is the only exception type that can cross the server→client boundary.
- **Toolchain**: Node.js 22+, `npx create-root` to scaffold starter code, VS Code or WebStorm, the Root desktop client itself (needed to test).

## Account and project setup (do this once per App, before writing code)

1. Install the Root desktop client (rootapp.com/download) and register a Root account from it (Register → fill in info → Register). One account serves as both a Root member and a Root developer.
2. Open the Root Developer Portal at **dev.rootapp.com** and log in with that account — this alone enrolls you in the developer program.
3. In the portal, go to **My apps → New app**, name it, agree to terms, **Create**. This "project" is the container for the App's identity, versions, code, and state — one project per App.
4. In that App's **Settings** tab, **Generate new dev token**. The resulting `DEV_TOKEN` encapsulates your identity, infra authorization, and a community ID. Treat it like a password. Generating your first `DEV_TOKEN` for a project auto-creates a test community named `<project>-test` (recreated if you delete it).
5. `npx create-root` to scaffold, add `DEV_TOKEN` to the server project per the generated README, then build/run locally — both client and server run on your machine against the test community during development.

Later, publishing to a real community goes through the **Publish** section of the docs (store submission) — re-check those docs when you reach that stage, this skill doesn't cover store review.

## Recommended patterns

- **Keep the App thin where possible.** Root Apps are good at being a focused control surface (a game, a dashboard, a player) inside one channel — they are not a drop-in replacement for Root's native text chat. If your App needs ongoing discussion alongside it, the safe default is **two channels**: the App's own channel for the interactive surface, plus a normal Root text channel for discussion, cross-linked with **App links** (shareable deep links into a specific place in your App that a member can post in a text channel). Whether the Community Access API can read/write a *different* channel's messages directly (letting you merge both into one channel) is **unconfirmed in the public docs** — verify directly against the live docs site or by testing before designing around it; don't assume it.
- **Persistence**: two server-side options — a `KeyValueStore` (SQLite-backed key-value pairs, zero setup, auto-backed-up) for simple state, or a full SQLite database for structured/relational data (e.g., an ordered playlist table with a position column). Pick SQLite-proper whenever you need ordering, joins, or more than trivial structure.
- **Scheduled/future work**: the **Job scheduler** API (create/get/edit/delete/list tasks for future dates, DB-backed, auto-backed-up) is the right tool for "do X at time Y" server-side logic (e.g., advance a shared playback position) — don't hand-roll timers that only live in one client's memory, since all clients need to agree on shared state.
- **File uploads**: `rootClient.asset.fileUpload()` (client, opens a file picker, returns a temporary upload token) → pass the token to your server via your own networking → `rootServer.dataStore.assets.create()` (server, exchanges tokens for permanent asset URIs). Token expires, so convert promptly. Size/access-control limits for assets are not documented — test with a representative file before relying on this for anything large (e.g., long audio).
- **Diagnostics**: use the Community log API to send diagnostic messages from your server to community admins instead of inventing your own side channel.
- **Audiences**: Member groups combine users + community roles into named audiences for broadcasts/custom access control — use this instead of re-deriving "who can see/do X" by hand.

## Audio (client)

- Bundle short audio in `client/public/` (e.g. `public/audio/theme.mp3`), referenced by relative URL. This is copied into `client/dist` and deployed with `root-manifest.json`. **Counts against a 100 MiB total package size limit** — fine for UI sounds/theme music, not a real media library.
- Formats: MP3/WAV play everywhere; desktop also plays OGG, FLAC, AAC.
- Use an `Audio` object or JSX `<audio>` with `ref`/`src`/`loop`; `play()` returns a promise and **rejects on load failure** — wrap in try/catch.
- `<audio>` streams rather than fully buffering, which is what you want for anything long.
- **Known Android bug**: seeking (`currentTime`) past the loaded portion can kill playback, because package-bundled files stream without a `Content-Length` header. Workaround: fetch the file, convert to a Blob, play via an object URL, and `URL.revokeObjectURL()` when switching tracks. Desktop and iPhone don't have this problem.
- Web Audio API is available for mixing/scheduling/tone generation; `decodeAudioData` fully decodes before playback (budget ~9x the file's compressed size in memory). `AudioContext` can start `suspended` with no sound and no error — always `await resume()` before starting a buffer source. iPhone's silent switch mutes Web Audio but not plain `<audio>` elements.
- **Unconfirmed and important if your App needs continuous audio from your own backend (not a bundled file)**: whether a Root App client's `<audio>`/media element can load an arbitrary external HTTPS domain at all, and whether any manifest/CSP permission is required. The public docs only document the bundled-file path. Spike this for real (point an `<audio>` tag at a real external URL in a throwaway App) before committing to an architecture that depends on it, and report the result back rather than assuming either way.

## Configuration (root-manifest.json)

- Required: `id` (assigned by the Developer Portal when you create the project), `version` (semver), `package` (client/server deployment + startup config).
- Optional: `settings` (global settings a community's Manage-Apps members configure at/after install), `permissions` (gates which Community API methods your code may call — skip this field entirely if you call none).
- Network/CSP allowances, if any exist for external domains, are not documented publicly as of this writing — don't assume a permission field exists for this; verify directly.

## Toolset checklist for an agent starting on a Root App task

- Node.js 22+, npm/npx
- `npx create-root` (scaffolding)
- A code editor (VS Code/WebStorm) with TypeScript support
- The Root desktop client installed, logged into a Root account enrolled in the Developer Portal
- A `DEV_TOKEN` for the specific App project (from dev.rootapp.com → My apps → the App → Settings) — **this is per-human-account; get it from whoever owns the Root developer identity, don't generate your own unless told to**
- Familiarity with Protobuf (service/message definitions) for client-server networking
- A browser devtools mindset for the client (it's a "browser-like container") plus awareness that mobile and desktop behave differently (see Audio section)

## Doc map (primary sources, check these directly for anything this skill doesn't cover)

- Dev portal overview: https://docs.rootapp.com/docs/overview/develop-on-root
- Get started (account/project/DEV_TOKEN): https://docs.rootapp.com/docs/app-docs/get-started/
- App home / section map: https://docs.rootapp.com/docs/app-docs/app-home/
- Develop → Client: https://docs.rootapp.com/docs/app-docs/develop/client/
- Develop → Client → Audio: https://docs.rootapp.com/docs/app-docs/develop/client/audio/
- Develop → Server (Lifecycle, GUIDs, Assets, Community log, Job scheduler, Client attachment, Member groups, Community access, Persistence): https://docs.rootapp.com/docs/app-docs/develop/server/
- Develop → Networking (protobuf): https://docs.rootapp.com/docs/app-docs/develop/networking/
- Configure (manifest/settings/permissions): https://docs.rootapp.com/docs/app-docs/configure/
- API reference index: https://docs.rootapp.com/docs/app-api-reference/
- Tutorials (Showdown — live voting game; Tasks — task tracker w/ DB): https://docs.rootapp.com/docs/app-docs/tutorials/
- Community concepts (channels, roles — owner/admin/member): https://support.rootapp.com/docs/leader/what-is-community
- Official dev community (ask platform questions the docs don't answer, e.g. audio-external-URL, cross-channel Community Access): rootapp.gg/developer

## Known open questions (don't guess — verify, then update this skill)

1. Can a Root App client's audio/media element load an external HTTPS domain, and is any manifest/CSP permission required?
2. Can the server-side Community Access API read/post messages into a *different* channel than the App's own (needed to merge a player + chat into one view)?
3. Are there documented size/format limits on the Assets API for large/long files (e.g., a 60–90 minute audio mix)?
