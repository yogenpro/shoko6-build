# shoko6-build

Builds **Shoko Server 6.x** as a self-contained `win-x64` binary, because
upstream does not ship one.

## Why this exists

Shoko 6.0 has no stable release. As of 2026-09-26 the newest published
ShokoServer release is **v5.3.3** (March 2026); the 487 `v6.0.0-dev.*` tags
exist as git tags with **no release assets**. Upstream's daily build
(`build-daily.yml`) produces a `Shoko.CLI` for `linux-x64`, `linux-arm64`,
and `osx-arm64` — **no `win-x64`**. The only Windows artifacts are
`Shoko.TrayService`, a tray application and the wrong shape for a headless
service. Artifact downloads also require GitHub authentication.

So: build it yourself. This repo is a ~30-line workflow, not a fork.

## What it does

```
checkout ShokoAnime/ShokoServer @ v6.0.0-dev.487   (pinned, not a fork)
  ↓
setup .NET 10.0.x
  ↓
download Shoko-WebUI 2.6.0-dev.145, inject into Shoko.Server/webui
  ↓
dotnet publish -c Release -r win-x64 -f net10.0 --self-contained true Shoko.CLI
  ↓
verify webui + Shoko.CLI.exe actually present
  ↓
upload artifact (30-day retention)
```

One publish step, about ten minutes on a GitHub runner.

## Usage

Builds are manual:

```sh
gh workflow run build.yml
gh run watch
gh run download
```

To test a candidate version without editing the pins:

```sh
gh workflow run build.yml -f shoko_ref=v6.0.0-dev.490 -f webui_version=2.6.0-dev.150
```

Each artifact contains `BUILD-PROVENANCE.json` recording the resolved
commit SHA, WebUI version, RID, and build time.

## No secrets

The build has **no `secrets:` block at all**. Upstream substitutes a shared
maintainer TMDB key and a Sentry DSN into `Constants.cs` for the community
binary; both are optional here.

There is deliberately **no TMDB key**, which means Shoko uses **AniDB
ordering** — the default for a fresh install. Adding a key changes season
and episode numbering, so treat it as a migration decision rather than a
build fix. See `AGENTS.md`.

## Output

A self-contained `win-x64` publish directory — no .NET runtime required on
the target host. Contents are the standard Shoko layout: `Shoko.CLI.exe`,
`webui/`, `plugins/`, `Dependencies/`, plus the provenance file.

## Upgrading

Edit the two pins in `.github/workflows/build.yml`:

```yaml
env:
  SHOKO_REF: v6.0.0-dev.487        # must be a v6.0.0-dev.* tag
  WEBUI_VERSION: 2.6.0-dev.145     # must be the matching 2.6.0-dev.* line
```

`Shoko.Server 6.0.0-dev.*` pairs with `Shoko-WebUI 2.6.0-dev.*`; the
`2.5.x` WebUI line belongs to Shoko 5.x.

Commit, push, then trigger the workflow. The full procedure, a failure-mode
table, install guidance, and ecosystem gotchas (exFAT, symlinks, the legacy
scanner's per-file symlink limitation) are in
**[AGENTS.md](AGENTS.md)**.

## Status

Pinned to `v6.0.0-dev.487` / WebUI `2.6.0-dev.145`. The first build has not
been run yet — the pins are the author's initial choice, not a tested build.
Treat the first run as the feasibility test for the .NET 10 toolchain.

## Not affiliated with Shoko

This is an unofficial convenience build. It contains no Shoko source; the
workflow fetches upstream at a pinned ref. See
[ShokoAnime/ShokoServer](https://github.com/ShokoAnime/ShokoServer) for the
project itself and its GPL-3.0 license.

## License

MIT — see [LICENSE](LICENSE). Covers this repository only; the Shoko Server
source it builds is GPL-3.0 and licensed separately by its authors.
