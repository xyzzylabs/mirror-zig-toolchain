# mirror-zig-toolchain

A GitHub composite action that copies a specific Zig toolchain onto a
GitHub Release **you** control — every tarball verified against the
official [minisign][minisign] key before upload.

It produces a private Zig mirror; it does not install Zig. To install
Zig in a workflow, use [xyzzylabs/setup-zig][setup-zig] (which can also
consume the mirror — see [Consuming](#consuming-the-mirror)).

## Do you need this?

Usually no — `setup-zig`, `anyzig`, or a plain download from
`ziglang.org` / the community mirrors is the right answer. Reach for
this only if:

- **Your runner can only reach `github.com`.** Some sandboxed
  containers and locked-down CI sit behind an egress proxy that blocks
  `ziglang.org` and the community mirrors, so every normal download
  fails. A Release on a repo they already pull from is reachable.
- **You pin a nightly (`-dev`) build.** `ziglang.org/builds` purges old
  nightlies; once yours ages out, CI breaks. A Release you own keeps the
  bytes fetchable as long as the pin lives in `build.zig.zon`.

## Usage

Mirror the version pinned in `build.zig.zon` to a `zig-toolchain`
release on the current repo (rerun after each pin bump):

```yaml
name: mirror-zig
on:
  workflow_dispatch:
permissions:
  contents: read
jobs:
  mirror:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - uses: xyzzylabs/mirror-zig-toolchain@v1
```

With an explicit version and platform subset:

```yaml
      - uses: xyzzylabs/mirror-zig-toolchain@v1
        with:
          version: 0.17.0-dev.1275+59a628c6d
          platforms: |
            x86_64-linux
            aarch64-linux
```

## Inputs

| Name            | Default                               | Description                                                             |
|-----------------|---------------------------------------|-------------------------------------------------------------------------|
| `version`       | _auto_                                | Version to mirror. Empty → `.minimum_zig_version` from `version-file`.   |
| `version-file`  | `build.zig.zon`                       | File read when `version` is empty.                                       |
| `platforms`     | x86_64/aarch64 × linux/macos          | Space- or newline-separated `<arch>-<os>` targets.                       |
| `release-tag`   | `zig-toolchain`                       | Release to upload to. Created if missing; existing assets kept.          |
| `release-title` | `Zig toolchain mirror`                | Title used only when first creating the release.                         |
| `mirror`        | `''`                                  | Optional mirror base URL to try first. Cannot be `ziglang.org`.          |
| `token`         | `${{ github.token }}`                 | Token with `contents: write`.                                           |

## Behaviour

Per platform it uploads the tarball, its `.minisig`, and a merged
`SHA256SUMS`. Re-running for a new pin adds the new assets and keeps the
old ones, so a branch still on the previous pin keeps working.

Each tarball is fetched from your `mirror`, then the community mirror
list, then `ziglang.org`, and is only uploaded once it passes minisign
verification **and** its signed trusted-comment names the exact file
requested — so a lying mirror can't get bytes onto your release.
`minisign` is installed via `apt-get`/`brew` if absent.

## Consuming the mirror

In CI, point setup-zig's `mirror:` input at the release download base
(the asset names already match what setup-zig fetches):

```yaml
- uses: xyzzylabs/setup-zig@v1
  with:
    version: 0.17.0-dev.1275+59a628c6d
    mirror: https://github.com/<owner>/<repo>/releases/download/zig-toolchain
```

In a restricted sandbox, download and verify directly:

```sh
gh release download zig-toolchain --repo <owner>/<repo> \
  --pattern 'zig-x86_64-linux-<version>.tar.xz*'
minisign -Vm zig-x86_64-linux-<version>.tar.xz \
  -P RWSGOq2NVecA2UPNdBUZykf1CCb147pkmdtYxgb3Ti+JO/wCYvhbAb/U
```

## License

MIT. See [LICENSE](LICENSE). Not a public hosted mirror — it mirrors
*your* pin onto *your* release, nothing more.

[minisign]: https://jedisct1.github.io/minisign/
[setup-zig]: https://github.com/xyzzylabs/setup-zig
