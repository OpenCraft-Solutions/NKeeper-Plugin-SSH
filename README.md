# NKeeper SSH plugin

The official `ssh` plugin for [NKeeper](https://github.com/OpenCraft-Solutions/NKeeper) 2.0: SSH terminals in the browser. It lives in a repository of its own and is released on its own, so releasing it isn't a release of NKeeper (NKeeper's `docs/architecture.md` §9).

## What it does

- **Service type** `ssh`, port 22 by default, checked by core's TCP health probe.
- **Session type** `ssh` ("Terminal"), on `ssh` services. Core opens the SSH connection, checks the server's host key against its trust store, asks the user about a new one and signs in with the credential. The plugin never sees a password or a key: it opens a shell on an `xterm-256color` terminal and carries the bytes both ways.
  - `options`: the terminal's `columns` and `rows`. Anything that isn't a number in range falls back to 80 × 24.
  - Binary frames are the terminal's bytes; a text frame `resize` `{ columns, rows }` resizes it.
- **Host action** `ssh.terminal` ("Terminal"), which opens the terminal with a credential.
- **Frontend**: the terminal tab, built on xterm.js, fitted to its tab. When the shell exits it says so and offers to reconnect.

The plugin's ID is `ssh`, and it is built in: it is signed with NKeeper's official key and installed with NKeeper.

## Layout

```text
backend/        the plugin: SshPlugin.cs, nkeeper-plugin.json (the manifest), the project
frontend/       the terminal view: a Vite library build to dist/index.js
tests/          the backend tests, which start NKeeper's test server with this plugin loaded
tests/e2e/     a real-browser test, terminal.mjs, for an installed NKeeper
.github/        CI and the release workflow
```

It builds against NKeeper's published SDK, never core's source:

| Package | Registry | Used for |
| --- | --- | --- |
| `NKeeper.Abstractions` | GitHub Packages (NuGet) | the backend SDK |
| `NKeeper.Plugin.Build` | GitHub Packages (NuGet) | builds, packs and signs the plugin |
| `NKeeper.Testing` | GitHub Packages (NuGet) | the test server and the OpenSSH container fixture |
| `@opencraft-solutions/nkeeper-sdk` | GitHub Packages (npm) | the frontend SDK, imported as `@nkeeper/sdk` |

## Building

You need the .NET 10 SDK (see `global.json`), Node.js 24 (see `.nvmrc`) and Docker for the tests.

GitHub Packages needs a token even for public packages. Make a GitHub personal access token that can read packages (`read:packages`) and give it to the build in the environment. It is never written to the repository:

```bash
export NUGET_USERNAME=<your GitHub user name>
export NUGET_AUTH_TOKEN=<token>
export NODE_AUTH_TOKEN=<token>
```

Then, from the repository root:

```bash
(cd frontend && npm ci && npm run lint && npm run typecheck)
dotnet restore NKeeper.Plugins.Ssh.slnx
dotnet build NKeeper.Plugins.Ssh.slnx --configuration Release
```

Building the plugin builds its frontend, packs `ssh-<version>.nkp` into `artifacts/plugins/` and signs it with a development key, which the build creates under `.dev/keys/` on first use. Both are ignored by git. To load the package in a development NKeeper, point `Plugins:Directory` at `artifacts/plugins` and `Plugins:DevelopmentTrustDirectory` at `.dev/keys`.

Pinned versions: NuGet packages are locked in `packages.lock.json` (CI restores with `--locked-mode`), and npm packages in `frontend/package-lock.json`. Run `dotnet restore --force-evaluate` after changing a version.

## Testing

```bash
dotnet test NKeeper.Plugins.Ssh.slnx --configuration Release
```

The tests need Docker. They start PostgreSQL and an OpenSSH server in containers, and NKeeper's test server (from `NKeeper.Testing`) with this plugin's built package loaded and a development key trusted. They check what the plugin contributes, and that a terminal opens at the size asked for, resizes and ends when the shell exits.

### The browser test

`tests/e2e/terminal.mjs` drives an installed NKeeper with headless Chromium: it signs in, creates a host with a saved credential, opens a terminal, trusts the server's key, checks it draws, runs commands and resizes, opens another with a typed credential, and deletes the host. It creates only its own host (`e2e-<random>`). CI doesn't run it. To run it against an NKeeper that has this plugin installed:

```bash
cd tests/e2e
npm install
npx playwright install chromium
BASE=https://nkeeper.example ADMIN_PASSWORD=… SSH_ADDRESS=… SSH_USER=… SSH_PASSWORD=… node terminal.mjs
```

`ADMIN_USER` (default `admin`), `BASE` (default `https://localhost`) and `SCREENSHOTS` (where the screenshots go, by default the script's folder) are optional. Use a throwaway account on the SSH server.

## Continuous integration

`.github/workflows/ci.yml` runs on every push to `main` and every pull request: it restores from GitHub Packages with the workflow's token, lints and type-checks the frontend, checks formatting, builds, runs the tests and packs the plugin signed with a development key, which it keeps as a build artifact.

## Releasing

1. Raise `version` in `backend/nkeeper-plugin.json` (and keep `frontend/package.json` in step), and merge it to `main`.
2. Tag the merge commit `v<version>`, for example `v0.2.0`, and push the tag.
3. `.github/workflows/release.yml` runs. It checks that the tag matches the manifest's version and is on `main`, builds and tests, and then, in the protected `release` environment, which needs an owner's approval, packs `ssh-<version>.nkp` signed with the official key and publishes a GitHub release with the package and its SHA-256.

The official key's private half, `NKEEPER_RELEASE_KEY` (a PEM), is a secret of the `release` environment. It is available only to the signing job, goes to memory-backed storage for the packing step alone, and is removed when that step ends. It is never in the repository or in a log.

NKeeper's installer picks the release up when its `deploy/plugins.lock` is updated with the new URL and SHA-256.

## License

MIT, as NKeeper. See [LICENSE](LICENSE).
