# Debian packages from Cargo projects

This is a [Cargo](https://doc.rust-lang.org/cargo/) helper command which automatically creates binary [Debian packages](https://www.debian.org/doc/debian-policy/ch-binary.html) (`.deb`) from Cargo projects.

## Installation

```sh
rustup update   # Bookworm's Rust is too outdated, use Trixie or rustup.rs
cargo install cargo-deb
```

Requires Rust 1.76+, and optionally `dpkg`, `dpkg-dev` and `liblzma-dev`. Compatible with Ubuntu. If the LZMA dependency causes you headaches, try `cargo install cargo-deb --no-default-features`.

If you get a compilation error, run `rustup update`! If you get an error running `rustup update`, uninstall your rust/cargo package, and install [the official Rust](https://rustup.rs/) instead.

## Usage

```sh
cargo deb
```

When you run `cargo deb` from the base directory of your Rust/Cargo project, the Debian package will be created in `target/debian/<project_name>_<version>-1_<arch>.deb` (or you can change the location with the `--output` option).

This package can be installed with `dpkg -i target/debian/*.deb`. `cargo deb --install` builds and immediately installs the package on the local system.

## Optional Configuration

No configuration is necessary to make a basic package from a Cargo project with a binary. This command obtains basic information it needs from [the `Cargo.toml` file](https://doc.rust-lang.org/cargo/reference/manifest.html). It uses Cargo fields: `name`, `version`, `license`, `license-file`, `description`, `readme`, and `homepage` or `repository`. For a more complete Debian package, you may also define a new table, `[package.metadata.deb]` that contains additional metadata (described below).

For a Debian package that includes one or more systemd unit files you may also wish to define a new (inline) table, `[package.metadata.deb.systemd-units]`, so that the unit files are automatically added as assets and the units are properly installed. [Systemd integration](./systemd.md)

### Debug symbols

Debug symbols are stripped from built binaries by default, unless `[profile.release] debug` is enabled in `Cargo.toml` (`debug = "line-tables-only"` is recommended). The `--separate-debug-symbols` and `--dbgsym` options package debug symbols as separate files installed at `/usr/lib/debug/<build-id-or-path>.debug`. This can also be enabled via `[package.metadata.deb]` under `separate-debug-symbols`.

### `[package.metadata.deb]` options

Everything is optional:

- **name**: The name of the Debian package. If not present, the name of the crate is used.
- **maintainer**: The person maintaining the Debian packaging. If not present, the first author is used. Can be set via `--maintainer` on the command line.
- **copyright**: To whom and when the copyright of the software is granted. If not present, the list of authors is used.
- **license-file**: 2-element array with a location of the license file and the amount of lines to skip at the top. If not present, package-level `license-file` is used.
- **depends**: The runtime [dependencies](https://www.debian.org/doc/debian-policy/ch-relationships.html) of the project. Generated automatically when absent, or if the list includes the `$auto` keyword.
- **pre-depends**: The [pre-dependencies](https://www.debian.org/doc/debian-policy/ch-relationships.html) of the project. This will be empty by default.
- **recommends**: The recommended [dependencies](https://www.debian.org/doc/debian-policy/ch-relationships.html) of the project. This will be empty by default.
- **suggests**: The suggested [dependencies](https://www.debian.org/doc/debian-policy/ch-relationships.html) of the project. This will be empty by default.
- **enhances**: A list of packages this package can enhance. This will be empty by default.
- **conflicts**, **breaks**, **replaces**, **provides** — [package transition](https://wiki.debian.org/PackageTransition) control.
- **extended-description**: An extended description of the project — the more detailed the better. Either **extended-description-file** (see below) or package's `readme` file is used if it is not provided.
- **extended-description-file**: A file with extended description of the project. When specified, used if **extended-description** is not provided.
- **revision**: An additional version of the Debian package (when the package is updated more often than the project). It defaults to "1", but can be set to an empty string to omit the revision. Can be set via `--deb-revision` on the command line.
- **section**: The [application category](https://packages.debian.org/bookworm/) that the software belongs to.
- **priority**: Defines if the package is `required` or `optional`.
- **assets**: Files to be included in the package and the permissions to assign them. If assets are not specified, they default to `["$auto"]`, which takes binaries built by Cargo (copied to `/usr/bin/`) and the package's `readme` (copied to `usr/share/doc/…`).
    1. `source`: the first argument of each asset is the location of that asset in the Rust project. Glob patterns are allowed. Always use `target/release/` path prefix for packaging binaries built by Cargo, *even if that's not the real path* to your target directory. Cargo-deb uses this prefix to detect what to compile, and will replace it with the actual target dir path, taking into account cross-compilation, build profiles, workspaces, `CARGO_TARGET_DIR`, custom configs, etc. If you try to "fix" the hardcoded `target/release` paths, you will break cargo-deb, and make it package stale files and mishandle debug info.
    2. `dest`: the second argument is where the file will be copied. If it starts with `usr/lib`, it will be changed to `usr/lib/$tuple` when multiarch option is enabled.
        - If is argument ends with `/` it will be inferred that the target is the directory where the file will be copied.
        - Otherwise, it will be inferred that the source argument will be renamed when copied.
    3. `mode`: the third argument is the permissions (octal string) to assign that file.
- **merge-assets**: [See "Merging Assets" section under "Advanced Usage"](#merging-assets)
- **maintainer-scripts**: directory containing `templates`, `preinst`, `postinst`, `prerm`, or `postrm` [scripts](https://www.debian.org/doc/debian-policy/ch-maintainerscripts.html).
- **conf-files**: [List of configuration files](https://www.debian.org/doc/manuals/maint-guide/dother.en.html#conffiles) that the package management system will not overwrite when the package is upgraded.
- **triggers-file**: Path to triggers control file for use by the dpkg trigger facility.
- **changelog**: Path to Debian-formatted [changelog file](https://www.debian.org/doc/manuals/maint-guide/dreq.en.html#changelog).
- **features**: List of [Cargo features](https://doc.rust-lang.org/cargo/reference/manifest.html#the-features-section) to use when building the package.
- **default-features**: whether to use default crate features in addition to the `features` list (default `true`).
- **separate-debug-symbols**: whether to keep debug symbols, but strip them from executables and save them in separate files (default `false`). If it is enabled, then `cargo deb --no-separate-debug-symbols` can be used to suppress extraction of the debug symbols.
- **preserve-symlinks**: Whether to preserve symlinks in the asset files (default `false`).
- **systemd-units**: Optional configuration settings for automated installation of [systemd units](./systemd.md).
- **conf-files**: List of absolute paths of [config files outside `/etc`](https://www.debian.org/doc/manuals/maint-guide/dother.en.html#conffiles) `["/not-etc/app/config"]`. You still need to list the files in `assets` to have them packaged.
- **profile**: Cargo build profile to use. Defaults to `release`.

### Example of custom `Cargo.toml` additions

```toml
[package.metadata.deb]
maintainer = "Michael Aaron Murphy <mmstickman@gmail.com>"
copyright = "2017, Michael Aaron Murphy <mmstickman@gmail.com>"
license-file = ["LICENSE", "4"]
extended-description = """\
A simple subcommand for the Cargo package manager for \
building Debian packages from Rust projects."""
depends = "$auto"
section = "utility"
priority = "optional"
assets = [
    # target/release path is special, and gets replaced by cargo-deb with the actual target dir path.
    ["target/release/cargo-deb", "usr/bin/", "755"],
    # both array and object syntaxes are equivalent:
    { source = "README.md", dest = "usr/share/doc/cargo-deb/README", mode = "644"},
]
# assets = ["$auto"] # the default if assets are not specified
```

## Advanced usage

`--fast` flag uses quicker/lighter compression in the `.deb` file. Useful for quick deployment of large packages. `--dbgsym` can also improve speed by making two packages in parallel.

[`--multiarch=same`](https://wiki.ubuntu.com/MultiarchSpec#Binary_package_control_fields) will use `/usr/lib/$debian-target-tuple/` for library paths, allowing library packages for different architectures to co-exist.

### `[package.metadata.deb.variants.$name]`

There can be multiple variants of the metadata in one `Cargo.toml` file. `--variant=name` selects the variant to use. Options set in a variant override `[package.metadata.deb]` options. It automatically appends the variant name to the package name (or you can set the `name` in the variant).

### Separate debug info

To get debug symbols in the package, set in `Cargo.toml`:

```toml
[profile.release]
debug = "line-tables-only"
# or debug = 1 for fatter debug info
```

Note: building using the `dev` profile is intentionally unsupported. You can specify other profiles with `--profile`.

#### Separate `-dbgsym.ddeb` package

```sh
cargo deb --dbgsym
```

Removes debug symbols from the executables, and makes a second `-dbgsym.ddeb` package with only `/usr/lib/debug/.build-id/*` files. Requires GNU `objcopy` tool. The `.ddeb` package is a regular deb package that can be installed together with the base `.deb` package.

#### Separate debug files in the same package

```sh
cargo deb --separate-debug-symbols --compress-debug-symbols
```

Removes debug symbols from the executables, and places them in separate files in `/usr/lib/debug/.build-id/*`. Requires GNU `objcopy` tool. `--compress-debug-symbols` makes the debug symbols compressed internally, taking less space on disk.

### Merging Assets

When defining a variant it can be useful to also define different assets. If the `merge-assets` option is used, `cargo-deb` will merge the list of assets provided to the option with the parent asset list. There are three merging strategies, `append`, `by.dest`, and `by.src`.

- **merge-assets.append**: Appends this list of assets to the parent list of assets.
- **merge-assets.by.dest**: Merges this list of assets to the parent list of assets, joining on the destination path. Will replace both the source path and permissions.
- **merge-assets.by.src**: Merges this list of assets to the parent list of assets, joining on the source path. Will replace both the destination path and permissions.

**Note**: Using both `append`, and a `by.*` option are allowed, w/ the former being applied before the latter.

The special `"$auto"` entry is not expanded before merging. Explicit paths take precedence over `"$auto"`.

#### Example of `merge-assets`

```toml
# Example parent asset list
[package.metadata.deb]
assets = [
    # binary
    ["target/release/example", "usr/bin/", "755"],
    # assets
    ["assets/*", "var/lib/example", "644"],
    ["target/release/assets/*", "var/lib/example", "644"],
    ["3.txt", "var/lib/example/3.txt", "644"],
    ["3.txt", "var/lib/example/merged.txt", "644"],
]

# Example merging by appending asset list
[package.metadata.deb.variants.mergeappend]
merge-assets.append = [
    ["4.txt", "var/lib/example/appended/4.txt", "644"]
]

# Example merging by `dest` path
[package.metadata.deb.variants.mergedest]
merge-assets.by.dest = [
    ["4.txt", "var/lib/example/merged.txt", "644"]
]

# Example merging by `src` path
[package.metadata.deb.variants.mergesrc]
merge-assets.by.src = [
    ["3.txt", "var/lib/example/merged-2.txt", "644"]
]

# Example merging by appending and by `src` path
[package.metadata.deb.variants.mergeappendandsrc]
merge-assets.append = [
    ["4.txt", "var/lib/example/appended/4.txt", "644"]
]
merge-assets.by.src = [
    ["3.txt", "var/lib/example/merged-2.txt", "644"]
]
```

### `[package.metadata.deb.systemd-units]`

[See systemd integration](./systemd.md).

### Cross-compilation

`cargo deb` supports cross-compilation. It can be run from any unix-like host, including macOS, provided that the build environment is set up for cross-compilation:

* The cross-compilation target has to be [installed via rustup](https://github.com/rust-lang-nursery/rustup.rs#cross-compilation) (e.g. `rustup target add i686-unknown-linux-gnu`) and has to be [installed for the host system](https://wiki.debian.org/ToolChain/Cross) (e.g. `apt-get install libc6-dev-i386`). Note that [Rust's](https://forge.rust-lang.org/release/platform-support.html) and [Debian's architecture names](https://www.debian.org/ports/) are different. See `rustc --print target-list` for the list of supported values for the `--target` argument.
* A Linux-compatible linker and system libraries (e.g. glibc or musl) must be installed and available to Rust/Cargo,
   * `dpkg --add-architecture <debian architecture name>`
   * `apt-get install pkg-config build-essential crossbuild-essential-<debian architecture name>`
* Cargo must be [configured to use a cross-linker](https://doc.rust-lang.org/cargo/reference/config.html#targettriplelinker).
* Cargo dependencies that use C libraries probably won't work, unless you install a target's sysroot for `pkg-config`. Setting `PKG_CONFIG_ALLOW_CROSS=1` *will not help* at all, and will only make things *worse*.
   * `apt-get install libssl-dev:<debian architecture name>`
* Cargo dependencies that build C code probably won't work, unless you install a C compiler for the target system, and configure appropriate `CC_<target>` variables.
   * `export HOST_CC=gcc`
   * `export CC_x86_64_unknown_linux_gnu=/usr/bin/x86_64-linux-gnu-gcc` (correct the target and paths for your OS)
* Stripping probably won't work, unless you install versions compatible with the target and configure their paths in `.cargo/config` by adding `[target.<target triple>] strip = { path = "…" } objcopy = { path = "…" }`. Alternatively, use `--no-strip`.

Yes, these requirements are onerous. You can also try [`cross`](https://lib.rs/crates/cross) or [`cargo zigbuild`](https://lib.rs/crates/cargo-zigbuild), since Zig is way better at cross-compiling, and then run `cargo deb --target=… --no-build`.

```sh
cargo deb --target=i686-unknown-linux-gnu
```

Cross-compiled archives are saved in `target/<target triple>/debian/*.deb`. The actual archive path is printed on success.

Note that you can't use cross-compilation to build for an older version of Debian. If you need to support Debian releases older than the host, consider using a container or a VM, or make a completely static binary for MUSL instead.

### Custom build flags

If you would like to handle the build process yourself, you can use `cargo deb --no-build` so that the `cargo-deb` command will not attempt to rebuild your project.

    cargo deb -- <cargo build flags>

Flags after `--` are passed to `cargo build`, so you can use options such as `-Z`, `-j`, or `--timings`. Please use that only for options that `cargo-deb` doesn't support natively.

### Workspaces

Cargo-deb understands workspaces and can build all crates in the workspace if necessary. However, you must choose one crate to be the source of the package metadata. You can select which crate to build with `-p crate_name` or `--manifest-path=<path/to/Cargo.toml>`.

### Custom version strings

    cargo deb --deb-version 1.my-custom-version

Overrides the version string generated from the Cargo manifest, including revision. Alternatively, `--deb-revision` can be used to change only the suffix.

## Troubleshooting

For maximum logging, use:

```sh
cargo deb -vv
RUST_LOG=debug cargo deb -vv
```

### Undefined reference to `lzma_stream_encoder_mt` error

This happens when the system-provided LZMA library is too old. Try with a bundled version:

```sh
cargo install cargo-deb --features=static-lzma
```

or use the xz command-line tool by setting the `--compress-system` flag.

> [!NOTE]
> cargo-deb uses the [xz2](https://lib.rs/crates/xz2) crate that bundles an old safe version of liblzma 5.2 by the original maintainer, and a simple Cargo-based build script.
> It is **unaffected** by the CVE-2024-3094.
