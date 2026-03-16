# Snowfall Lib — Comprehensive Reference

> Unified configuration for systems, packages, modules, shells, templates, and more with Nix Flakes.
> Built on top of [flake-utils-plus](https://github.com/gytis-ivaskevicius/flake-utils-plus).

---

## Table of Contents

1. [Overview](#overview)
2. [Flake Inputs & Outputs](#flake-inputs--outputs)
3. [Getting Started](#getting-started)
4. [Library Bootstrap (`snowfall-lib/default.nix`)](#library-bootstrap)
5. [Library Modules](#library-modules)
   - [attrs](#attrs)
   - [checks](#checks)
   - [flake](#flake)
   - [fp (Functional Programming)](#fp-functional-programming)
   - [fs (Filesystem)](#fs-filesystem)
   - [home](#home)
   - [internal](#internal)
   - [module](#module)
   - [overlay](#overlay)
   - [package](#package)
   - [path](#path)
   - [shell](#shell)
   - [system](#system)
   - [template](#template)
6. [NixOS / Darwin / Home Modules](#nixos--darwin--home-modules)
   - [NixOS User Module](#nixos-user-module)
   - [Darwin User Module](#darwin-user-module)
   - [Home-Manager User Module](#home-manager-user-module)
7. [Directory Conventions](#directory-conventions)
8. [Special Arguments Injected by Snowfall Lib](#special-arguments-injected-by-snowfall-lib)
9. [Virtual Systems](#virtual-systems)
10. [Version History](#version-history)

---

## Overview

Snowfall Lib is a Nix Flakes library that provides a **convention-over-configuration** approach to managing:

- NixOS, nix-darwin, and virtual system configurations
- Home-Manager user configurations
- Reusable NixOS/darwin/home-manager modules
- Nixpkgs overlays
- Custom packages
- Development shells
- CI checks
- Flake templates
- User library extensions

It works by scanning well-known directory layouts inside your flake, automatically wiring everything into flake outputs while giving you full override capability.

### Design Principles

- **File-system driven**: directories under `systems/`, `homes/`, `packages/`, `modules/`, `overlays/`, `shells/`, `checks/`, and `templates/` are discovered automatically.
- **Composable library**: every sub-library is a plain attribute set merged into `lib.snowfall.*`.
- **Namespace isolation**: all user packages are placed under a configurable namespace attribute in nixpkgs (e.g. `pkgs.myorg`).
- **Functional style**: heavy use of `pipe`, point-free composition, and higher-order helpers.

---

## Flake Inputs & Outputs

### Inputs (`flake.nix`)

| Input | URL |
|---|---|
| `nixpkgs` | `github:nixos/nixpkgs/release-25.11` |
| `flake-utils-plus` | `github:gytis-ivaskevicius/flake-utils-plus/master` |

### Outputs

```
snowfall-lib
├── mkLib         # function: create the extended lib
├── mkFlake       # function: convenience wrapper (mkLib + lib.mkFlake)
├── nixosModules
│   └── user      # ./modules/nixos/user/default.nix
├── darwinModules
│   └── user      # ./modules/darwin/user/default.nix
├── homeModules
│   └── user      # ./modules/home/user/default.nix
├── formatter     # nixfmt-tree for x86_64-linux, aarch64-linux, x86_64-darwin, aarch64-darwin
└── snowfall      # internal config & lib exposed for documentation tooling
    ├── config    # { root, src, namespace, lib-dir, meta }
    ├── raw-config
    └── internal-lib
```

---

## Getting Started

### Creating the Library

```nix
# In your flake.nix
{
  inputs.snowfall-lib.url = "github:snowfallorg/lib";
  inputs.nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";

  outputs = inputs:
    inputs.snowfall-lib.lib.mkFlake {
      inherit inputs;
      src = ./.;

      snowfall = {
        namespace = "myorg";   # optional, default: "internal"
        meta = {
          name = "my-flake";
          title = "My Flake";
        };
      };
    };
}
```

### Using `mkLib` Directly

```nix
let
  lib = inputs.snowfall-lib.lib.mkLib {
    inherit inputs;
    src = ./.;
    snowfall.namespace = "myorg";
  };
in
lib.mkFlake { ... }
```

### `mkLib` Signature

```
mkLib : { inputs, src, snowfall? } -> lib
```

`mkLib` returns the merged nixpkgs lib extended with:
- all `lib` attributes from every input that exposes one (accessible as `lib.<input-name>.*`)
- the `snowfall.*` sub-library namespace
- the user's own `lib/` modules (merged under `lib.<namespace>.*`)

### `mkFlake` Signature

```
mkFlake : {
  inputs,
  src,
  snowfall?,
  systems?,
  homes?,
  modules?,
  overlays?,
  packages?,
  shells?,
  checks?,
  templates?,
  channels?,
  channels-config?,
  outputs-builder?,
  alias?,
  extra-exported-overlays?,
} -> <flake-outputs>
```

All options are optional; only `inputs` and `src` are required.

| Option | Type | Description |
|---|---|---|
| `inputs` | `Attrs` | Flake inputs (must include `self`) |
| `src` | `Path` | Root of the user's flake |
| `snowfall` | `Attrs` | Snowfall-specific config (namespace, meta, root) |
| `systems` | `Attrs` | Overrides/extra modules for system configs |
| `homes` | `Attrs` | Overrides/extra modules for home configs |
| `modules` | `Attrs` | Override maps for `nixos`, `darwin`, `home` module sets |
| `overlays` | `[Overlay]` | Extra overlays applied to channels |
| `packages` | `Attrs` | Additional packages merged over discovered ones |
| `shells` | `Attrs` | Additional dev shells merged over discovered ones |
| `checks` | `Attrs` | Additional checks merged over discovered ones |
| `templates` | `Attrs` | Additional templates merged over discovered ones |
| `channels` | `Attrs` | Per-channel nixpkgs configuration |
| `channels-config` | `Attrs` | Global nixpkgs config applied to all channels |
| `outputs-builder` | `channels -> Attrs` | Additional per-system outputs |
| `alias` | `Attrs` | Alias maps for packages, shells, checks, modules, templates |
| `extra-exported-overlays` | `Attrs` | Extra overlays to include in the `overlays` output |

---

## Library Bootstrap

**File**: `snowfall-lib/default.nix`

This is the entry point. It accepts `core-inputs` (Snowfall Lib's own inputs) and `user-options` (`{ inputs, src, snowfall? }`), then:

1. **Builds `snowfall-config`** from user options with defaults:
   - `namespace` defaults to `"internal"`
   - `root` defaults to `src`
   - `meta.name` / `meta.title` default to `null`

2. **Collects library inputs**: gathers the `.lib` attribute from every input that has one (both from core inputs and user inputs).

3. **Bootstraps the Snowfall sub-library** (`snowfall-lib`): uses `fix` to lazily load every subdirectory of `snowfall-lib/` as a Nix file, passing a shared `attrs` set containing `{ snowfall-lib, snowfall-config, core-inputs, user-inputs }`. All resulting attribute sets are deep-merged.

4. **Builds `base-lib`**: shallow-merges `nixpkgs.lib`, input libs, top-level (non-attrset) snowfall lib values, and `{ snowfall = snowfall-lib; }`.

5. **Loads the user's `lib/` modules**: discovers all `default.nix` files under `<src>/lib/`, calls each with `{ inputs, snowfall-inputs, namespace, lib }`, and deep-merges the results into `user-lib`.

6. **Returns the final `lib`**: `merge-deep [ base-lib user-lib ]`.

---

## Library Modules

Each module file exports an attribute set that is deep-merged into the top-level `snowfall-lib`. The modules are accessed as `lib.snowfall.<module>.<function>`.

---

### attrs

**File**: `snowfall-lib/attrs/default.nix`

Attribute set utilities.

#### `attrs.map-concat-attrs-to-list`

```
(a -> b -> [c]) -> Attrs -> [c]
```

Map over an attribute set and flatten the result into a single list.

```nix
attrs.map-concat-attrs-to-list (name: value: [name value]) { x = 1; y = 2; }
# => [ "x" 1 "y" 2 ]
```

#### `attrs.merge-deep`

```
[Attrs] -> Attrs
```

Recursively merge a list of attribute sets (later entries win at every level).

```nix
attrs.merge-deep [{ a.b = 1; } { a.c = 2; }]
# => { a = { b = 1; c = 2; }; }
```

#### `attrs.merge-shallow`

```
[Attrs] -> Attrs
```

Merge a list of attribute sets at the top level only (later entries win; sub-attrs are replaced, not merged).

```nix
attrs.merge-shallow [{ a = { x = 1; }; } { a = { y = 2; }; }]
# => { a = { y = 2; }; }
```

#### `attrs.merge-shallow-packages`

```
[Attrs] -> Attrs
```

Merge package attribute sets: derivations are kept as-is; nested attribute sets (e.g. `some.value`) are merged one level deep.

```nix
attrs.merge-shallow-packages [ { vim = <drv>; some.value = true; } { some.value = false; } ]
# => { vim = <drv>; some.value = false; }
```

#### `attrs.merge-with-aliases`

```
(Attrs -> Attrs -> Attrs) -> [Attrs] -> Attrs -> Attrs
```

Fold a list of items using a merge function, then apply aliases so that `alias.default = "vim"` adds `result.default = result.vim`.

#### `attrs.apply-aliases-and-overrides`

```
Attrs -> Attrs -> Attrs -> Attrs
```

Given a pre-built attribute set, apply aliases and then overlay overrides on top.

```nix
attrs.apply-aliases-and-overrides packages-set { default = "vim"; } { extra = ...; }
```

---

### checks

**File**: `snowfall-lib/checks/default.nix`

#### `check.create-checks`

```
{ channels, src?, pkgs?, overrides?, alias? } -> Attrs
```

Discover all `default.nix` files under `<snowfall-root>/checks/`, build them as derivations, apply aliases and overrides, and filter by the current system.

Each check's `default.nix` receives via `callPackageWith`:
- All `pkgs` attributes
- `channels` — all nixpkgs channel sets
- `lib` — the full Snowfall-extended lib
- `inputs` — user inputs (without `src`)
- `namespace` — the configured namespace string

```nix
# checks/my-check/default.nix
{ pkgs, lib, ... }:
pkgs.runCommand "my-check" {} "echo ok > $out"
```

---

### flake

**File**: `snowfall-lib/flake/default.nix`

Utilities for working with flake inputs, plus the central `mkFlake` function.

#### `flake.without-self`

```
Attrs -> Attrs
```

Remove the `self` key from an attribute set of flake inputs.

#### `flake.without-src`

```
Attrs -> Attrs
```

Remove the `src` key from an attribute set.

#### `flake.without-snowfall-inputs`

```
Attrs -> Attrs
```

Remove both `self` and `src` (composed from the two above).

#### `flake.without-snowfall-options`

```
Attrs -> Attrs
```

Remove all Snowfall-specific top-level `mkFlake` options so the remainder can be forwarded safely to `flake-utils-plus.lib.mkFlake`. Removed keys: `systems`, `modules`, `overlays`, `packages`, `outputs-builder`, `outputsBuilder`, `packagesPrefix`, `hosts`, `homes`, `channels-config`, `templates`, `checks`, `alias`, `snowfall`.

#### `flake.get-libs`

```
Attrs -> Attrs
```

Transform an attribute set of inputs into an attribute set of their `.lib` attributes; entries without a `lib` attrset are dropped.

```nix
flake.get-libs { nixpkgs = nixpkgs; empty = {}; }
# => { nixpkgs = nixpkgs.lib; }
```

#### `mkFlake` (the main entry point)

See [Getting Started](#getting-started) for the full signature. Internally it:

1. Creates system configurations via `system.create-systems`
2. Creates home-manager configurations via `home.create-homes`
3. Creates NixOS, Darwin, and Home modules via `module.create-modules`
4. Creates overlays via `overlay.create-overlays`
5. Creates packages, shells, and checks inside `outputs-builder`
6. Merges everything and delegates to `flake-utils-plus.lib.mkFlake`
7. Post-processes `packages` output to include home-manager activation packages (as `homeConfigurations-<user>`)

---

### fp (Functional Programming)

**File**: `snowfall-lib/fp/default.nix`

Small functional programming helpers.

#### `fp.compose`

```
(b -> c) -> (a -> b) -> a -> c
```

Compose two functions right-to-left.

```nix
fp.compose (x: x + 2) (x: x + 1)  # equivalent to x: (x+1)+2
```

#### `fp.compose-all`

```
[(x -> y)] -> a -> b
```

Compose a list of functions right-to-left.

```nix
fp.compose-all [ add-two add-one ]  # equivalent to fp.compose add-two add-one
```

#### `fp.call`

```
(a -> b) -> a -> b
```

Apply a function to an argument (identity combinator, useful for point-free pipelines).

#### `fp.apply`

```
a -> (a -> b) -> b
```

Flip of `call` — apply an argument to a function.

---

### fs (Filesystem)

**File**: `snowfall-lib/fs/default.nix`

All filesystem helpers are safe: operations on non-existent paths return `{}` or `[]`.

#### Kind predicates

```
fs.is-file-kind      : String -> Bool   -- kind == "regular"
fs.is-symlink-kind   : String -> Bool   -- kind == "symlink"
fs.is-directory-kind : String -> Bool   -- kind == "directory"
fs.is-unknown-kind   : String -> Bool   -- kind == "unknown"
```

Used with `builtins.readDir` results.

#### `fs.get-file`

```
String -> String
```

Resolve a path relative to the **user's flake source** (`user-inputs.src`).

```nix
fs.get-file "systems"
# => "/path/to/user-flake/systems"
```

#### `fs.get-snowfall-file`

```
String -> String
```

Resolve a path relative to `snowfall-config.root` (the root of the Snowfall-configured portion of the flake, which may differ from `src`).

```nix
fs.get-snowfall-file "modules/nixos"
# => "/path/to/user-flake/modules/nixos"
```

#### `fs.internal-get-file`

```
String -> String
```

Resolve a path relative to the Snowfall Lib flake itself (internal use).

#### `fs.safe-read-directory`

```
Path -> Attrs
```

`builtins.readDir` that returns `{}` if the path does not exist.

#### `fs.get-entries-by-kind`

```
(String -> Bool) -> Path -> [Path]
```

Return all entries in a directory whose kind satisfies the predicate. Returns full paths.

#### `fs.get-directories`

```
Path -> [Path]
```

Return all subdirectories of a path.

#### `fs.get-directories-with-default`

```
Path -> [Path]
```

Return all subdirectories that contain a `default.nix` file.

```nix
fs.get-directories-with-default ./systems/x86_64-linux
# => [ "./systems/x86_64-linux/my-host" ]
```

#### `fs.get-files`

```
Path -> [Path]
```

Return all regular files (non-recursive) in a directory.

#### `fs.get-files-recursive`

```
Path -> [Path]
```

Return all regular files recursively under a directory.

#### `fs.filter-files`

```
(Path -> Bool) -> Path -> [Path]
```

Filter the non-recursive file listing by a predicate.

#### `fs.filter-files-recursive`

```
(Path -> Bool) -> Path -> [Path]
```

Filter the recursive file listing by a predicate.

#### `fs.get-nix-files`

```
Path -> [Path]
```

All `*.nix` files (non-recursive) in a directory.

#### `fs.get-nix-files-recursive`

```
Path -> [Path]
```

All `*.nix` files recursively under a directory.

#### `fs.get-default-nix-files`

```
Path -> [Path]
```

All files named exactly `default.nix` (non-recursive).

#### `fs.get-default-nix-files-recursive`

```
Path -> [Path]
```

All `default.nix` files recursively. This is the primary function used to discover packages, modules, overlays, etc.

#### `fs.get-non-default-nix-files`

```
Path -> [Path]
```

All `*.nix` files whose name is **not** `default.nix` (non-recursive).

#### `fs.get-non-default-nix-files-recursive`

```
Path -> [Path]
```

All `*.nix` files recursively whose name is **not** `default.nix`.

---

### home

**File**: `snowfall-lib/home/default.nix`

Home-Manager configuration management.

> **Requires** the `home-manager` flake input.

#### `home.split-user-and-host`

```
String -> { user: String, host: String }
```

Parse a `"user@host"` string. If no `@` is present, `host` is `""`.

```nix
home.split-user-and-host "alice@my-machine"
# => { user = "alice"; host = "my-machine"; }
```

#### `home.create-home`

```
{
  path,
  name?,
  modules?,
  specialArgs?,
  channelName?,
  system?,
} -> Attrs
```

Create a single home-manager configuration. `name` defaults to the parent directory name of `path`. If the name contains no `@`, the system is appended automatically: `"alice@x86_64-linux"`.

The configuration is built using `home-manager.lib.homeManagerConfiguration` and automatically includes:
- The Snowfall home user module (`modules/home/user/default.nix`)
- A nix-registry patch module that disables the flake-utils-plus options module (incompatible with standalone home-manager)
- Default `snowfallorg.user.enable = true` and `snowfallorg.user.name`

Special args injected into modules:

| Arg | Value |
|---|---|
| `system` | The target system string |
| `name` | The unique `user@system` name |
| `user` | Parsed username |
| `host` | Parsed hostname (may be `""`) |
| `format` | `"home"` |
| `inputs` | User flake inputs (without `src`) |
| `namespace` | Snowfall namespace string |
| `pkgs` | The nixpkgs package set for the system |
| `lib` | The Snowfall-extended lib with `hm` attribute |

#### `home.get-target-homes-metadata`

```
Path -> [{ system: String, name: String, path: Path }]
```

Scan a directory (e.g. `homes/x86_64-linux`) for subdirectories that contain `default.nix` and return metadata for each.

#### `home.create-homes`

```
{
  users?,
  modules?,
} -> Attrs
```

Discover and create all home-manager configurations. Scans `<snowfall-root>/homes/<system>/<user>/default.nix`. Applies per-user overrides from `homes.users.<name>` and shared modules from `homes.modules`.

Auto-discovered home modules from `modules/home/` are injected into every home.

#### `home.create-home-system-modules`

```
Attrs -> [Module]
```

When home-manager is used **inside** a NixOS or Darwin system (not standalone), this produces the list of NixOS/Darwin modules that integrate each home into the system. It handles:

- Passing `extraSpecialArgs` to home-manager
- Injecting `home-manager.sharedModules`
- Per-user `home-manager.users.<name>` configuration scoped to the correct host
- `home-manager.useGlobalPkgs = true` by default

---

### internal

**File**: `snowfall-lib/internal/default.nix`

Internal helpers not intended for direct user consumption.

#### `internal.system-lib`

The complete library as seen by system and module evaluation contexts. Shallow-merges `nixpkgs.lib`, all input libs, and `{ <namespace> = user-lib; }`.

#### `internal.user-lib`

The user's own library loaded from `<snowfall-root>/lib/`. Deep-merges all discovered `default.nix` files. Each file can be a function accepting `{ inputs, snowfall-inputs, namespace, lib }` or a plain attribute set.

#### `internal.create-simple-derivations`

```
{
  type,
  channels,
  src?,
  pkgs?,
  overrides?,
  alias?,
} -> Attrs
```

Generic helper used by both `shell.create-shells` and `check.create-checks`. Discovers `default.nix` files under `src` (defaults to `<snowfall-root>/<type>`), builds them with `callPackageWith`, and filters by the current system. Each derivation receives `{ channels, lib, inputs, namespace }`.

---

### module

**File**: `snowfall-lib/module/default.nix`

#### `module.create-modules`

```
{
  src?,
  overrides?,
  alias?,
} -> Attrs
```

Discover all `default.nix` files recursively under `src` and expose them as a set keyed by their relative path (e.g. `"networking/firewall"`).

`src` defaults to `<snowfall-root>/modules/nixos`.

Each module wrapper:
1. Derives `system`, `target`, `format`, and `virtual` from module arguments.
2. Injects `lib`, `inputs`, `namespace` into module arguments.
3. Calls the module with the enriched args.
4. Attaches `_file` for error attribution.

Modules are keyed by their **relative path** from `src`, without the trailing `/default.nix`, e.g. `"hardware/nvidia"`.

```nix
# modules/nixos/hardware/nvidia/default.nix
{ lib, pkgs, config, namespace, ... }:
{
  options.${namespace}.hardware.nvidia.enable = lib.mkEnableOption "NVIDIA support";
  config = lib.mkIf config.${namespace}.hardware.nvidia.enable { ... };
}
```

---

### overlay

**File**: `snowfall-lib/overlay/default.nix`

#### `overlay.create-overlays-builder`

```
{
  src?,
  namespace?,
  extra-overlays?,
} -> channels -> [Overlay]
```

Produce the `overlaysBuilder` function expected by `flake-utils-plus`. Returns a list of overlays that:
1. Expose all user packages under `pkgs.<namespace>`.
2. Apply any overlays discovered under `src` (defaults to `<snowfall-root>/overlays`).
3. Append any extra overlays.

Each discovered overlay `default.nix` receives `user-inputs // { channels, namespace, inputs, lib }`.

#### `overlay.create-overlays`

```
{
  src?,
  packages-src?,
  namespace?,
  extra-overlays?,
} -> Attrs
```

Create the exported `overlays` flake output. Returns a set containing:

- `"package/<name>"` — a per-package overlay for each discovered package.
- `"<name>"` — each discovered overlay from `overlays/`.
- `"default"` — a combined overlay that applies all packages and overlays.
- Any extra overlays passed in.

Overlays that set `__dontExport = true` are excluded from the exported set.

---

### package

**File**: `snowfall-lib/package/default.nix`

#### `package.create-packages`

```
{
  channels,
  src?,
  pkgs?,
  overrides?,
  alias?,
  namespace?,
} -> Attrs
```

Discover all `default.nix` files recursively under `src` (defaults to `<snowfall-root>/packages`), build each with `callPackageWith`, apply aliases and overrides, and filter by the current system.

Each package's `default.nix` receives via `callPackageWith`:
- All standard `pkgs` attributes
- `pkgs.<namespace>` pointing to the currently-building package set (enables packages to depend on sibling packages)
- `channels` — all nixpkgs channel sets
- `lib` — the full Snowfall-extended lib
- `inputs` — user inputs
- `namespace` — the namespace string

Packages are keyed by the **parent directory name** of their `default.nix`, e.g. `packages/my-tool/default.nix` → `"my-tool"`.

A `meta.snowfall.path` attribute is attached to every built derivation for traceability.

---

### path

**File**: `snowfall-lib/path/default.nix`

File path manipulation utilities.

#### `path.split-file-extension`

```
String -> [String]
```

Split `"my-file.md"` into `[ "my-file" "md" ]`. Asserts that the file has an extension.

#### `path.has-any-file-extension`

```
String -> Bool
```

Return `true` if the string/path has any file extension.

#### `path.get-file-extension`

```
String -> String
```

Return the extension of a file name (e.g. `"txt"` from `"my-file.final.txt"`). Returns `""` if none.

#### `path.has-file-extension`

```
String -> String -> Bool
```

`has-file-extension "nix" "foo.nix"` → `true`. Curried for use with `filter`.

#### `path.get-parent-directory`

```
Path -> String
```

Return the **name** (not full path) of the parent directory.

```nix
path.get-parent-directory "/a/b/c/default.nix"
# => "c"
```

Composed from `baseNameOf ∘ dirOf`.

#### `path.get-file-name-without-extension`

```
Path -> String
```

Return the file name without its extension.

```nix
path.get-file-name-without-extension ./some-dir/my-file.pdf
# => "my-file"
```

#### `path.get-output-name`

```
Path -> String
```

Convenience for package/shell/check output naming. Discards the Nix path string context (required when using a path as an attrset key) and returns the parent directory name.

```nix
path.get-output-name ./foo/bar/default.nix
# => "bar"
```

Composed from `unsafeDiscardStringContext ∘ get-parent-directory`.

#### `path.get-directory-name`

```
Path -> String
```

Return the base name of a directory path, discarding string context.

```nix
path.get-directory-name /templates/foo
# => "foo"
```

#### `path.get-relative-module-path`

```
String -> Path -> String
```

Strip the `src` prefix and the trailing `/default.nix` from a module path, yielding the relative module key used in `module.create-modules`.

```nix
path.get-relative-module-path "/modules/nixos" "/modules/nixos/foo/bar/default.nix"
# => "foo/bar"
```

---

### shell

**File**: `snowfall-lib/shell/default.nix`

#### `shell.create-shells`

```
{ channels, src?, pkgs?, overrides?, alias? } -> Attrs
```

Discover all `default.nix` files under `<snowfall-root>/shells/`, build them as derivations, apply aliases and overrides, and filter by the current system.

Each shell's `default.nix` receives the same arguments as checks (see [checks](#checks)).

```nix
# shells/dev/default.nix
{ pkgs, lib, ... }:
pkgs.mkShell {
  packages = with pkgs; [ git curl ];
}
```

---

### system

**File**: `snowfall-lib/system/default.nix`

System configuration management for NixOS, nix-darwin, and virtual (nixos-generators) targets.

#### `system.get-inferred-system-name`

```
Path -> String
```

Return the system name from a file path. For `*.nix` files returns the parent directory name; for directories returns `baseNameOf`.

#### `system.is-darwin`

```
String -> Bool
```

True if the system target string contains `"darwin"` (e.g. `"aarch64-darwin"`).

#### `system.is-linux`

```
String -> Bool
```

True if the system target string contains `"linux"`.

#### `system.is-virtual`

```
String -> Bool
```

True if the target corresponds to a nixos-generators virtual format (e.g. `"x86_64-iso"`).

#### `system.get-virtual-system-type`

```
String -> String
```

Return the virtual system format string embedded in the target, or `""` if not virtual. See [Virtual Systems](#virtual-systems) for the full list.

```nix
system.get-virtual-system-type "x86_64-iso"
# => "iso"

system.get-virtual-system-type "x86_64-linux"
# => ""
```

#### `system.get-target-systems-metadata`

```
Path -> [{ target: String, name: String, path: Path }]
```

For a given target directory (e.g. `systems/x86_64-linux`), enumerate subdirectories containing `default.nix` and return metadata for each.

#### `system.get-system-builder`

```
String -> (Attrs -> Attrs)
```

Return the appropriate system builder function for a target:

| Target | Builder |
|---|---|
| Virtual (any format) | `nixos-generators.nixosGenerate` with `format` set |
| `*-darwin` | `darwin.lib.darwinSystem` |
| `*-linux` | `nixpkgs.lib.nixosSystem` |

All builders automatically include the corresponding Snowfall user module.

> **Note**: Virtual systems require the `nixos-generators` input. Darwin systems require the `darwin` input.

#### `system.get-system-output`

```
String -> String
```

Return the flake output attribute name for configurations:

| Target | Output |
|---|---|
| Virtual | `"<format>Configurations"` |
| Darwin | `"darwinConfigurations"` |
| Linux | `"nixosConfigurations"` |

#### `system.get-resolved-system-target`

```
String -> String
```

For virtual targets, replace the format with `"linux"` to get the real CPU architecture target. For real targets, returns unchanged.

```nix
system.get-resolved-system-target "x86_64-iso"
# => "x86_64-linux"
```

#### `system.create-system`

```
{
  target?,
  system?,
  path,
  name?,
  modules?,
  specialArgs?,
  channelName?,
  builder?,
  output?,
  systems?,
  homes?,
} -> Attrs
```

Create a single system configuration. All optional fields default from the target string.

Special args injected into system modules:

| Arg | Value |
|---|---|
| `target` | The original target string (e.g. `"x86_64-iso"`) |
| `system` | The resolved system (e.g. `"x86_64-linux"`) |
| `format` | `"linux"`, `"darwin"`, or the virtual format string |
| `virtual` | `Bool` |
| `host` | The system name |
| `systems` | Attribute set of all created systems (for cross-system references) |
| `lib` | The Snowfall-extended lib |
| `inputs` | User inputs (without `src`) |
| `namespace` | Snowfall namespace |

If the `home-manager` input exists, home-manager modules are automatically added.

#### `system.create-systems`

```
{
  systems?,
  homes?,
} -> Attrs
```

Discover and create all system configurations. Scans `<snowfall-root>/systems/<target>/<name>/default.nix`.

- Applies per-host overrides from `systems.hosts.<name>`
- Applies shared NixOS modules from `systems.modules.nixos`
- Applies shared Darwin modules from `systems.modules.darwin`
- Auto-discovers and injects user modules from `modules/nixos/` or `modules/darwin/`
- All created systems are passed back as the `systems` special arg (enables cross-host references)

---

### template

**File**: `snowfall-lib/template/default.nix`

#### `template.create-templates`

```
{
  src?,
  overrides?,
  alias?,
} -> Attrs
```

Discover all **directories** under `<snowfall-root>/templates/` and expose them as flake templates. If the template directory contains a `flake.nix`, its `description` field is automatically extracted and attached to the template metadata.

```nix
# templates/my-template/flake.nix
{
  description = "A minimal NixOS configuration template";
  # ...
}
```

---

## NixOS / Darwin / Home Modules

These modules are part of the flake's `nixosModules`, `darwinModules`, and `homeModules` outputs, and are also automatically injected by Snowfall Lib into every created system/home.

---

### NixOS User Module

**File**: `modules/nixos/user/default.nix`
**Output**: `nixosModules.user`

Provides `snowfallorg.users.<name>` options for declarative NixOS user management.

> **Migration note**: `snowfallorg.user` (singular) is a renamed alias for `snowfallorg.users` (plural).

#### Options

```
snowfallorg.users.<name>
├── create     Bool    (default: true)  — whether to create the OS user
├── admin      Bool    (default: true)  — whether to add the user to the wheel group
└── home
    ├── enable Bool    (default: true)
    ├── path   String  (default: "/home/<name>")
    └── config Submodule — home-manager-compatible configuration
```

When `create = true`, a corresponding `users.users.<name>` entry is created with:
- `isNormalUser = true`
- `name = <name>`
- `home = <path>`
- `group = "users"`
- `extraGroups = [ "wheel" ]` if `admin = true`

The `home.config` submodule accepts full home-manager configuration if `home-manager` is available as a flake input.

---

### Darwin User Module

**File**: `modules/darwin/user/default.nix`
**Output**: `darwinModules.user`

Same concept as the NixOS module but for nix-darwin. The `admin` option is absent (macOS user management works differently).

#### Options

```
snowfallorg.users.<name>
├── create  Bool    (default: true)
└── home
    ├── enable Bool   (default: true)
    ├── path   String (default: "/Users/<name>")
    └── config Submodule
```

When `create = true`, creates a `users.users.<name>` entry with `home` and `isHidden = false`.

---

### Home-Manager User Module

**File**: `modules/home/user/default.nix`
**Output**: `homeModules.user`

Provides `snowfallorg.user` options for a home-manager configuration.

#### Options

```
snowfallorg.user
├── enable         Bool   (default: false) — enable automatic home configuration
├── name           String — the username
└── home.directory String — the home directory path
```

When `enable = true`:
- `home.username` is set to `snowfallorg.user.name`
- `home.homeDirectory` is set to `snowfallorg.user.home.directory`

The default `home.directory` is determined in order:
1. `osConfig.users.users.<name>.home` if available (NixOS/darwin integration)
2. `/Users/<name>` on Darwin
3. `/home/<name>` on Linux

---

## Directory Conventions

Snowfall Lib auto-discovers files in the following directories relative to `snowfall-config.root` (defaults to your flake's `src`):

```
<root>/
├── systems/
│   └── <target>/          # e.g. x86_64-linux, aarch64-darwin, x86_64-iso
│       └── <hostname>/
│           └── default.nix
│
├── homes/
│   └── <target>/          # e.g. x86_64-linux
│       └── <user>/        # or <user>@<host> for host-scoped homes
│           └── default.nix
│
├── modules/
│   ├── nixos/
│   │   └── <path>/
│   │       └── default.nix
│   ├── darwin/
│   │   └── <path>/
│   │       └── default.nix
│   └── home/
│       └── <path>/
│           └── default.nix
│
├── packages/
│   └── <name>/
│       └── default.nix
│
├── overlays/
│   └── <name>/
│       └── default.nix
│
├── shells/
│   └── <name>/
│       └── default.nix
│
├── checks/
│   └── <name>/
│       └── default.nix
│
├── templates/
│   └── <name>/
│       ├── flake.nix      # optional — description is extracted automatically
│       └── ...
│
└── lib/
    └── <any-name>/
        └── default.nix    # merged into lib.<namespace>.*
```

All directories are optional; Snowfall Lib handles non-existent paths gracefully.

---

## Special Arguments Injected by Snowfall Lib

### System modules (`modules/nixos/`, `modules/darwin/`)

| Arg | Type | Description |
|---|---|---|
| `system` | `String` | Resolved system (e.g. `"x86_64-linux"`) |
| `target` | `String` | Original target (e.g. `"x86_64-iso"`) |
| `format` | `String` | `"linux"`, `"darwin"`, or the virtual format |
| `virtual` | `Bool` | Whether this is a virtual system |
| `systems` | `Attrs` | All created system configs (for cross-system references) |
| `lib` | `Attrs` | Snowfall-extended lib |
| `inputs` | `Attrs` | User flake inputs (without `src`) |
| `namespace` | `String` | Snowfall namespace string |

### System configurations (`systems/<target>/<name>/default.nix`)

Same as system modules plus:

| Arg | Type | Description |
|---|---|---|
| `host` | `String` | The system's hostname |

### Home configurations (`homes/<target>/<user>/default.nix`)

| Arg | Type | Description |
|---|---|---|
| `system` | `String` | The home's target system |
| `name` | `String` | Unique home name (`user@system`) |
| `user` | `String` | The username |
| `host` | `String` | The bound hostname (may be `""`) |
| `format` | `String` | Always `"home"` |
| `inputs` | `Attrs` | User flake inputs (without `src`) |
| `namespace` | `String` | Snowfall namespace |
| `pkgs` | `Attrs` | nixpkgs package set for this system |
| `lib` | `Attrs` | Snowfall-extended lib with `hm` attribute |

### Packages (`packages/<name>/default.nix`)

Arguments are supplied via `callPackageWith` so any subset can be requested:

| Arg | Type | Description |
|---|---|---|
| `pkgs.*` | Any | All standard nixpkgs attributes |
| `pkgs.<namespace>` | `Attrs` | All currently-building user packages |
| `channels` | `Attrs` | All nixpkgs channel sets |
| `lib` | `Attrs` | Snowfall-extended lib |
| `inputs` | `Attrs` | User flake inputs |
| `namespace` | `String` | Snowfall namespace |

### Shells and Checks

Same as packages.

### Overlays (`overlays/<name>/default.nix`)

Overlays are imported as functions and called with:

```nix
user-inputs // {
  channels,
  namespace,
  inputs,  # same as user-inputs
  lib,
}
```

The overlay function itself should return the standard `final: prev: { ... }` overlay.

---

## Virtual Systems

Virtual systems use [nixos-generators](https://github.com/nix-community/nixos-generators) to produce machine images. They are enabled by using a virtual format name **instead of** `linux` in the target string.

The supported formats (in priority order, most-specific first):

| Format | Target example |
|---|---|
| `amazon` | `x86_64-amazon` |
| `azure` | `x86_64-azure` |
| `cloudstack` | `x86_64-cloudstack` |
| `docker` | `x86_64-docker` |
| `do` | `x86_64-do` |
| `gce` | `x86_64-gce` |
| `install-iso-hyperv` | `x86_64-install-iso-hyperv` |
| `hyperv` | `x86_64-hyperv` |
| `install-iso` | `x86_64-install-iso` |
| `iso` | `x86_64-iso` |
| `kexec` | `x86_64-kexec` |
| `kexec-bundle` | `x86_64-kexec-bundle` |
| `kubevirt` | `x86_64-kubevirt` |
| `proxmox-lxc` | `x86_64-proxmox-lxc` |
| `lxc-metadata` | `x86_64-lxc-metadata` |
| `lxc` | `x86_64-lxc` |
| `openstack` | `x86_64-openstack` |
| `proxmox` | `x86_64-proxmox` |
| `qcow` | `x86_64-qcow` |
| `raw-efi` | `x86_64-raw-efi` |
| `raw` | `x86_64-raw` |
| `sd-aarch64-installer` | `aarch64-sd-aarch64-installer` |
| `sd-aarch64` | `aarch64-sd-aarch64` |
| `vagrant-virtualbox` | `x86_64-vagrant-virtualbox` |
| `virtualbox` | `x86_64-virtualbox` |
| `vm-bootloader` | `x86_64-vm-bootloader` |
| `vm-nogui` | `x86_64-vm-nogui` |
| `vmware` | `x86_64-vmware` |
| `vm` | `x86_64-vm` |

> **Important**: The order matters. `"vm-bootloader"` must appear before `"vm"` so that a target containing `"vm-bootloader"` does not falsely match `"vm"`.

Virtual systems output their configurations under `<format>Configurations` (e.g. `isoConfigurations`). The `format` special arg is set to the virtual format string so modules can conditionally apply settings.

---

## Version History

| Tag | Notable Changes |
|---|---|
| `v3.0.3` | Update nixpkgs to 25.11; `fold` → `foldr` correctness fix |
| `v3.0.2` | Fix: flatten home configs into packages for flake schema compliance; expose activation packages |
| `v3.0.1` | Refactor: eliminate DRY violations; add `fs.get-directories-with-default`, `path.get-relative-module-path`, `path.get-directory-name`, `path.get-output-name`; switch `fold` to `foldr` |
| `v3.0.0` | Refactor: point-free style, pipe-based data transformations, improved attrs helpers, `create-simple-derivations` shared helper |
| `v2.1.x` | Per-channel configuration; same-username-across-targets support; namespace injection in overlays and home-manager |
| `v2.0.0` | Auto-extract template descriptions; replace `./` path roots with `self` for evaluation speed |
| `v1.0.x` | Initial release |
