# proto

Static [proto](https://moonrepo.dev/docs/proto) plugin definitions for third-party CLI tools that are not managed by proto core.

The plugin files live under `plugins/`. Each TOML file is a non-WASM proto plugin that describes how proto should resolve versions, download release archives or standalone binaries, unpack when needed, and expose the executable through proto shims and bins.

## Tools

| Tool | Plugin file | Executable | Upstream project | Supported platforms |
| --- | --- | --- | --- | --- |
| `golang-migrate` | `plugins/golang-migrate.toml` | `migrate` | `golang-migrate/migrate` | Linux, macOS, and Windows on `x86_64` and `aarch64` |
| `mongosh` | `plugins/mongosh.toml` | `mongosh` | `mongodb-js/mongosh` | Linux and macOS on `x86_64` and `aarch64`; Windows on `x86_64` |
| `psql` | `plugins/psql.toml` | `psql` | `theseus-rs/postgresql-binaries` | Linux and macOS on `x86_64` and `aarch64`; Windows on `x86_64` |
| `sentrux` | `plugins/sentrux.toml` | `sentrux` | `sentrux/sentrux` | Linux on `x86_64` and `aarch64`; macOS on `aarch64`; Windows on `x86_64` |
| `skillshare` | `plugins/skillshare.toml` | `skillshare` | `runkids/skillshare` | Linux, macOS, and Windows on `x86_64` and `aarch64` |
| `staticcheck` | `plugins/staticcheck.toml` | `staticcheck` | `dominikh/go-tools` | Linux and macOS on `x86_64` and `aarch64`; Windows on `x86_64` |

## Requirements

- `proto` installed and activated in your shell.
- `git`, because these plugins resolve available versions from upstream Git tags.
- Archive tools supported by proto for unpacking `.tar.gz` and `.zip` releases.

See the official proto install guide if proto is not installed yet:

```sh
bash <(curl -fsSL https://moonrepo.dev/install/proto.sh)
```

## Enable the plugins

Add the plugins to a `.prototools` file in the repository that wants to use them.

Use the raw GitHub URLs for this repository:

```toml
golang-migrate = "<version>"
mongosh = "<version>"
psql = "<version>"
sentrux = "<version>"
skillshare = "<version>"
staticcheck = "<version>"

[plugins.tools]
golang-migrate = "https://raw.githubusercontent.com/trypanic/proto-plugins/main/plugins/golang-migrate.toml"
mongosh = "https://raw.githubusercontent.com/trypanic/proto-plugins/main/plugins/mongosh.toml"
psql = "https://raw.githubusercontent.com/trypanic/proto-plugins/main/plugins/psql.toml"
sentrux = "https://raw.githubusercontent.com/trypanic/proto-plugins/main/plugins/sentrux.toml"
skillshare = "https://raw.githubusercontent.com/trypanic/proto-plugins/main/plugins/skillshare.toml"
staticcheck = "https://raw.githubusercontent.com/trypanic/proto-plugins/main/plugins/staticcheck.toml"
```

Replace:

- `<version>` with a release version from the upstream tool. Staticcheck must be pinned to a patch release such as `2026.2.2`; its `.0` releases are not installable (see the staticcheck section).

For example, when pinned in a project-level `.prototools`, running `proto install` from that project will install the configured tools.

## Install and run

Install everything pinned in the current `.prototools` context:

```sh
proto install
```

Install one tool explicitly:

```sh
proto install golang-migrate <version>
proto install mongosh <version>
proto install psql <version>
proto install sentrux <version>
proto install skillshare <version>
proto install staticcheck <version>
```

Run through proto:

```sh
proto run golang-migrate -- -version
proto run mongosh -- --version
proto run psql -- --version
proto run sentrux -- --version
proto run skillshare -- --help
proto run staticcheck -- -version
```

After proto shims are active in `PATH`, the primary executable can also be called directly:

```sh
migrate -version
mongosh --version
psql --version
sentrux --version
skillshare --help
staticcheck -version
```

## CI validation

Pushes to `main` run GitHub Actions validation for the checked-out plugin definitions:

- Linux `x86_64` on `ubuntu-24.04`
- Linux `aarch64` on `ubuntu-24.04-arm`
- macOS `x86_64` on `macos-15-intel`
- macOS `aarch64` on `macos-15`
- Windows `x86_64` on `windows-2025`
- Windows `aarch64` on `windows-11-arm`

Each job installs proto, creates a temporary `.prototools` that points to the checked-out TOML plugin files under `plugins/`, installs supported plugin tools, and verifies the installed binaries. Mongosh is validated on Linux and macOS for both configured architectures, and on Windows `x86_64`. Psql is validated on Linux and macOS for both configured architectures, and on Windows `x86_64`. Sentrux is validated on Linux for both configured architectures, on macOS `aarch64`, and on Windows `x86_64`. Staticcheck is validated on Linux and macOS for both configured architectures, and on Windows `x86_64`.

## Plugin details

### golang-migrate

`plugins/golang-migrate.toml` installs the `migrate` binary from GitHub release archives:

- Linux: `migrate.linux-{arch}.tar.gz`
- macOS: `migrate.darwin-{arch}.tar.gz`
- Windows: `migrate.windows-{arch}.zip`

The plugin maps proto/Rust architecture names to the release archive naming used by `golang-migrate`:

| proto arch | Release arch |
| --- | --- |
| `x86_64` | `amd64` |
| `aarch64` | `arm64` |

The plugin resolves available versions from Git tags in `https://github.com/golang-migrate/migrate` and downloads archives from:

```text
https://github.com/golang-migrate/migrate/releases/download/v{version}/{download_file}
```

### mongosh

`plugins/mongosh.toml` installs the `mongosh` (MongoDB Shell) binary from the MongoDB downloads server:

- Linux: `mongosh-{version}-linux-{arch}.tgz`
- macOS: `mongosh-{version}-darwin-{arch}.zip`
- Windows: `mongosh-{version}-win32-{arch}.zip`

The upstream archives unpack to a `mongosh-{version}-{platform}-{arch}/` directory containing the executable under `bin/`. The plugin maps proto/Rust architecture names to the release archive naming used by mongosh:

| proto arch | Release arch |
| --- | --- |
| `x86_64` | `x64` |
| `aarch64` | `arm64` |

The plugin resolves available versions from Git tags in `https://github.com/mongodb-js/mongosh` and downloads archives from:

```text
https://downloads.mongodb.com/compass/{download_file}
```

### psql

`plugins/psql.toml` installs the PostgreSQL `psql` client binary from GitHub release archives published by `theseus-rs/postgresql-binaries`:

- Linux: `postgresql-{version}-{arch}-unknown-linux-gnu.tar.gz`
- macOS: `postgresql-{version}-{arch}-apple-darwin.tar.gz`
- Windows: `postgresql-{version}-{arch}-pc-windows-msvc.tar.gz`

The plugin uses proto/Rust architecture names directly because the release archive names use Rust target triples.

The plugin resolves available versions from Git tags in `https://github.com/theseus-rs/postgresql-binaries` and downloads archives from:

```text
https://github.com/theseus-rs/postgresql-binaries/releases/download/{version}/{download_file}
```

### sentrux

`plugins/sentrux.toml` installs the `sentrux` binary from GitHub release assets. Sentrux publishes standalone executable binaries instead of archive files:

- Linux: `sentrux-linux-{arch}`
- macOS: `sentrux-darwin-arm64`
- Windows: `sentrux-windows-x86_64.exe`

The plugin uses proto/Rust architecture names directly where upstream release names do the same. macOS currently only publishes an `arm64` binary, so the plugin supports proto `aarch64` on macOS and does not configure macOS `x86_64`.

The plugin resolves available versions from Git tags in `https://github.com/sentrux/sentrux` and downloads binaries from:

```text
https://github.com/sentrux/sentrux/releases/download/v{version}/{download_file}
```

### skillshare

`plugins/skillshare.toml` installs the `skillshare` binary from GitHub release archives:

- Linux: `skillshare_{version}_linux_{arch}.tar.gz`
- macOS: `skillshare_{version}_darwin_{arch}.tar.gz`
- Windows: `skillshare_{version}_windows_{arch}.zip`

The plugin maps proto/Rust architecture names to the release archive naming used by `skillshare`:

| proto arch | Release arch |
| --- | --- |
| `x86_64` | `amd64` |
| `aarch64` | `arm64` |

The plugin resolves available versions from Git tags in `https://github.com/runkids/skillshare` and downloads archives from:

```text
https://github.com/runkids/skillshare/releases/download/v{version}/{download_file}
```

### staticcheck

`plugins/staticcheck.toml` installs the `staticcheck` binary from GitHub release archives. Staticcheck tags its `.0` releases with two parts (`2026.2`) and its patch releases with three (`2026.2.2`). Proto has no token for the raw tag, so one download URL cannot serve both forms. The plugin resolves only the three-part patch tags and downloads them by `{version}`; a `.0` release such as `2026.1.0` or `2026.2.0` is not installable. Pin a patch release, such as `2026.2.2`, the first that reads Go 1.27.2+ export data.

- Linux: `staticcheck_linux_{arch}.tar.gz`
- macOS: `staticcheck_darwin_{arch}.tar.gz`
- Windows: `staticcheck_windows_{arch}.tar.gz`

The upstream archives contain the executable under a `staticcheck/` directory. The plugin maps proto/Rust architecture names to the release archive naming used by Staticcheck:

| proto arch | Release arch |
| --- | --- |
| `x86_64` | `amd64` |
| `aarch64` | `arm64` |

The plugin resolves available versions from Git tags in `https://github.com/dominikh/go-tools` and downloads archives from:

```text
https://github.com/dominikh/go-tools/releases/download/{version}/{download_file}
```

## Development notes

- Keep these plugins static and declarative unless a tool needs custom logic. Simple CLI archive installs fit proto's non-WASM plugin format.
- Add a `[platform.*]` entry only when the upstream project publishes a matching archive or standalone binary.
- Keep `[install.exes.<name>] primary = true` aligned with the executable users should run by default.
- If upstream release archive names use different architecture labels, map them under `[install.arch]`.
- If upstream tags do not parse cleanly as proto versions, add a `version-pattern` under `[resolve]`.

## References

- [proto documentation](https://moonrepo.dev/docs/proto)
- [proto plugins](https://moonrepo.dev/docs/proto/plugins)
- [proto non-WASM plugins](https://moonrepo.dev/docs/proto/non-wasm-plugin)
- [proto configuration](https://moonrepo.dev/docs/proto/config)
