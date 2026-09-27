# AGENTS.md — operating notes for this build repo

Guidance for an AI agent (or a human) asked to bump the Shoko Server /
Shoko-WebUI version, diagnose a failed build, or decide whether to adopt
this at all. Read this before changing anything.

## What this repo is, and what it deliberately is not

A **build-only** repo. It contains one workflow and no source. The workflow
checks out `ShokoAnime/ShokoServer` at a pinned ref and builds it.

It is **not a fork**, and that is load-bearing. Reasons, in case anyone is
tempted to "simplify" it into one:

- A fork means owning 2,723 upstream blobs and resolving merge conflicts
  against a project shipping ~3 commits/day. We would inherit that churn
  and never benefit from it.
- Upstream's `build-release.yml` contains `plugin-nuget`, `queue-nuget`,
  `buildtools-nuget`, and `buildtools-targets-nuget` jobs that push to
  Shoko's own NuGet feeds. Those need `NUGET_API_KEY` and would fail on a
  fork. We skip them entirely.
- We do not redistribute their code. We build it and run it ourselves.

`actions/checkout` can fetch any public repo at any ref, so a fork is
unnecessary. Upgrading is a one-line diff.

## The two pins

Everything version-related lives in the `env:` block near the top of
`.github/workflows/build.yml`:

```yaml
env:
  SHOKO_REF: v6.0.0-dev.487
  WEBUI_VERSION: 2.6.0-dev.145
  DOTNET_VERSION: '10.0.x'
  RID: win-x64
```

Only `SHOKO_REF` and `WEBUI_VERSION` change during an upgrade.

## Upgrade procedure

### 1. Find the newest upstream tag

```sh
gh api repos/ShokoAnime/ShokoServer/tags --jq '.[].name' | grep '^v6\.0\.0-dev' | head -5
```

Expect ~3 new tags per day. Do not build a ref you have not read the release
notes for; the 6.x line is a full rewrite and dev tags can change behaviour
daily.

### 2. Find the matching WebUI version

```sh
curl -sS https://raw.githubusercontent.com/ShokoAnime/Shoko-WebUI/metadata/manifest.json \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['Dev'][0]['version'])"
```

**Version-line rule — get this wrong and the UI breaks silently:**
`Shoko.Server 6.0.0-dev.*` pairs with `Shoko-WebUI 2.6.0-dev.*`. The `2.5.x`
line pairs with Shoko **5.x**. A 2.5.x WebUI on a 6.x server is a two-major
mismatch.

### 3. Confirm the WebUI release exists

The download URL is derived, not stored:

```
https://github.com/ShokoAnime/Shoko-WebUI/releases/download/v${WEBUI_VERSION}/Shoko-WebUI-v${WEBUI_VERSION}.zip
```

Verify before committing the pin (a 404 fails the build late, after a
several-minute publish):

```sh
curl -sI -o /dev/null -w '%{http_code}\n' "https://github.com/ShokoAnime/Shoko-WebUI/releases/download/v2.6.0-dev.146/Shoko-WebUI-v2.6.0-dev.146.zip"
```

Expect `302` (redirect to the asset CDN). `200` also fine. `404` means the
tag does not exist yet.

### 4. Edit the two lines, commit, push

```sh
$EDITOR .github/workflows/build.yml   # change SHOKO_REF and WEBUI_VERSION
git add .github/workflows/build.yml
git commit -m "build: shoko ${NEW_REF} + webui ${NEW_WEBUI}"
git push
```

### 5. Trigger the build

Builds are **manual only** (`workflow_dispatch`). A committed bump does not
build by itself, deliberately: with a moving ref we would rebuild three times
a day for no reason.

```sh
gh workflow run build.yml
# or override the pins for a one-off trial without committing:
gh workflow run build.yml -f shoko_ref=v6.0.0-dev.490 -f webui_version=2.6.0-dev.150

gh run watch        # watch the newest run
gh run download     # fetch the artifact
```

The `-f` inputs exist so you can test a candidate version without dirtying
the pinned defaults. This is the preferred way to evaluate an upgrade.

### 6. Read the provenance file

Every artifact contains `BUILD-PROVENANCE.json` with the resolved commit
SHA, WebUI version, RID, and a UTC timestamp. A downloaded zip is
self-describing months later — check it before installing.

## Reading a build failure

| Symptom | Likely cause | Action |
|---|---|---|
| `error NETSDK1045` / "current .NET SDK does not support net10.0" | `DOTNET_VERSION` too low | Bump to `10.0.x`; SDK must match the target framework |
| `webui/index.html missing after injection` | `WEBUI_VERSION` tag does not exist | Verify the URL returns 302/200 (step 3), fix the pin |
| `webui looks truncated: only N files` | Zip downloaded but truncated | Re-run; check `WEBUI_VERSION` pin |
| `webui did not reach publish output` | csproj `Content Include` changed upstream | Inspect `Shoko.Server/Shoko.Server.csproj` for the `webui\**\*` item |
| `NU1202` / package restore errors | `Shoko.BuildTools` source generator changed | Read the upstream commit log for the range you skipped |
| `Shoko.CLI.exe missing from publish output` | Upstream renamed the entry point | Check `Shoko.CLI/Shoko.CLI.csproj` `AssemblyName`; upstream's 6.x installer references `ShokoServer.exe`, so a rename is possible |

