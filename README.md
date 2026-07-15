# mirror-zig-toolchain

A GitHub composite action that copies a specific Zig toolchain onto a
GitHub Release **you** control — every tarball verified against the
official [minisign][minisign] key before upload.

It's a *producer* of a private Zig mirror. It does not install Zig into
a workflow; for that use [xyzzylabs/setup-zig][setup-zig] (which can
then *consume* the mirror — see below).

## Do you need this?

Normally you don't. The usual path — `setup-zig`, `anyzig`, or a plain
download — pulls Zig from `ziglang.org` or the community mirrors, and
that's the right answer for almost everyone.

You need this action if **either** of these bites you:

- **Your environment can only reach `github.com`.** Some sandboxed dev
  containers and locked-down CI runners sit behind an egress proxy that
  allows a `github.com`-shaped hole and nothing else — so they can reach
  neither `ziglang.org` nor the community mirrors, and every normal
  download fails. A Release on a repo they already pull from is
  reachable.
- **You pin a nightly (`-dev`) build and need it to stay fetchable.**
  `ziglang.org/builds` purges old nightlies, and the community mirrors
  keep them only so long. Once your pinned build ages out upstream, CI
  breaks. Copying the tarball onto a Release you own stops the clock —
  the bytes live as long as the pin does.

If neither applies, use `setup-zig` directly and move on.

## How it fits

```
        pin in build.zig.zon (.minimum_zig_version)
                        │
        ┌───────────────┴────────────────┐
        │  this action (the producer)     │   runs in CI on a pin bump
        │  fetch → minisign-verify →      │   → uploads verified tarballs
        │  upload to your GitHub Release  │     to a release you control
        └───────────────┬────────────────┘
                        │
        ┌───────────────┴────────────────┐
        │  a consumer                     │   setup-zig's `mirror:` input,
        │  reads the release assets       │   or a shell script in a
        │  (github.com-reachable)         │   restricted sandbox
        └─────────────────────────────────┘
```

## Quick start

Mirror the version pinned in `build.zig.zon` onto a `zig-toolchain`
release on the current repo:

```yaml
name: mirror-zig
on:
  workflow_dispatch:       # rerun after each pin bump
permissions:
  contents: read
jobs:
  mirror:
    runs-on: ubuntu-latest
    permissions:
      contents: write      # to manage the release
    steps:
      - uses: actions/checkout@v4
      - uses: xyzzylabs/mirror-zig-toolchain@v1
```

Explicit version and a subset of platforms:

```yaml
      - uses: xyzzylabs/mirror-zig-toolchain@v1
        with:
          version: 0.17.0-dev.1275+59a628c6d
          platforms: |
            x86_64-linux
            aarch64-linux
```

## Inputs

| Name            | Default                               | Description                                                                     |
|-----------------|---------------------------------------|---------------------------------------------------------------------------------|
| `version`       | _auto_                                | Zig version to mirror. Empty → read `.minimum_zig_version` from `version-file`.  |
| `version-file`  | `build.zig.zon`                       | File to read the version from when `version` is empty.                           |
| `platforms`     | the four x86_64/aarch64 × linux/macos | Space- or newline-separated `<arch>-<os>` targets, in Zig's tarball naming.      |
| `release-tag`   | `zig-toolchain`                       | Release tag to upload to. Created if missing; existing assets are kept.          |
| `release-title` | `Zig toolchain mirror`                | Title used only when the release is first created.                               |
| `mirror`        | `''`                                  | Optional single mirror base URL to try first. Cannot be `ziglang.org`.           |
| `token`         | `${{ github.token }}`                 | Token with `contents: write` to manage the release.                             |

## What it uploads

Per requested platform: the `.tar.xz` (or `.zip` for Windows) tarball
and its `.minisig` sidecar, plus one `SHA256SUMS` covering all tarballs.
Re-running for a new pin **adds** the new version's assets and leaves the
old ones in place, and **merges** the new checksums into `SHA256SUMS` —
so an in-flight branch still on the previous pin keeps working.

## How it verifies

For each tarball the action tries your optional `mirror`, then the live
community mirror list (hardcoded fallback if `ziglang.org` is down),
then `ziglang.org` itself. The first source whose bytes pass **both**
checks wins:

1. `minisign -V` against upstream's public key (signature + trusted
   comment).
2. The signed filename in the trusted comment matches the requested
   tarball — so a validly-signed *different* tarball can't be passed off
   under the requested name.

An unverifiable tarball is never uploaded — a lying or compromised
mirror cannot get bytes onto your release. `minisign` is installed via
`apt-get`/`brew` if absent (ubuntu and macOS runners work out of the
box).

## Consuming the mirror

**In CI (retention case)** — point setup-zig's `mirror:` input at the
release's download base URL. The asset names match Zig's tarball naming,
which is exactly what setup-zig fetches:

```yaml
- uses: xyzzylabs/setup-zig@v1
  with:
    version: 0.17.0-dev.1275+59a628c6d
    mirror: https://github.com/<owner>/<repo>/releases/download/zig-toolchain
```

**In a restricted sandbox (egress case)** — fetch the asset directly
and verify it yourself:

```sh
gh release download zig-toolchain --repo <owner>/<repo> \
  --pattern 'zig-x86_64-linux-<version>.tar.xz*'
minisign -Vm zig-x86_64-linux-<version>.tar.xz \
  -P RWSGOq2NVecA2UPNdBUZykf1CCb147pkmdtYxgb3Ti+JO/wCYvhbAb/U
```

## Not a public mirror

This action mirrors *your* pinned version onto *your* release. It is not
a hosted, browse-and-download Zig mirror, and it intentionally doesn't
try to be — hosting arbitrary Zig builds for the world is a different
(and much heavier) job. If you just want Zig in CI, that's
[setup-zig][setup-zig]; if you want to run a real public mirror, see the
[community mirror docs][mirrors].

## License

MIT. See [LICENSE](LICENSE).

[minisign]: https://jedisct1.github.io/minisign/
[setup-zig]: https://github.com/xyzzylabs/setup-zig
[mirrors]: https://github.com/ziglang/www.ziglang.org/blob/main/MIRRORS.md
