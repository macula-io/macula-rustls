# Macula fork of `rustls` 0.23.40

Vendored fork of [rustls/rustls](https://github.com/rustls/rustls) at
version `0.23.40`, mechanically widened so the entire `feature = "std"`
surface activates on `target_os = "none"` via the [`macula-std`] shim.

Used by [macula-kernel](https://codeberg.org/macula-internal/macula-kernel)
as the TLS crypto backend for kernel-resident `quinn-proto`. quinn-proto
reaches into the `rustls::quic::{Connection, ClientConnection,
ServerConnection}` high-level wrapper types + the `builder_with_provider`
constructors on `ClientConfig` / `ServerConfig`, all of which are upstream
gated behind `feature = "std"`. We cannot enable that feature directly
because rustls does `extern crate std;` under it, and the kernel target
has no `std` crate.

[`macula-std`]: https://codeberg.org/macula-internal/macula-std

## The patches

### src/lib.rs — extern crate aliases

```diff
-#[cfg(any(feature = "std", test))]
+#[cfg(all(any(feature = "std", test), not(target_os = "none")))]
 extern crate std;
+
+#[cfg(target_os = "none")]
+extern crate macula_std as std;
```

The two arms are mutually exclusive. Host builds with `feature = "std"`
keep the upstream `extern crate std;` exactly as it was. Kernel builds
get the `macula-std` shim aliased as `std`, so every `use std::*` path
in the crate resolves through it.

### Bulk cfg widening (~135 gates)

Every `cfg(feature = "std")` is widened to
`cfg(any(feature = "std", target_os = "none"))`. Every
`cfg(not(feature = "std"))` is widened to
`cfg(not(any(feature = "std", target_os = "none")))`. A few `cfg_attr`
variants (`derive(Debug)`, `allow(dead_code)`, `doc(hidden)`) get the
same treatment. The `cfg(any(feature = "std", feature = "hashbrown"))`
sites also get `target_os = "none"` added as a third disjunct.

This activates the entire `feature = "std"` surface on the kernel
target without flipping the feature itself. Host builds remain
identical.

### Cargo.toml — macula-std target dep

```diff
+[target.'cfg(target_os = "none")'.dependencies.macula-std]
+git = "https://codeberg.org/macula-internal/macula-std.git"
+branch = "main"
```

Pulled in only when compiling for the kernel target.

## Why widen, not enable

Enabling `feature = "std"` from the consumer side would force
`extern crate std;` at lib.rs:392, which fails on `target_os = "none"`
because there is no `std` sysroot crate. The widened-cfg approach
sidesteps the cargo-feature unification and re-routes every std-tagged
item through `macula-std` purely via cfg.

The downside is that this fork's source diverges from upstream more
than the macula1 macula-bytes / macula-rustc-hash patches did. Mitigate
by re-rolling on each upstream 0.23.x bump with the same sed pipeline;
see "Versioning policy" below.

## Versioning policy

Track `rustls` upstream releases of 0.23.x. Re-roll this fork by:

1. Downloading the new `rustls-X.Y.Z.crate` tarball from crates.io.
2. Extracting into a fresh checkout (replaces this directory contents).
3. Applying the lib.rs extern-crate-alias patch (manual, ~10 lines).
4. Re-running the cfg-widening sed:
   ```bash
   find src -type f -name "*.rs" -exec sed -i \
     -e 's|cfg(feature = "std")|cfg(any(feature = "std", target_os = "none"))|g' \
     -e 's|cfg(not(feature = "std"))|cfg(not(any(feature = "std", target_os = "none")))|g' \
     -e 's|cfg(any(feature = "std", feature = "hashbrown"))|cfg(any(feature = "std", feature = "hashbrown", target_os = "none"))|g' \
     -e 's|cfg_attr(feature = "std", derive(Debug))|cfg_attr(any(feature = "std", target_os = "none"), derive(Debug))|g' \
     -e 's|cfg_attr(not(feature = "std"), allow(dead_code))|cfg_attr(not(any(feature = "std", target_os = "none")), allow(dead_code))|g' \
     {} \;
   ```
5. Appending the macula-std `[target.'cfg(target_os = "none")'.dependencies]`
   block to `Cargo.toml`.
6. Spot-check `cargo build --target x86_64-macula.json` succeeds.
7. Tagging as `vX.Y.Z-macula1`.
8. Bumping the git ref in macula-kernel's `[patch.crates-io]` block.

When upstream gains first-class no_std support (the long-running
[rustls#3070](https://github.com/rustls/rustls/issues/3070) tracks
this), retire the fork.

## Upstream contribution

The upstream maintainers are aware of the no_std demand and have
landed many of the gates intentionally; widening them via a single
custom-cfg knob (or a `kernel` feature) might be acceptable as a PR.
File one once we have working downstream evidence that the fork
actually carries weight at runtime.

## License

Inherited from upstream rustls: Apache-2.0 OR ISC OR MIT. No new code
beyond the cfg widening + extern-crate alias.
