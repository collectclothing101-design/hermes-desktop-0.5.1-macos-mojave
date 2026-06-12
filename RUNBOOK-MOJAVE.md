# Hermes Desktop — macOS 10.14.6 Mojave (x64) Build Runbook

This fork is patched to produce a desktop app that **launches on macOS 10.14.6 Mojave (Intel/x64)**.

## Why these changes were needed

| Constraint | Fact | Consequence |
|---|---|---|
| Electron runtime | Chromium **M116 is the last Chromium to run on macOS 10.13/10.14**; M117+ needs 10.15. Electron 27+ bundles Chromium ≥118 and requires macOS 10.15. | The newest Electron that launches on Mojave is **Electron 26.6.10** (Chromium 116 / Node 18.16.1). The repo originally pinned **Electron 39** → would not launch on Mojave at all. |
| Native module | `better-sqlite3@12` declares `engines.node >= 20`. Electron 26 ships Node 18.16. | Downgraded to **better-sqlite3@11.10.0** (Node-18 compatible; the `Database`/`prepare`/`run` API the code uses is unchanged). |
| Architecture | Mojave is **x64-only** (no Apple Silicon, no arm64). | Build target locked to `--x64`. |
| App code | Scanned: no post-Chromium-116 JS (`Object.groupBy`, `Promise.withResolvers`, …), no post-Electron-26 APIs, `contextIsolation`/`sandbox` already on. | **No source rewrite required** — config + deps only. |

## What was changed

1. **`package.json`**
   - `devDependencies.electron`: `^39.2.6` → **`26.6.10`** (pinned)
   - `dependencies.better-sqlite3`: `^12.8.0` → **`11.10.0`**
   - Added `@electron/rebuild` (devDep) + scripts: `build:mac:x64`, `rebuild:native`
   - `postinstall` (`electron-builder install-app-deps`) now rebuilds `better-sqlite3` against Electron 26 automatically.
2. **`electron.vite.config.ts`**
   - main/preload build `target: "node18"`; renderer build + dep-optimizer `target: "chrome116"` (transpiles output down to the Mojave runtime).
3. **`electron-builder.yml` → `mac:`**
   - `minimumSystemVersion: "10.14.6"` (sets `LSMinimumSystemVersion`)
   - `target: dmg + zip`, both `arch: x64`
   - `hardenedRuntime: false`, `notarize: false`, `gatekeeperAssess: false`, `identity: null` — i.e. an **unsigned local build** (notarization/hardened runtime are for signed modern-macOS distribution and add friction here). Sign it yourself if you have a Developer ID; see the optional step below.

## Build it (on a Mac running macOS 11+ — NOT on the Mojave machine)

The bundler (Vite 7 / electron-vite 5) needs **Node 20.19+ or 22.12+**, and Node 20+ does not run on Mojave. So build on any newer Mac, then copy the artifact to the Mojave machine. Building x64 from an Apple Silicon Mac is fine — electron-builder pulls the x64 Electron 26 binary.

```bash
# Prereqs on the build Mac: Node ≥ 20.19, Xcode Command Line Tools (xcode-select --install), python3
node -v                      # must be >= 20.19
rm -rf node_modules out dist package-lock.json
npm install                  # postinstall rebuilds better-sqlite3 for Electron 26
npm run rebuild:native       # belt-and-suspenders: force better-sqlite3 against Electron 26 ABI
npm run build:mac:x64        # → dist/hermes-desktop-0.5.1-x64.dmg  (+ .zip)
```

Artifacts land in `dist/`. Copy the `.dmg` (or `.zip`) to the Mojave Mac.

## Install on the Mojave Mac

Because the build is unsigned, Gatekeeper will block a double-click. Either:

- Right-click the app → **Open** → **Open** (one-time per app), or
- Strip the quarantine flag in Terminal:
  ```bash
  xattr -dr com.apple.quarantine "/Applications/Hermes Agent.app"
  ```

## Hermes Agent backend prerequisite (separate from this GUI)

This desktop app is a front-end that installs/runs the **Hermes Agent** Python backend
(`NousResearch/hermes-agent`, via `uv` + Python, fetched by the in-app installer or
`scripts/install.sh`). That backend has its own toolchain needs (git, `uv`, Python, ripgrep,
ffmpeg). `uv` and a modern Python still install on Mojave, but verify the agent's own
requirements separately — the Electron downgrade only fixes the GUI shell, not the agent.

## Optional: sign with your own Developer ID (still no notarization)

In `electron-builder.yml` set `identity: "Developer ID Application: Your Name (TEAMID)"`
and keep `notarize: false`. Mojave does not require notarization for locally-installed apps.

## If you can ONLY build on the Mojave machine itself (no newer Mac available)

You'd additionally have to downgrade the *build toolchain* so it runs on Node 18 (last Node
to support Mojave): electron-vite 2.x, Vite 4.x, `@vitejs/plugin-react` 4.x, React 18 +
react-dom 18, Tailwind 3.x (Tailwind 4's Vite plugin needs newer Node), and matching
`@types/react@18`. This is a larger, more invasive change with its own React-19→18 API
adjustments. The "build on a newer Mac, run on Mojave" path above is strongly preferred.

## Honest caveats

- Mojave and Electron 26 are both **end-of-life** and receive no security updates — fine for a
  personal machine, not for anything internet-exposed or handling sensitive data.
- Tailwind v4 leans on `oklch()` / `color-mix()` (Chrome 111+, so OK on 116); extreme-edge CSS
  may render with minor polish differences, but layout/function are intact.
- The final `.dmg` must be produced **on macOS** — electron-builder cannot cross-build a macOS
  DMG from Linux/Windows.
