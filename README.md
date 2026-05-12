# action-build

Reusable GitHub Actions workflows for the [unpins](https://unpins.org)
project. Each package repo (e.g. `unpins/jq`, `unpins/tree`, `unpins/unpin`)
calls into these so the build/release pipeline lives in one place.

## Workflows

### `build.yml`

Builds the package on every push or tag, verifies the artifact is portable
(statically linked on Linux, no `/nix/store` references on macOS, no companion
DLLs on Windows), and uploads it as a GitHub Actions artifact. Optionally
creates a GitHub Release on tag pushes.

```yaml
jobs:
  build:
    uses: unpins/action-build/.github/workflows/build.yml@main
    with:
      package_name: 'jq'
      create_release: ${{ startsWith(github.ref, 'refs/tags/') }}
    secrets:
      CACHIX_AUTH_TOKEN: ${{ secrets.CACHIX_AUTH_TOKEN }}
```

### `release.yml`

Manual-dispatch workflow. Reads the upstream version from
`nix eval .#default.version`, finds the highest `v<upstream>-N` tag, computes
`N+1`, pushes the new tag, and delegates to `build.yml` with
`create_release: true`.

```yaml
jobs:
  release:
    permissions:
      contents: write
    uses: unpins/action-build/.github/workflows/release.yml@main
    with:
      package_name: 'jq'
    secrets:
      CACHIX_AUTH_TOKEN: ${{ secrets.CACHIX_AUTH_TOKEN }}
```

## Inputs

| Input              | Type      | Default | Purpose                                                                                            |
| ------------------ | --------- | ------- | -------------------------------------------------------------------------------------------------- |
| `package_name`     | string    | —       | Name of the binary in `result/bin/`. Also the prefix of the published artifact.                    |
| `package_data`     | boolean   | `false` | Also publish `result/share` as a `.tar.zst` (man pages, completions, etc.).                        |
| `bootstrap_naming` | boolean   | `false` | Use `<pkg>-<arch>-<os>[.exe]` instead of `<pkg>-<version>-<os>-<arch>[.exe]`. See below.           |
| `create_release`   | boolean   | `false` | Create a GitHub Release in the calling repo (only `build.yml`).                                    |
| `platforms`        | JSON      | 4 native targets | Matrix of `{runner, os, arch, attr?}`. Add Windows by setting `attr` to a `pkgsCross` derivation. |

### `bootstrap_naming`

The default `<pkg>-<version>-<os>-<arch>[.exe]` is what `unpin install` parses
when picking the right asset from a GitHub release page — every package repo
should leave this **off**.

Only the `unpin` CLI itself sets `bootstrap_naming: true`. It ships with a
stable URL on `unpins.org` (`unpin-x86_64-linux`, `unpin-x86_64-windows.exe`,
…) because users download it by hand before they have `unpin` to install
anything.

## Portability gate

`build.yml` rejects an artifact that fails its OS-specific portability check:

- **Linux**: `file` must report `statically linked`.
- **macOS**: `otool -L` must not reference `/nix/store`.
- **Windows**: no companion `*.dll` next to the `.exe`; imports may only name
  system DLLs (uppercase, e.g. `KERNEL32.dll`).

The Nix flake in the calling repo is responsible for producing a binary that
passes — see the existing `unpins/*` repos for examples.

## License

MIT.
