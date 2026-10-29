# Agent Instructions

This repository publishes static Proto plugin definitions for third-party CLI tools. Treat `README.md` as the public source of truth for consumers, and keep this file focused on how agents should maintain or use the repository safely.

## Repository Purpose

- Each `plugins/*.toml` file is a Proto non-WASM tool plugin.
- The plugins are intended to be consumed from other repositories through raw GitHub URLs in `[plugins.tools]`.
- The supported public tool IDs are the TOML file names without the extension, for example `golang-migrate`, `psql`, `sentrux`, `skillshare`, and `staticcheck`.
- Changes here can affect multiple downstream projects, so prefer small, verifiable edits.

## Maintenance Rules

- Keep plugins static and declarative unless a tool genuinely requires custom logic.
- Before changing archive or binary names, platform support, or architecture mappings, verify the upstream release asset names from the tool's official releases.
- Use Proto/Rust architecture names in `platform.*.archs`, such as `x86_64` and `aarch64`.
- Use `[install.arch]` when upstream release assets use different architecture names, such as `amd64` or `arm64`.
- Keep `[install.exes.<name>] primary = true` aligned with the binary users should run.
- Do not remove an OS or architecture unless upstream no longer publishes a compatible asset or the plugin is known to fail.
- Avoid pinning `latest` in examples or tests. Use explicit versions so validation is reproducible.
- If supported platforms change, update the plugin TOML, README support matrix, plugin details, and CI matrix together.

## Local Validation

Validate against an isolated proto home so user state doesn't pollute the result:

```sh
repo_root="$(pwd)"
tmpdir="$(mktemp -d)"
mkdir -p "$tmpdir/user-home"
cat > "$tmpdir/.prototools" <<EOF
golang-migrate = "4.19.0"
psql = "18.3.0"
sentrux = "0.5.7"
skillshare = "0.19.5"
staticcheck = "2026.2.2"

[plugins.tools]
golang-migrate = "file://$repo_root/plugins/golang-migrate.toml"
psql = "file://$repo_root/plugins/psql.toml"
sentrux = "file://$repo_root/plugins/sentrux.toml"
skillshare = "file://$repo_root/plugins/skillshare.toml"
staticcheck = "file://$repo_root/plugins/staticcheck.toml"
EOF

cd "$tmpdir"
PROTO_HOME="$tmpdir/.proto" HOME="$tmpdir/user-home" proto install golang-migrate --config-mode local
PROTO_HOME="$tmpdir/.proto" HOME="$tmpdir/user-home" proto install psql --config-mode local
PROTO_HOME="$tmpdir/.proto" HOME="$tmpdir/user-home" proto install sentrux --config-mode local
PROTO_HOME="$tmpdir/.proto" HOME="$tmpdir/user-home" proto install skillshare --config-mode local
PROTO_HOME="$tmpdir/.proto" HOME="$tmpdir/user-home" proto install staticcheck --config-mode local

"$tmpdir/.proto/tools/golang-migrate/4.19.0/migrate" -version    # → 4.19.0
"$tmpdir/.proto/tools/psql/18.3.0/bin/psql" --version    # → psql (PostgreSQL) 18.3
SENTRUX_SKIP_GRAMMAR_DOWNLOAD=1 PROTO_HOME="$tmpdir/.proto" HOME="$tmpdir/user-home" proto run sentrux -- --version    # → sentrux 0.5.7
"$tmpdir/.proto/tools/skillshare/0.19.5/skillshare" --version    # → skillshare v0.19.5
"$tmpdir/.proto/tools/staticcheck/2026.2.2/staticcheck/staticcheck" -version    # → staticcheck 2026.2.2 (0.8.2)
```

Use explicit tool versions, never `latest` — validation must be reproducible. Staticcheck is pinned to a patch release (`2026.2.2`): the plugin resolves only three-part tags, because upstream tags its `.0` releases with two parts (`2026.2`) and proto has no token for the raw tag. Only include `sentrux` in local validation on platforms it publishes: Linux `x86_64`/`aarch64`, macOS `aarch64`, and Windows `x86_64`.


## CI Expectations

- Pushes to `main` should validate Linux, macOS, and Windows support.
- Keep runner labels aligned with GitHub-hosted runner availability.
- Test the checked-out plugin files with `file://./plugins/<plugin>.toml`, not the raw `main` URLs. This validates the commit under test.
- Each CI job should install Proto, generate a temporary `.prototools`, install every supported plugin, and verify the installed executable.
- If a new plugin is added, update every workflow so the new plugin is installed and checked.

## README Expectations

Update `README.md` whenever any consumer-facing behavior changes:

- Add or remove supported tools.
- Change tool IDs, executable names, raw GitHub URLs, platforms, architectures, or version examples.
- Change usage commands.
- Change CI coverage or validation expectations.

The README should remain suitable for both humans and agents consuming this public repository from other projects.

## Consuming From Other Repositories

When using these plugins in another repository, add the raw URLs to that repository's `.prototools`:

```toml
golang-migrate = "<version>"
psql = "<version>"
sentrux = "<version>"
skillshare = "<version>"
staticcheck = "<version>"

[plugins.tools]
golang-migrate = "https://raw.githubusercontent.com/trypanic/proto-plugins/main/plugins/golang-migrate.toml"
psql = "https://raw.githubusercontent.com/trypanic/proto-plugins/main/plugins/psql.toml"
sentrux = "https://raw.githubusercontent.com/trypanic/proto-plugins/main/plugins/sentrux.toml"
skillshare = "https://raw.githubusercontent.com/trypanic/proto-plugins/main/plugins/skillshare.toml"
staticcheck = "https://raw.githubusercontent.com/trypanic/proto-plugins/main/plugins/staticcheck.toml"
```

Prefer explicit versions known to exist upstream. Run `proto install --config-mode local` from the consuming repository to verify the configuration.

## Adding a New Plugin

1. Confirm the upstream project publishes versioned release archives or standalone binaries for each target OS and architecture.
2. Create `plugins/<tool-id>.toml`.
3. Use `type = "cli"` for simple CLI archive or standalone binary installs.
4. Add `[platform.*]` entries only for release assets that actually exist.
5. Add `[install]`, `[install.arch]` when needed, `[install.exes.*]`, and `[resolve]`.
6. Validate with an isolated Proto home.
7. Add the tool to CI workflows.
8. Update `README.md` with public usage and platform details.

## Sources

Prefer official sources when verifying behavior:

- Proto docs: `https://moonrepo.dev/docs/proto`
- Proto non-WASM plugins: `https://moonrepo.dev/docs/proto/non-wasm-plugin`
- GitHub-hosted runners: `https://docs.github.com/actions/reference/runners/github-hosted-runners`
- Upstream tool release pages linked from each plugin's `[resolve].git-url`
