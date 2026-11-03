# AGENTS.md

Instructions for coding agents (Claude Code, Codex, Kimi Code, Pi, OpenCode, and others) working in this repository. This is the single agent instruction file: do not add `CLAUDE.md` or other agent-specific copies. Treat `README.md` as the public, consumer-facing source of truth, and keep this file focused on how to maintain the repository safely.

## Repository Purpose

Static [proto](https://moonrepo.dev/docs/proto) non-WASM plugin definitions for third-party CLI tools.

- Each `plugins/<tool-id>.toml` file is a standalone proto plugin.
- The plugins are consumed from other repositories through raw GitHub URLs in their `.prototools` `[plugins.tools]` table.
- The tool ID is the TOML file name without the extension: `golang-migrate`, `mongosh`, `psql`, `sentrux`, `skillshare`, `staticcheck`, and `task`.
- Changes here propagate to every downstream consumer pinned at `main`, so prefer small, verifiable edits.

## Plugin Authoring Rules

- Keep plugins static and declarative. Use `type = "cli"` for archive-based or standalone binary installs; only reach for custom logic if the tool genuinely needs it.
- Add a `[platform.<os>]` entry only when the upstream project actually publishes that release asset. Verify against the tool's official releases page before adding or removing one.
- Do not remove an OS or architecture unless upstream no longer publishes a compatible asset or the plugin is known to fail.
- `archs` use proto/Rust names (`x86_64`, `aarch64`). Map them to upstream archive labels (`amd64`, `arm64`, `x64`, etc.) under `[install.arch]`.
- Keep `[install.exes.<name>] primary = true` aligned with the executable users should run.
- If upstream Git tags don't parse cleanly as proto versions, add `version-pattern` under `[resolve]`.
- When supported platforms change, update the plugin TOML, the README support matrix and plugin-details section, and all CI workflows together.

## Supported Platforms

| Tool | Linux | macOS | Windows |
| --- | --- | --- | --- |
| `golang-migrate` | `x86_64`, `aarch64` | `x86_64`, `aarch64` | `x86_64`, `aarch64` |
| `mongosh` | `x86_64`, `aarch64` | `x86_64`, `aarch64` | `x86_64` |
| `psql` | `x86_64`, `aarch64` | `x86_64`, `aarch64` | `x86_64` |
| `sentrux` | `x86_64`, `aarch64` | `aarch64` | `x86_64` |
| `skillshare` | `x86_64`, `aarch64` | `x86_64`, `aarch64` | `x86_64`, `aarch64` |
| `staticcheck` | `x86_64`, `aarch64` | `x86_64`, `aarch64` | `x86_64` |
| `task` | `x86_64`, `aarch64` | `x86_64`, `aarch64` | `x86_64`, `aarch64` |

## Local Validation

Validate against an isolated proto home so user state doesn't pollute the result. Recent proto versions ignore plugins declared in an untrusted local `.prototools`, so the temporary directory is trusted through `PROTO_TRUSTED_PATHS` (CI is trusted automatically).

```sh
repo_root="$(pwd)"
tmpdir="$(mktemp -d)"
mkdir -p "$tmpdir/user-home"
cat > "$tmpdir/.prototools" <<EOF
golang-migrate = "4.19.0"
mongosh = "2.8.3"
psql = "18.3.0"
sentrux = "0.5.7"
skillshare = "0.19.5"
staticcheck = "2026.2.2"
task = "3.54.0"

[plugins.tools]
golang-migrate = "file://$repo_root/plugins/golang-migrate.toml"
mongosh = "file://$repo_root/plugins/mongosh.toml"
psql = "file://$repo_root/plugins/psql.toml"
sentrux = "file://$repo_root/plugins/sentrux.toml"
skillshare = "file://$repo_root/plugins/skillshare.toml"
staticcheck = "file://$repo_root/plugins/staticcheck.toml"
task = "file://$repo_root/plugins/task.toml"
EOF

cd "$tmpdir"
export PROTO_HOME="$tmpdir/.proto" HOME="$tmpdir/user-home" PROTO_TRUSTED_PATHS="$tmpdir"
proto install --config-mode local

"$tmpdir/.proto/tools/golang-migrate/4.19.0/migrate" -version    # → 4.19.0
"$tmpdir/.proto/tools/mongosh/2.8.3/bin/mongosh" --version    # → 2.8.3
"$tmpdir/.proto/tools/psql/18.3.0/bin/psql" --version    # → psql (PostgreSQL) 18.3
SENTRUX_SKIP_GRAMMAR_DOWNLOAD=1 proto run sentrux -- --version    # → sentrux 0.5.7
"$tmpdir/.proto/tools/skillshare/0.19.5/skillshare" --version    # → skillshare v0.19.5
"$tmpdir/.proto/tools/staticcheck/2026.2.2/staticcheck/staticcheck" -version    # → staticcheck 2026.2.2 (0.8.2)
"$tmpdir/.proto/tools/task/3.54.0/task" --version    # → 3.54.0
```

- Use explicit tool versions, never `latest`, so validation is reproducible.
- Staticcheck is pinned to a patch release (`2026.2.2`): the plugin resolves only three-part tags, because upstream tags its `.0` releases with two parts (`2026.2`) and proto has no token for the raw tag.
- Only include a tool in local validation on platforms it supports (see the table above).
- `psql` and `sentrux` link dynamically against `libssl.so.3` on Linux; a host without OpenSSL 3 fails to run them even when the install succeeds.

## CI

Three workflows under `.github/workflows/` run on push to `main`: `proto-linux.yml` (Linux `x86_64` + `aarch64`), `proto-macos.yml` (macOS `x86_64` + `aarch64`), and `proto-windows.yml` (Windows `x86_64` + `aarch64`).

- Each job writes a temporary `.prototools` referencing `file://./plugins/<plugin>.toml` (the checked-out commit, not raw `main` URLs), runs `proto install --config-mode local`, then asserts each supported binary's version output.
- Sentrux is skipped on macOS `x86_64` and Windows `aarch64` because upstream does not publish those release assets. Staticcheck, psql, and mongosh are skipped on Windows `aarch64` for the same reason.
- Keep runner labels aligned with [GitHub-hosted runner availability](https://docs.github.com/actions/reference/runners/github-hosted-runners).
- The workflows pass `cache: false` to `moonrepo/setup-toolchain@v0`. Caching `~/.proto` between runs would poison validation: setup-toolchain restores from a constant restore-key prefix (`moonrepo-toolchain-v3-<platform>-<arch>`) that ignores `cache-version`, so a broken install from a previous run survives plugin TOML fixes. Plugin tools install fast, so caching is not worth the lost reproducibility.
- When adding a plugin, update every workflow so the new tool is installed and verified.

## Adding a New Plugin

1. Confirm upstream publishes versioned release archives or standalone binaries for every target OS and architecture.
2. Create `plugins/<tool-id>.toml` with `type = "cli"`, `[platform.*]`, `[install]` (+ `[install.arch]` if labels differ), `[install.exes.*]`, and `[resolve]`.
3. Validate locally as above.
4. Add the tool to all CI workflows.
5. Update `README.md` (tools table, `.prototools` example, install/run examples, CI validation notes, plugin-details section).
6. Update this file (tool ID list, supported platforms table, local validation block, CI skip notes).

## Keeping Docs in Sync

`README.md` and `AGENTS.md` must describe the same tools, platforms, versions, and CI coverage. Update both whenever any of these change: tools added or removed, tool IDs, executable names, raw URLs, supported platforms or architectures, usage commands, pinned validation versions, or CI coverage.

## Sources

Prefer official sources when verifying behavior:

- Proto docs: `https://moonrepo.dev/docs/proto`
- Proto non-WASM plugins: `https://moonrepo.dev/docs/proto/non-wasm-plugin`
- Proto config and trust: `https://moonrepo.dev/docs/proto/config`
- GitHub-hosted runners: `https://docs.github.com/actions/reference/runners/github-hosted-runners`
- Upstream tool release pages linked from each plugin's `[resolve].git-url`
