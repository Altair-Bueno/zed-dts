# zed-dts

A Zed editor extension adding Device Tree Source (DTS) language support.

## What this is

- Rust `cdylib` compiled to `wasm32-wasip1`, implementing Zed's `Extension` trait
- Syntax highlighting via `joelspadin/tree-sitter-devicetree` (tree-sitter grammar)
- LSP via `devicetree-language-server` (npm package, installed automatically by the extension)
- Supports `.dts`, `.dtsi`, `.overlay` — both Zephyr RTOS and Linux kernel workflows

## Key files

| File                                           | Purpose                                                                            |
| ---------------------------------------------- | ---------------------------------------------------------------------------------- |
| `extension.toml`                               | Extension metadata, grammar pin, LSP registration                                  |
| `Cargo.toml`                                   | Rust manifest — `cdylib`, `zed_extension_api = "0.7.0"`                            |
| `src/dts.rs`                                   | Extension logic: npm install fallback + LSP command + workspace config passthrough |
| `languages/dts/config.toml`                    | Language config: extensions, comments, brackets                                    |
| `languages/dts/highlights.scm`                 | Syntax highlighting queries — copied from submodule, do not edit directly          |
| `tree-sitter-devicetree/`                      | Git submodule at pinned commit                                                     |
| `languages/dts/LICENSE-tree-sitter-devicetree` | MIT license for the vendored grammar                                               |
| `examples/`                                    | Sample DTS files used to test the extension locally                                |
| `.zed/settings.json`                           | LSP context configuration for the examples                                         |

## Build

```bash
cargo build --target wasm32-wasip1 --release
```

The `wasm32-wasip1` target must be installed (`rustup target add wasm32-wasip1`).

## Testing locally

Load the extension as a dev extension in Zed:

1. Open the command palette → `zed: install dev extension`
2. Select this repository directory
3. Open `examples/zedboard-v1.dts` — the status bar should show **DTS** and the LSP should start

The `.zed/settings.json` at the repo root configures the LSP context for the examples, so
diagnostics, completion, and hover work out of the box on those files.

## LSP settings

All settings are namespaced under `"devicetree"` in `.zed/settings.json`. Flat keys (no
namespace) are silently ignored by the LSP — this is a common gotcha.

```json
{
  "lsp": {
    "dts-language-server": {
      "settings": {
        "devicetree": { "..." }
      }
    }
  }
}
```

Files are only analysed if they belong to an active context. There are two ways to set one up:

- **Ad-hoc** (`"allowAdhocContexts": true`): the LSP auto-creates a context for every file
  you open. Good for quick exploration, no path configuration needed.
- **Explicit context**: define a `contexts` array with `dtsFile` + `includePaths`. The root
  `.dts` anchors the context; all `.dtsi`/`.overlay` files reachable via `#include` are
  covered automatically. Required for diagnostics and cross-file navigation to work correctly.

The `.zed/settings.json` in this repo uses an explicit context pointing at the example files.

## Reference implementations

Two existing extensions were used as implementation references:

**[zed_bitbake](https://github.com/anikinmd/zed_bitbake)** — primary reference for `src/dts.rs`.
Bitbake also uses an npm-based LSP (`language-server-bitbake`), so its three-tier binary
resolution pattern (system binary → cached npm script → fresh npm install) was adapted
directly. The `language_server_workspace_configuration` passthrough pattern also comes from
here.

**[zed_neocmake](https://github.com/k0tran/zed_neocmake)** — reference for overall extension
structure and `extension.toml` layout. neocmakelsp is a GitHub-release binary (not npm), so
its `language_server_command` was not used, but the project structure, `Cargo.toml`, and
`extension.toml` schema were used as a baseline.

## Reference documentation

- [Zed extension API](https://zed.dev/docs/extensions/languages)
- [zed_extension_api crate docs](https://docs.rs/zed_extension_api/latest/zed_extension_api/)
- [devicetree-language-server (dts-lsp)](https://github.com/kylebonnici/dts-lsp) — LSP settings reference
- [joelspadin/tree-sitter-devicetree](https://github.com/joelspadin/tree-sitter-devicetree) — grammar source

## Updating the grammar

The grammar C parser is fetched by Zed at the commit pinned in `extension.toml`. The query
files (`highlights.scm`) are vendored in `languages/dts/` and must be kept in sync manually.

1. Update the submodule:
   ```bash
   git submodule update --remote tree-sitter-devicetree
   ```
2. Copy the updated queries:
   ```bash
   cp tree-sitter-devicetree/queries/highlights.scm languages/dts/highlights.scm
   ```
3. Copy the license if it changed:
   ```bash
   cp tree-sitter-devicetree/LICENSE languages/dts/LICENSE-tree-sitter-devicetree
   ```
4. Update `commit` in `extension.toml` to the new submodule HEAD:
   ```bash
   git -C tree-sitter-devicetree rev-parse HEAD
   # paste that SHA into extension.toml under [grammars.devicetree]
   ```
5. Commit everything:
   ```bash
   git add extension.toml languages/dts/highlights.scm languages/dts/LICENSE-tree-sitter-devicetree tree-sitter-devicetree
   git commit -m "chore: update tree-sitter-devicetree grammar"
   ```
