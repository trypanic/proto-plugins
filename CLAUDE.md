# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

Static [proto](https://moonrepo.dev/docs/proto) non-WASM plugin definitions for third-party CLI tools. Each top-level `<tool-id>.toml` is a standalone plugin consumed from other repositories via raw GitHub URLs in their `.prototools` `[plugins.tools]` table. Tool ID = TOML filename without extension (`golang-migrate`, `mongosh`, `psql`, `sentrux`, `skillshare`, `staticcheck`).

Changes here propagate to every downstream consumer pinned at `main`. Prefer small, verifiable edits.

## Plugin Authoring Rules

- Keep plugins static and declarative. Use `type = "cli"` for archive-based or standalone binary installs; only reach for custom logic if the tool genuinely needs it.
- Add a `[platform.<os>]` entry only when the upstream project actually publishes that release asset — verify against the tool's GitHub releases page before adding/removing.
- `archs` use proto/Rust names (`x86_64`, `aarch64`). Map to upstream archive labels (`amd64`, `arm64`, etc.) under `[install.arch]`.
- Keep `[install.exes.<name>] primary = true` aligned with the executable users should run.
- If upstream Git tags don't parse cleanly as proto versions, add `version-pattern` under `[resolve]`.
- When supported platforms change, update the plugin TOML, README support matrix + plugin-details section, and all CI workflows together.

## Local Validation

Validate against an isolated proto home so user state doesn't pollute the result:

```sh
repo_root="$(pwd)"
tmpdir="$(mktemp -d)"
mkdir -p "$tmpdir/user-home"
cat > "$tmpdir/.prototools" <<EOF
golang-migrate = "4.19.0"
mongosh = "2.8.3"
sentrux = "0.5.7"
skillshare = "0.19.5"
staticcheck = "2026.2.2"

[plugins.tools]
golang-migrate = "file://$repo_root/plugins/golang-migrate.toml"
mongosh = "file://$repo_root/plugins/mongosh.toml"
sentrux = "file://$repo_root/plugins/sentrux.toml"
skillshare = "file://$repo_root/plugins/skillshare.toml"
staticcheck = "file://$repo_root/plugins/staticcheck.toml"
EOF

cd "$tmpdir"
PROTO_HOME="$tmpdir/.proto" HOME="$tmpdir/user-home" proto install golang-migrate --config-mode local
PROTO_HOME="$tmpdir/.proto" HOME="$tmpdir/user-home" proto install mongosh --config-mode local
PROTO_HOME="$tmpdir/.proto" HOME="$tmpdir/user-home" proto install sentrux --config-mode local
PROTO_HOME="$tmpdir/.proto" HOME="$tmpdir/user-home" proto install skillshare --config-mode local
PROTO_HOME="$tmpdir/.proto" HOME="$tmpdir/user-home" proto install staticcheck --config-mode local

"$tmpdir/.proto/tools/golang-migrate/4.19.0/migrate" -version    # → 4.19.0
"$tmpdir/.proto/tools/mongosh/2.8.3/bin/mongosh" --version    # → 2.8.3
SENTRUX_SKIP_GRAMMAR_DOWNLOAD=1 PROTO_HOME="$tmpdir/.proto" HOME="$tmpdir/user-home" proto run sentrux -- --version    # → sentrux 0.5.7
"$tmpdir/.proto/tools/skillshare/0.19.5/skillshare" --version    # → skillshare v0.19.5
"$tmpdir/.proto/tools/staticcheck/2026.2.2/staticcheck/staticcheck" -version    # → staticcheck 2026.2.2 (0.8.2)
```

Use explicit tool versions, never `latest` — validation must be reproducible. Staticcheck is pinned to a patch release (`2026.2.2`): the plugin resolves only three-part tags, because upstream tags its `.0` releases with two parts (`2026.2`) and proto has no token for the raw tag. Only include `sentrux` in local validation on platforms it publishes: Linux `x86_64`/`aarch64`, macOS `aarch64`, and Windows `x86_64`. Only include `psql` on platforms `theseus-rs/postgresql-binaries` publishes: Linux and macOS `x86_64`/`aarch64`, and Windows `x86_64`. Only include `mongosh` on platforms it publishes: Linux and macOS `x86_64`/`aarch64`, and Windows `x86_64`.

## CI

Three workflows under `.github/workflows/` run on push to `main`: `proto-linux.yml` (Linux x86_64 + aarch64), `proto-macos.yml` (macOS x86_64 + aarch64), and `proto-windows.yml` (Windows x86_64 + aarch64). Each writes a temporary `.prototools` referencing `file://./plugins/<plugin>.toml` (the checked-out commit, not raw `main` URLs), runs `proto install --config-mode local`, then asserts each supported binary's version output. Sentrux is skipped on macOS x86_64 and Windows aarch64 because upstream does not publish those release assets. Staticcheck, psql, and mongosh are skipped on Windows aarch64 because upstream does not publish those release archives.

The workflows pass `cache: false` to `moonrepo/setup-toolchain@v0`. Caching `~/.proto` between runs would otherwise poison validation: setup-toolchain restores from a constant restore-key prefix (`moonrepo-toolchain-v3-<platform>-<arch>`) that ignores `cache-version`, so a broken install from a previous CI run survives plugin TOML fixes. Plugin tools are fast to install fresh, so caching is not worth the lost reproducibility.

When adding a plugin, update all workflows so the new tool is installed and verified.

## Adding a New Plugin

1. Confirm upstream publishes versioned release archives or standalone binaries for every target OS+arch.
2. Create `plugins/<tool-id>.toml` with `type = "cli"`, `[platform.*]`, `[install]` (+ `[install.arch]` if labels differ), `[install.exes.*]`, `[resolve]`.
3. Validate locally as above.
4. Add the tool to all CI workflows.
5. Update `README.md` (support matrix, install/run examples, plugin-details section).

## README Sync

`README.md` is the public consumer-facing source of truth. Update it whenever any consumer-facing behavior changes: tools added/removed, tool IDs, executable names, raw URLs, supported platforms/archs, usage commands, or CI coverage.
