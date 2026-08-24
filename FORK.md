# trjh/herdr — tab status colors

A single-feature fork of [herdrdev/herdr](https://github.com/herdrdev/herdr),
branched from `v0.8.2`.

## What the patch adds

`[ui] tab_status_colors` (default `false`). When enabled, each tab cell in the
desktop tab row takes the color of the agent state inside it, and the active
tab is that same cell inverted — the grammar the plain tabs already use.

| Tab holds | Inactive | Active |
|---|---|---|
| a blocked agent | `red` fill, contrasting ink | inverted, bold |
| a working agent | `yellow` fill | inverted, bold |
| a finished (unseen) agent | `teal` fill | inverted, bold |
| a seen-idle agent | `green` fill | inverted, bold |
| **no agent** | unchanged | unchanged |

Every tab holding a detected agent takes a color. The plain styling therefore
means exactly one thing — *this is not an agent tab* — so a shell, an editor,
or a log tail stays visually distinct from an agent that simply has nothing
left to report.

The colors are the theme's existing semantic tokens, so `[theme.custom]`
retints them:

```toml
[ui]
tab_status_colors = true

[theme.custom]
red    = "#DC3C28"   # blocked
yellow = "#FFA028"   # working
teal   = "#0064C8"   # done
green  = "#22B450"   # idle
```

Foreground is chosen by the luma of the fill, so labels stay legible in light
and dark themes. An ANSI color that cannot be measured (the `terminal` theme)
falls back to the contrast color the active tab already uses.

## Why a fork and not a pull request

herdr does not accept unsolicited pull requests — see
[CONTRIBUTING.md](CONTRIBUTING.md). Implementation PRs are limited to
maintainers and the names in `.github/APPROVED_CONTRIBUTORS`, and everything
else is closed automatically. Upstream issue
[#2979](https://github.com/herdrdev/herdr/issues/2979) proposed exactly this
feature and was closed `not planned` with a pointer to Discussions.

So: this lives here, and upstream interest belongs in a Discussion, not a PR.

## Building on macOS 26

Two things bite, neither of them herdr's fault. Both are worked around without
touching the repo or any system state.

**1. zig version.** `vendor/libghostty-vt` requires zig **0.15.2** exactly
(`flake.nix` pins `zig_0_15`). Homebrew ships 0.16, which fails at
`requireZig`. Fetch the pinned build:

```bash
curl -fsSL -o zig.tar.xz \
  "$(curl -fsSL https://ziglang.org/download/index.json \
     | jq -r '."0.15.2"."aarch64-macos".tarball')"
tar xf zig.tar.xz
```

**2. zig 0.15.2 cannot link libc against the macOS 26 SDK.** That SDK's
`usr/lib/libSystem.B.tbd` lists `arm64e-macos` but not plain `arm64-macos`, so
zig's `aarch64-macos` target resolves none of libc and every symbol comes back
undefined — reproducible with a zig hello-world, nothing to do with herdr.
The 15.4 SDK still carries the `arm64-macos` slice.

zig finds the SDK by shelling out to `xcrun --show-sdk-path`, so a shim on
`PATH` redirects it. `--sysroot` is *not* enough: zig links its own build
runner before that flag applies.

```bash
mkdir -p /tmp/shim && cat > /tmp/shim/xcrun <<'SH'
#!/bin/bash
for a in "$@"; do
  [ "$a" = --show-sdk-path ] && {
    echo /Library/Developer/CommandLineTools/SDKs/MacOSX15.4.sdk; exit 0; }
done
exec /usr/bin/xcrun "$@"
SH
chmod +x /tmp/shim/xcrun
```

**3. (only on networks with broken IPv6)** zig's package fetcher hard-fails
with `NetworkUnreachable` where curl succeeds. Pre-seed the cache — the hashes
are checked against the manifests, so this is not a trust shortcut:

```bash
find vendor/libghostty-vt ~/.cache/zig/p -name build.zig.zon \
  | xargs grep -hoE 'https://[^" ]+\.(tar\.gz|tar\.xz|tar\.zst|tgz)' | sort -u \
  | while read -r u; do
      f=/tmp/zigdeps/$(basename "$u")
      [ -s "$f" ] || curl -fsSL --ipv4 --create-dirs -o "$f" "$u"
      zig fetch "$f"
    done
```

Then:

```bash
PATH=/tmp/shim:$PATH ZIG=/path/to/zig-0.15.2/zig cargo build --release
```

Install alongside the official binary rather than over it, so `herdr update`
still has something to manage:

```bash
install -m 755 target/release/herdr ~/.local/bin/herdr-tabcolors
```

## Tests

```bash
PATH=/tmp/shim:$PATH ZIG=/path/to/zig-0.15.2/zig cargo test --bin herdr ui::tabs
```