**Two assertions guard against a silent-success trap.** Upstream's repo
ships a placeholder `Shoko.Server/webui/index.html`, so the build succeeds
even when the real WebUI never downloads. The workflow therefore asserts
file counts in two places — after injection and in the publish output —
rather than trusting exit codes. If you see those assertions fail, the
build "passed" everything else but produced a server with no UI.

**If the publish step itself fails, do not paper over it.** The .NET 10
toolchain is bleeding edge and `Shoko.BuildTools` runs source generation;
a genuine incompatibility is possible. Report the actual compiler output.

## Why there are no secrets

Upstream's release workflow substitutes a maintainer-supplied shared key
into `Constants.cs` for the community binary. Both substitutions are
optional for a private build and both are intentionally absent here:

- **TMDB** — `Constants.cs` holds the placeholder
  `TMDB_API_KEY_GOES_HERE`, and its own comment says "or insert the key in
  your settings". Without a key, Shoko falls back to **AniDB ordering**,
  which is the default for a fresh install. Supplying a key changes season
  and episode numbering for any series that gains a TMDB link, so treat it
  as a deliberate migration rather than a build fix.
- **Sentry** — no DSN means no crash reporting. Set `SentryOptOut` in the
  server settings to be explicit.

To add your own TMDB key: create a repo secret `TMDB_API`, then re-add the
step (upstream's script is
`./.github/workflows/ReplaceTmdbApiKey.ps1 -apiKey ${{ secrets.TMDB_API }}`)
immediately before the publish step. Expect episode renumbering afterwards.

## The offline-importer plugin is deliberately absent

Upstream's build fetches `revam/dotnet-shoko-plugin-offline-importer` and
injects it. **Do not add it without checking its `manifest.json`
`abstraction` field first.** Plugins hard-pin an abstraction version — the
ShokoRelay plugin declares `abstraction: 6.0.0` on all three archives and
will not load against a mismatched server. The offline-importer's releases
predate the 6.x line, so they may target 5.x abstractions. Its purpose is
bulk AniDB title import for offline series ID resolution, which is not
needed when importing files you already have locally.

## Artifact retention

`retention-days: 30`. Download promptly. To keep a copy permanently,
publish it as a release instead:

```sh
gh release create shoko-6.0.0-dev.487 dist.zip
```

## Install notes (general)

The artifact is self-contained, so the target host needs no .NET runtime.

Two things to settle before installing over an existing Shoko 5.x:

- **Run side by side first.** Give 6.x its own data directory and its own
  port, and do not register it as a service yet. Keep 5.x serving until the
  6.x instance is proven.
- **The data directory is not migratable.** There is no documented 5.x→6.x
  database migration; 6.x is a rewrite. Treat the catalog as rebuildable by
  re-importing from your media. Take a full backup of the 5.x install
  (binaries *and* data directory) before starting — that backup is the
  rollback.

If you need a Windows service, NSSM works; point `Application` at
`Shoko.CLI.exe`, set `AppDirectory` to the publish folder, and use the
server's home/data directory environment variable. 6.x has no built-in
service installer, so service wiring is yours to manage.

## Ecosystem notes

Facts about the 6.x ecosystem that are easy to get wrong. They are recorded
here because they are not obvious from the upstream docs.

- **The plugin VFS must live on a filesystem that supports reparse
  points.** ShokoRelay's VFS is created *inside* a managed import folder —
  `VfsBuilder` does `Path.Combine(ImportRoot, rootFolderName)` and
  `ResolveImportRootPath` derives `ImportRoot` from the managed folder's own
  path. The `VFS Root Path` setting is a *folder name*, not an absolute
  path. So if your import folders are on exFAT, the VFS cannot be created
  there. A workable layout is an NTFS directory containing one symlink per
  release folder, pointed at the real media elsewhere.
- **exFAT supports neither hardlinks nor symlinks.** Hardlinks also cannot
  cross volumes, so a hardlink-based VFS is not a workaround for exFAT
  media.
- **The legacy `ShokoRelay.bundle` scanner (v1.2.36) does not recognise
  per-file symlinks.** It logs the folder and lists the files, then processes
  none of them. *Directory* symlinks work fine. The modern plugin exists
  partly to fix this class of layout problem, and it is why moving to 6.x
  is often worth it.
- **The legacy scanner matches files by `parent folder name + filename`**
  and takes series/season/episode from the Shoko API response, not from the
  folder layout. Ancestor directories above the immediate parent are
  therefore irrelevant to matching, but the *immediate* parent folder name
  must be preserved or matching breaks.
