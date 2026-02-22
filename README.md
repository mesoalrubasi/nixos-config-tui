# nixos-config-tui

**WARNING: This is an experimental proof-of-concept. The patched Nix evaluator
may have subtle bugs — do NOT use this in production. All internal NixOS tests
pass, but hidden invariants may be broken. (tho it does work, just no guarantees!)**

A TUI which allows you to browse values and dependencies of NixOS options that influenced a NixOS configuration.
This can be used to diff two configurations at the configuration/option-level as opposed to diffing the resulting derivation.

For a detailed explanation and showcase, see the [blog post](https://oddlama.org/blog/tracking-options-in-nixos/).

## Screenshots


<table>
    <tr>
        <td>
            <b>Dependency graph</b>
            <img width="3264" height="1626" alt="image" src="https://github.com/user-attachments/assets/f0df5794-0794-49af-9975-247f9c56e5e2" />
        </td>
        <td>
            <b>Exploring a configuration</b>
            <img width="3829" height="2113" alt="image" src="https://github.com/user-attachments/assets/61d34a5c-4ee0-4951-a517-a745941d296b" />
        </td>
    </tr>
    <tr>
        <td>
            <b>Diffing two configurations</b>
            <img width="3829" height="2113" alt="image" src="https://github.com/user-attachments/assets/dbc77951-5a3a-4be6-8256-47532eca3990" />
        </td>
        <td>
            <b>Textual diff</b>
            <img width="3829" height="2113" alt="image" src="https://github.com/user-attachments/assets/e63fc457-4729-4ea2-9887-5f168510d2a6" />
        </td>
    </tr>
</table>

## Quick start

You do _not_ need to change your system's Nix daemon. Since all changes are in
expression evaluation, it suffices to run the patched Nix CLI.

1. Enter a shell with the patched nix binary and the `nixos-config` utility:

   ```bash
   nix shell github:oddlama/nix/thunk-origins-v1 github:oddlama/nixpkgs/thunk-origins-v1#nixos-config
   ```

2. Define a host using the patched nixpkgs and set `trackDependencies = true`:

   ```nix
   # flake.nix
   {
     inputs.nixpkgs.url = "github:oddlama/nixpkgs/thunk-origins-v1";
     outputs = { self, nixpkgs }: {
       nixosConfigurations.host1 = nixpkgs.lib.nixosSystem {
         system = "x86_64-linux";
         trackDependencies = true;
         modules = [{
           boot.loader.grub.device = "nodev";
           fileSystems."/" = {
             device = "/dev/sda1";
             fsType = "ext4";
           };
           system.stateVersion = "25.11";
         }];
       };
     };
   }
   ```

   <details>
   <summary>Without flakes (click to expand)</summary>

   Clone `https://github.com/oddlama/nixpkgs`, check out `thunk-origins-v1`, then:

   ```nix
   # toplevel.nix
   let
     nixpkgs = import ./nixpkgs {};
     lib = nixpkgs.lib;
   in import ./nixpkgs/nixos/lib/eval-config.nix {
     inherit lib;
     trackDependencies = true;
     modules = [{
       boot.loader.grub.device = "nodev";
       fileSystems."/" = {
         device = "/dev/sda1";
         fsType = "ext4";
       };
       system.stateVersion = "25.11";
     }];
   }
   ```

   </details>

3. Build the tracked configuration (expect ~10-20s extra evaluation time):

   ```bash
   nix build --print-out-paths .#nixosConfigurations.host1.config.system.build.toplevel
   ```

   The resulting toplevel will contain `tracking.json`, `tracking-explicit.json`,
   and `tracking-deps.json` alongside the usual system files.

4. Explore, show or diff:

   ```bash
   # Explore a built toplevel
   nixos-config show /nix/store/...-nixos-toplevel-tracked

   # Explore from flake reference (no build needed)
   nixos-config show .#host1

   # Diff two toplevels
   nixos-config diff /nix/store/OLD /nix/store/NEW

   # Diff showing only explicitly defined values
   nixos-config diff --explicit /nix/store/OLD /nix/store/NEW

   # Textual diff as pseudo configuration.nix
   nixos-config text-diff --explicit /nix/store/OLD /nix/store/NEW
   ```

## How it works

A patch for the Nix evaluator adds primitives to create tracking scopes, tag
thunks with origin paths, and record attribute accesses. A small integration
in `lib/modules.nix` and `eval-config.nix` uses these primitives to tag all
option value thunks and register the `config`/`options` attrsets for tracking.

When any option value is forced during evaluation, its origin path is pushed as
the "current accessor" context. Any accesses to tracked attrsets during that
evaluation are recorded as dependencies. After evaluation, all edges and config
values are serialized into the toplevel output.

For a detailed technical explanation, see the [blog post](https://oddlama.org/blog/tracking-options-in-nixos/).

## Patches

The patches for Nix and nixpkgs are available in `contrib/`:

| Patch | Description |
|-------|-------------|
| `contrib/nix-add-thunk-origins.diff` | tracking builtins |
| `contrib/nixpkgs-add-tracking.diff` | `evalModules` integration, dependency-tracking.nix post-processing |

They are also maintained as branches:

- **nix:** [`oddlama/nix@thunk-origins-v1`](https://github.com/oddlama/nix/tree/thunk-origins-v1)
- **nixpkgs:** [`oddlama/nixpkgs@thunk-origins-v1`](https://github.com/oddlama/nixpkgs/tree/thunk-origins-v1)

## Raw tracking data

If you want to work with the tracking data directly instead of using
`nixos-config`, all information is available on the evalModules result:

```nix
let
  nixos = nixpkgs.lib.nixosSystem {
    trackDependencies = true;
    modules = [ ./configuration.nix ];
  };
  # Force evaluation of toplevel first to record all dependencies
  dependencyTracking = builtins.seq nixos.config.system.build.toplevel nixos.dependencyTracking;
in {
  toplevel = nixos.config.system.build.toplevel;
  inherit (dependencyTracking)
    rawDeps              # all raw dependency edges
    filteredDeps         # filtered + transitive closure
    configValues         # all leaf config values (JSON-safe)
    explicitConfigValues # only explicitly defined value (JSON-safe)
    leafNodes            # leaf node paths
    keptNodes            # all kept node paths
    counts               # summary statistics
    rawDotOutput         # Graphviz DOT of raw deps
    filteredDotOutput;   # Graphviz DOT of filtered deps
}
```

## License

Licensed under either of

- Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or <https://www.apache.org/licenses/LICENSE-2.0>)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or <https://opensource.org/licenses/MIT>)

at your option.
Unless you explicitly state otherwise, any contribution intentionally
submitted for inclusion in this project by you, as defined in the Apache-2.0 license,
shall be dual licensed as above, without any additional terms or conditions.
