# mirror-zig-toolchain

A GitHub composite action that fetches a pinned Zig toolchain, verifies
every byte against the official [minisign][minisign] key, and uploads it
to a GitHub Release you control.

It exists for two situations the normal download path doesn't cover:

- **Restricted egress.** Some environments (sandboxed dev containers,
  locked-down CI) only allow a `github.com`-shaped hole and can reach
  neither `ziglang.org` nor the community mirrors. Mirroring the pinned
  tarball onto *your own* repo's Release makes it fetchable there.
- **Retention.** `ziglang.org/builds` purges old `-dev` (nightly)
  builds, and the community mirrors keep them only so long. Copying the
  tarball onto a Release you control removes the retention clock — the
  bytes stay fetchable for as long as the pin lives in your
  `build.zig.zon`.

It does **not** install Zig into a workflow — for that, use
[xyzzylabs/setup-zig][setup-zig]. This action *produces* a mirror;
setup-zig (or a small bootstrap script) *consumes* one.

## Usage

Mirror the version pinned in `build.zig.zon` onto a `zig-toolchain`
release on the current repo:

```yaml
name: mirror-zig
on:
  workflow_dispatch:
permissions:
  contents: write
jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: xyzzylabs/mirror-zig-toolchain@v1
```

Pin an explicit version and a subset of platforms:

```yaml
      - uses: xyzzylabs/mirror-zig-toolchain@v1
        with:
          version: 0.17.0-dev.1275+59a628c6d
          platforms: |
            x86_64-linux
            aarch64-linux
```

## Inputs

| Name            | Default                          | Description                                                                                     |
|-----------------|----------------------------------|-------------------------------------------------------------------------------------------------|
| `version`       | _auto_                           | Zig version to mirror. Empty → read `.minimum_zig_version` from `version-file`.                 |
| `version-file`  | `build.zig.zon`                  | File to read the version from when `version` is empty.                                           |
| `platforms`     | the four x86_64/aarch64 × linux/macos | Space- or newline-separated `<arch>-<os>` targets, in Zig's tarball naming.                 |
| `release-tag`   | `zig-toolchain`                  | Release tag to upload to. Created if missing; existing assets are kept.                          |
| `release-title` | `Zig toolchain mirror`           | Title used only when the release is first created.                                              |
| `mirror`        | `''`                             | Optional single mirror base URL to try first. Cannot be `ziglang.org`.                          |
| `token`         | `${{ github.token }}`            | Token with `contents: write` to manage the release.                                             |

## What it uploads

For each requested platform: the `.tar.xz` (or `.zip` for Windows)
tarball and its `.minisig` sidecar, plus a single `SHA256SUMS` covering
all tarballs. Re-running for a new pin adds the new version's assets and
leaves the previous ones in place, so an in-flight branch on the old pin
keeps working.

## How the verification works

For every tarball, the action tries your optional `mirror`, then the
live community mirror list (hardcoded fallback if `ziglang.org` is
down), then `ziglang.org` itself. The first source whose bytes pass
**both** checks wins:

1. `minisign -V` against upstream's public key (signature + trusted
   comment).
2. The signed filename in the trusted comment matches the requested
   tarball — so a validly-signed but *different* tarball can't be passed
   off under the requested name.

An unverifiable tarball is never uploaded. Requires `minisign` on the
runner; the action installs it via `apt-get` or `brew` if absent
(ubuntu and macOS runners work out of the box).

## Consuming the mirror

In CI, point [xyzzylabs/setup-zig][setup-zig] at your release, or in a
restricted local sandbox, fetch the asset directly, e.g.:

```sh
gh release download zig-toolchain \
  --repo <owner>/<repo> \
  --pattern 'zig-x86_64-linux-<version>.tar.xz*'
minisign -Vm zig-x86_64-linux-<version>.tar.xz \
  -P RWSGOq2NVecA2UPNdBUZykf1CCb147pkmdtYxgb3Ti+JO/wCYvhbAb/U
```

## License

MIT. See [LICENSE](LICENSE).

[minisign]: https://jedisct1.github.io/minisign/
[setup-zig]: https://github.com/xyzzylabs/setup-zig
