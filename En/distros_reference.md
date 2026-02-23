# `distros` Module Reference

**Module:** `distros` (part of Nim's standard library)  
**Purpose:** Detect the current Linux distribution (or other OS) at runtime, and generate the correct native package-manager command needed to install a foreign dependency.

---

## Overview

When a Nim package depends on a C library or system tool, it cannot install that dependency itself — it must ask the user to run their OS's package manager. The challenge is that every Linux distribution uses a different package manager (`apt-get`, `pacman`, `yum`, `emerge`, …) and every distribution may give the same library a slightly different name.

`distros` solves both halves of this problem:

1. **Detection** — `detectOs` tells you which OS or distribution you are running on.
2. **Command generation** — `foreignDep` / `foreignDepInstallCmd` produce the exact `apt-get install …`, `pacman -S …`, or equivalent command for the current system.

The module is mainly used from `config.nims` / Nimble build scripts, but it works in any Nim program.

### Detection strategy

For generic OS families (Windows, Linux, macOS, BSD, POSIX) detection is done via Nim's compile-time `defined()` checks — fast and zero-cost.

For specific Linux distributions the module queries up to four shell commands, caching each result after the first call:

| Command | What it reveals |
|---|---|
| `cat /etc/os-release \| grep ^ID=` | Official distro ID (`ubuntu`, `arch`, `fedora`, …) |
| `lsb_release -d` | Human-readable description string |
| `uname -a` | Kernel + architecture + hostname |
| `hostnamectl` | Systemd-based distro/OS name |

NixOS and Nix build environments are detected by checking environment variables `NIX_BUILD_TOP` and `__NIXOS_SET_ENVIRONMENT_DONE` instead of running shell commands.

---

## Exported API

| Symbol | Kind | Description |
|---|---|---|
| `Distribution` | enum | All known OSes and Linux distributions |
| `LacksDevPackages` | const set | Distributions that do not split `-dev` packages |
| `foreignDeps` | var (seq) | Accumulated list of foreign dependency commands |
| `detectOs(d)` | template | Detect whether the current OS matches `d` |
| `foreignDepInstallCmd(pkg)` | proc | Return the install command for `pkg` on the current OS |
| `foreignCmd(cmd, requiresSudo)` | proc | Register an arbitrary shell command as a dependency |
| `foreignDep(pkg)` | proc | Register a package name as a foreign dependency |
| `echoForeignDeps()` | proc | Print all registered foreign dependencies to stdout |

---

## `Distribution`

```nim
type Distribution* {.pure.} = enum
  Windows, Posix, MacOSX, Linux,
  Ubuntu, Debian, Gentoo, Fedora, RedHat,
  OpenSUSE, Manjaro, Alpine, ArchLinux, NixOS,
  BSD, FreeBSD, NetBSD, OpenBSD, DragonFlyBSD,
  Haiku,
  ... # ~80 more values
```

### What it is

A `{.pure.}` enum listing every OS family and Linux distribution the module knows about. Because it is `{.pure.}`, you must qualify each value with `Distribution.` when using it directly in code — however, `detectOs` adds this qualifier for you automatically, so in practice you never type `Distribution.` yourself.

The enum spans:

- **Generic families:** `Windows`, `Posix`, `MacOSX`, `Linux`, `BSD` — detected at compile time via `defined()`.
- **BSD variants:** `FreeBSD`, `NetBSD`, `OpenBSD`, `DragonFlyBSD` — detected via `uname`.
- **Major Linux distros:** `Ubuntu`, `Debian`, `Fedora`, `RedHat`, `ArchLinux`, `Gentoo`, `Alpine`, `NixOS`, and many more — detected via `/etc/os-release`, `lsb_release`, `uname`, or `hostnamectl`.
- **Haiku** — detected via `defined(haiku)`.

### Example

```nim
import distros

# Enumerate all values of the enum
for d in Distribution:
  echo d
```

---

## `LacksDevPackages`

```nim
const LacksDevPackages* = {
  Distribution.Gentoo,
  Distribution.Slackware,
  Distribution.ArchLinux,
  Distribution.Artix,
  Distribution.Antergos,
  Distribution.BlackArch,
  Distribution.ArchBang
}
```

### What it is

A compile-time constant set of `Distribution` values that identifies distros where development header packages are **not** split from runtime packages. On Debian/Ubuntu-style systems a library ships as two packages: `libfoo` (runtime) and `libfoo-dev` (headers + static lib). On Arch Linux, Gentoo, and Slackware there is only one package containing both.

### When to use

Consult `LacksDevPackages` when you need to decide whether to request a `-dev` suffix:

```nim
import distros

proc installLibfoo() =
  if detectOs(Ubuntu) or detectOs(Debian):
    foreignDep "libfoo-dev"   # need the -dev variant
  elif detectOs(ArchLinux):
    foreignDep "libfoo"       # Arch ships headers in the same package

# Or generically:
proc devPkg(baseName: string): string =
  for d in LacksDevPackages:
    if detectOsImpl(d): return baseName
  return baseName & "-dev"
```

---

## `foreignDeps`

```nim
var foreignDeps*: seq[string] = @[]
```

### What it is

A module-level sequence that accumulates every shell command registered through `foreignCmd` or `foreignDep`. Each element is a ready-to-run string such as `"sudo apt-get install libssl-dev"`.

This variable is only defined when the module is **not** compiled under Nimble (`not defined(nimble)`). Inside a Nimble script, Nimble provides its own `foreignDeps` via `nimscriptapi`.

You can iterate over `foreignDeps` yourself, or call `echoForeignDeps()` to print them all.

### Example

```nim
import distros

foreignDep "zlib"
foreignDep "openssl"

echo "Run these commands to install dependencies:"
for cmd in foreignDeps:
  echo "  ", cmd
```

---

## `detectOs`

```nim
template detectOs*(d: untyped): bool
```

### What it does

Detects whether the program is currently running on the OS or distribution named by `d`. Returns `true` if the current environment matches, `false` otherwise.

`detectOs` is a **template**, not a proc, which gives it two practical advantages:

- **No `Distribution.` prefix required.** You write `detectOs(Ubuntu)` instead of `detectOs(Distribution.Ubuntu)`.
- **Compile-time short-circuit for OS families.** For `Windows`, `Linux`, `MacOSX`, `Posix`, and `BSD` the check reduces to a `defined()` call and is evaluated at compile time with zero runtime overhead.

For Linux-specific distributions the template delegates to `detectOsImpl`, which runs the appropriate shell commands (cached after the first call).

### Detection details per distribution

| Distribution | Method |
|---|---|
| `Windows`, `Linux`, `MacOSX`, `Posix`, `BSD` | `defined(...)` at compile time |
| `FreeBSD`, `NetBSD`, `OpenBSD` | `$d in uname()` |
| `Gentoo` | `"-Gentoo "` in `uname()` |
| `Ubuntu`, `Debian`, `Fedora`, `Alpine`, `Void`, `CentOS`, `Mageia`, `Zorin`, `Elementary`, `OpenMandriva` | distro name in `/etc/os-release` ID field |
| `RedHat` | `"rhel"` in `/etc/os-release` ID field |
| `ArchLinux` | `"arch"` in `/etc/os-release` ID field |
| `Artix` | `"artix"` in `/etc/os-release` ID field |
| `NixOS` | env var `NIX_BUILD_TOP` or `__NIXOS_SET_ENVIRONMENT_DONE` exists |
| `OpenSUSE` | `"suse"` in `uname()` or `lsb_release` |
| `GoboLinux` | `"-Gobo "` in `uname()` |
| `Solaris` | `"sun"` or `"solaris"` in `uname()` |
| `Haiku` | `defined(haiku)` |
| All other distros | name checked in all four sources (`os-release`, `lsb_release`, `uname`, `hostnamectl`) |

### When to use

Use `detectOs` at the top of any platform-specific branch in a Nimble build script or `config.nims` to choose the right package names and install commands for the current system.

### Examples

```nim
import distros

# Example 1 — simple binary branch
if detectOs(Ubuntu) or detectOs(Debian):
  foreignDep "libssl-dev"
elif detectOs(ArchLinux) or detectOs(Manjaro):
  foreignDep "openssl"
elif detectOs(Fedora):
  foreignDep "openssl-devel"

# Example 2 — nested checks for library name differences
if detectOs(Linux):
  if detectOs(Alpine):
    foreignDep "sqlite-dev"   # Alpine names it differently
  elif detectOs(Fedora) or detectOs(RedHat) or detectOs(CentOS):
    foreignDep "sqlite-devel"
  else:
    foreignDep "libsqlite3-dev"
elif detectOs(MacOSX):
  foreignDep "sqlite3"
elif detectOs(Windows):
  foreignDep "sqlite"  # via Chocolatey

# Example 3 — generic OS family only
if detectOs(Windows):
  echo "Running on Windows"
elif detectOs(MacOSX):
  echo "Running on macOS"
elif detectOs(Linux):
  echo "Running on Linux"

# Example 4 — NixOS / Nix environment
if detectOs(NixOS):
  foreignDep "openssl"       # nix-env -i openssl (no sudo needed)
else:
  foreignDep "libssl-dev"
```

---

## `foreignDepInstallCmd`

```nim
proc foreignDepInstallCmd*(foreignPackageName: string): (string, bool)
```

### What it does

Given a package name string, returns a tuple of:
- The complete shell command to install that package on the current OS (e.g. `"apt-get install libz-dev"`).
- A `bool` indicating whether the command requires root / administrator privileges (`true` = needs sudo/root, `false` = can run as a normal user).

The command is selected based on the detected OS/distribution:

| OS / Distribution | Command | Needs root |
|---|---|---|
| Windows | `choco install <pkg>` | `false` |
| BSD (generic) | `ports install <pkg>` | `true` |
| Ubuntu, Debian, Elementary, KNOPPIX, SteamOS | `apt-get install <pkg>` | `true` |
| Gentoo | `emerge install <pkg>` | `true` |
| Fedora | `yum install <pkg>` | `true` |
| RedHat | `rpm install <pkg>` | `true` |
| OpenSUSE | `yast -i <pkg>` | `true` |
| Slackware | `installpkg <pkg>` | `true` |
| OpenMandriva | `urpmi <pkg>` | `true` |
| ZenWalk | `netpkg install <pkg>` | `true` |
| NixOS | `nix-env -i <pkg>` | `false` |
| Solaris, FreeBSD | `pkg install <pkg>` | `true` |
| NetBSD, OpenBSD | `pkg_add <pkg>` | `true` |
| PCLinuxOS | `rpm -ivh <pkg>` | `true` |
| ArchLinux, Manjaro, Artix | `pacman -S <pkg>` | `true` |
| Void | `xbps-install <pkg>` | `true` |
| Haiku | `pkgman install <pkg>` | `true` |
| macOS / other | `brew install <pkg>` | `false` |
| Unknown Linux | `<your package manager here> install <pkg>` | `true` |

### When to use

Call `foreignDepInstallCmd` when you need the command string for display, logging, or to construct a more complex install script. If you just want to register the dependency in the standard way, call `foreignDep` instead — it calls `foreignDepInstallCmd` internally.

### Examples

```nim
import distros

# Inspect what command would be used without registering it
let (cmd, needsSudo) = foreignDepInstallCmd("libpng-dev")
echo "Install command: ", cmd
echo "Needs root: ", needsSudo

# Conditionally modify the command before registering
let (baseCmd, sudo) = foreignDepInstallCmd("ffmpeg")
if needsSudo:
  foreignCmd("sudo " & baseCmd)
else:
  foreignCmd(baseCmd)
```

---

## `foreignCmd`

```nim
proc foreignCmd*(cmd: string; requiresSudo = false)
```

### What it does

Appends a fully-formed shell command to the `foreignDeps` list. If `requiresSudo` is `true`, the string `"sudo "` is prepended to `cmd` before storing it.

This is the lowest-level registration proc. All higher-level procs (`foreignDep`, and indirectly `foreignDepInstallCmd`) eventually call `foreignCmd`.

Inside a Nimble script, the command is added to Nimble's own `nimscriptapi.foreignDeps` instead of the module-level `foreignDeps` variable.

### When to use

Use `foreignCmd` when:
- You need to register a command that is not a simple package install (e.g. a `pip install`, a `gem install`, or a multi-step build script).
- You have already built the install command string yourself and simply need to record it.
- You want full control over the exact string that appears in the dependency list.

### Examples

```nim
import distros

# Example 1 — register a Python package requirement
foreignCmd("pip install numpy", requiresSudo = false)

# Example 2 — register a manual build step (needs root)
foreignCmd("make install", requiresSudo = true)

# Example 3 — platform-conditional custom command
if detectOs(Ubuntu):
  foreignCmd("add-apt-repository ppa:example/ppa", requiresSudo = true)
  foreignCmd("apt-get update", requiresSudo = true)
  foreignCmd("apt-get install example-lib-dev", requiresSudo = true)
elif detectOs(MacOSX):
  foreignCmd("brew tap example/tap")
  foreignCmd("brew install example-lib")
```

---

## `foreignDep`

```nim
proc foreignDep*(foreignPackageName: string)
```

### What it does

The main high-level API for registering a foreign (non-Nim) package dependency. Given just a package name, it:

1. Calls `foreignDepInstallCmd(foreignPackageName)` to determine the correct install command and whether sudo is needed.
2. Passes both to `foreignCmd`, which prepends `"sudo "` if needed and appends the result to `foreignDeps`.

You never need to know the current distribution when calling `foreignDep` — the detection happens automatically inside `foreignDepInstallCmd`.

### When to use

`foreignDep` is the go-to choice for the vast majority of cases: you know the package name for each distribution, you call `foreignDep` inside the appropriate `detectOs` branch, and the module handles the rest.

The one thing `foreignDep` cannot do for you is handle distributions that give the same library different package names — you must call `foreignDep` with the right name inside the correct `detectOs` branch yourself.

### Examples

```nim
import distros

# Example 1 — basic usage in a Nimble script
if detectOs(Ubuntu) or detectOs(Debian):
  foreignDep "libblas-dev"
  foreignDep "liblapack-dev"
elif detectOs(Fedora) or detectOs(RedHat):
  foreignDep "blas-devel"
  foreignDep "lapack-devel"
elif detectOs(ArchLinux):
  foreignDep "blas"
  foreignDep "lapack"

# Example 2 — NixOS does not need sudo, but name is different
if detectOs(NixOS):
  foreignDep "openblas"
else:
  foreignDep "libopenblas-dev"

# Example 3 — after registration, inspect what was collected
foreignDep "zlib"
for cmd in foreignDeps:
  echo cmd
# On Ubuntu prints: sudo apt-get install zlib
```

---

## `echoForeignDeps`

```nim
proc echoForeignDeps*()
```

### What it does

Iterates over all commands stored in `foreignDeps` and prints each one to standard output with `echo`. Nothing more, nothing less.

This is the standard way to display the accumulated dependency list at the end of a Nimble install task, producing user-facing output like:

```
sudo apt-get install libblas-dev
sudo apt-get install libvoodoo
```

### When to use

Call `echoForeignDeps` at the end of your Nimble `before install` or `after install` hook, or at the end of a `config.nims` task, after all `foreignDep` / `foreignCmd` calls have been made. There is no point calling it before all dependencies have been registered.

### Example

```nim
import distros

# Register dependencies based on the current OS
if detectOs(Ubuntu) or detectOs(Debian):
  foreignDep "libssl-dev"
  foreignDep "libffi-dev"
elif detectOs(Fedora):
  foreignDep "openssl-devel"
  foreignDep "libffi-devel"
elif detectOs(ArchLinux):
  foreignDep "openssl"
  foreignDep "libffi"
elif detectOs(MacOSX):
  foreignDep "openssl"
  foreignDep "libffi"
elif detectOs(Windows):
  foreignDep "openssl"
  foreignDep "libffi"

echo "To complete the installation, run:"
echoForeignDeps()
# Example output on Ubuntu:
# To complete the installation, run:
# sudo apt-get install libssl-dev
# sudo apt-get install libffi-dev
```

---

## Full worked example — Nimble build script

```nim
# In your package's config.nims or .nimble file:

import distros

task checkDeps, "Check and display required system dependencies":

  # ── OpenSSL ────────────────────────────────────────────────────────────────
  if detectOs(Ubuntu) or detectOs(Debian) or detectOs(Elementary):
    foreignDep "libssl-dev"
  elif detectOs(Fedora) or detectOs(RedHat) or detectOs(CentOS):
    foreignDep "openssl-devel"
  elif detectOs(ArchLinux) or detectOs(Manjaro) or detectOs(Artix):
    foreignDep "openssl"
  elif detectOs(Alpine):
    foreignDep "openssl-dev"
  elif detectOs(NixOS):
    foreignDep "openssl"          # nix-env -i, no sudo
  elif detectOs(MacOSX):
    foreignDep "openssl"          # brew, no sudo
  elif detectOs(Windows):
    foreignDep "openssl"          # choco, no sudo

  # ── SQLite ─────────────────────────────────────────────────────────────────
  if detectOs(Ubuntu) or detectOs(Debian):
    foreignDep "libsqlite3-dev"
  elif detectOs(Fedora):
    foreignDep "sqlite-devel"
  elif detectOs(ArchLinux):
    foreignDep "sqlite"
  elif detectOs(Alpine):
    foreignDep "sqlite-dev"
  elif not detectOs(Windows):     # Windows bundles SQLite; others use brew/pkg
    foreignDep "sqlite"

  # ── Custom build step ──────────────────────────────────────────────────────
  if detectOs(Linux) and not (detectOs(NixOS)):
    foreignCmd "ldconfig", requiresSudo = true

  # ── Display ────────────────────────────────────────────────────────────────
  if foreignDeps.len > 0:
    echo "\nTo complete the installation, run the following commands:"
    echoForeignDeps()
  else:
    echo "No additional system dependencies required."
```

---

## Design notes and caveats

**Shell commands are run lazily and cached.** The four detection commands (`uname`, `cat /etc/os-release`, `lsb_release`, `hostnamectl`) are only executed when first needed and their output is stored in module-level string variables. Subsequent `detectOs` calls for any distribution on the same source reuse the cached string. This makes calling `detectOs` multiple times cheap.

**NimScript compatibility.** The module works both in compiled Nim programs and in NimScript (`config.nims`). In NimScript, `gorge` is used instead of `execProcess` to run shell commands.

**Package names are your responsibility.** `foreignDep` accepts whatever string you give it. If you pass the wrong package name for a distribution, the generated command will be syntactically correct but semantically wrong. Always test on the actual target distribution.

**`foreignDeps` is not defined under Nimble.** When the module is used from within a Nimble script (`defined(nimble)` is true), `foreignDeps` is Nimble's own variable from `nimscriptapi`. Accessing the module-level `foreignDeps` directly from a Nimble script may behave differently than expected.

**Unknown distributions fall through to a placeholder.** If none of the known Linux distributions are matched by `foreignDepInstallCmd`, the returned command is literally `"<your package manager here> install <pkg>"`. This serves as a visible reminder to the user that they need to find the right command themselves.
