# datafile-nv

Every operating system has a place a program's configuration belongs, a
place its cache belongs, and several more. This package answers where
those directories are on Linux, on macOS and on Windows. It is a
novo-lang port of Rust's [directories](https://docs.rs/directories) and
Python's [platformdirs](https://platformdirs.readthedocs.io), and the
[XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/)
is the authority for the Linux answers.

## What it is

A **role** is what a directory is for, rather than what a platform
calls it. The platforms disagree about the words and agree about the
roles: `AppData\Roaming` on Windows and `$XDG_CONFIG_HOME` on Linux are
both the place configuration goes.

| Role | What goes in it |
| --- | --- |
| `DfConfig` | Configuration a person edits. It roams between machines on Windows. |
| `DfConfigLocal` | Configuration that must not roam: a path, a machine identifier, a socket name. |
| `DfData` | Data the program produced and wants kept. |
| `DfDataLocal` | Data that must not roam: an index, a compiled artifact. |
| `DfCache` | Data the program can recompute. Anything may delete it. |
| `DfState` | State between runs: a history file, a last-session record. |
| `DfRuntime` | Sockets and pid files, emptied at logout. Linux only. |
| `DfLog` | Log files. |
| `DfPreference` | macOS's `Library/Preferences`, which is not its application-support directory. |
| `DfExecutable` | Where a user-installed program goes. Linux only. |

Two tables decide every answer. The **override table** says which
environment variable sets a role on a platform. The **default table**
says where the role goes under the home directory when no variable
does. `dfplat.override_variable` and `dfplat.default_suffix` are those
two tables; the rows below are the ones the test suite asserts cell by
cell.

On Linux, and on any other system this package does not recognise:

| Role | Variable | Default under the home directory |
| --- | --- | --- |
| Config | `XDG_CONFIG_HOME` | `.config` |
| Config, local | `XDG_CONFIG_HOME` | `.config` |
| Data | `XDG_DATA_HOME` | `.local/share` |
| Cache | `XDG_CACHE_HOME` | `.cache` |
| State | `XDG_STATE_HOME` | `.local/state` |
| Log | `XDG_STATE_HOME` | `.local/state` |
| Runtime | `XDG_RUNTIME_DIR` | none |
| Executable | none | `.local/bin` |

On macOS, where no role has an environment variable of its own:

| Role | Default under the home directory |
| --- | --- |
| Config | `Library/Application Support` |
| Cache | `Library/Caches` |
| Log | `Library/Logs` |
| Preference | `Library/Preferences` |
| Runtime, executable | absent |

On Windows:

| Role | Variable | Default under the profile |
| --- | --- | --- |
| Config | `APPDATA` | `AppData\Roaming` |
| Config, local | `LOCALAPPDATA` | `AppData\Local` |
| Cache | `LOCALAPPDATA` | `AppData\Local` |
| Runtime, executable | absent | absent |

The environment is read **once**, into a `DfEnv` value, by
`dfenv.capture`. That is the only function a program has to call that
touches the machine. Every directory is then a `[]` function over that
value, so the answers are consistent with each other and the whole
layout is one value a program can print, log or hand to a subprocess.

A `DfEnv` can also be built by hand with `dfenv.of`, naming the
platform, the home directory and the variables. A test on a Linux
machine can therefore assert the macOS and Windows columns.

A **project path** is the part of a directory that belongs to one
program. It is spelled differently on each platform, from a
**qualifier**, an **organization** and an **application** name.

| Platform | Project path for `org`, `novo-lang`, `novo` |
| --- | --- |
| Linux | `novo` |
| macOS | `org.novo-lang.novo` |
| Windows | `novo-lang\novo` |

## Install

```
novo pkg add datafile-nv
```

## Example

```novo
use dfenv
use dfapp
use dfmake

fn main() [io, fs]
    // Read the environment once. Every directory below is arithmetic
    // over the value this returns.
    let e = dfenv.capture()

    // Name the program. All three parts are used, because the three
    // platforms spell a project's directory differently.
    let me = dfapp.qualified("org", "novo-lang", "novo")

    match dfapp.check(me)
        // One of the three names is not a usable path component.
        Some(f) => println(dfenv.fault_message(f))
        None    =>
            // Create this program's cache directory and answer its path.
            match dfmake.ensure_app_dir(e, me, DfCache, DfModeDefault)
                Err(f)   => println(dfenv.fault_message(f))
                Ok(dir)  =>
                    println(dir)
                    // Where a configuration file of this name would go.
                    println(dfapp.config_file(e, me, "config.toml"))
```

On Linux with an ordinary home directory that prints
`/home/ada/.cache/novo` and `/home/ada/.config/novo/config.toml`. On
macOS it prints `/Users/ada/Library/Caches/org.novo-lang.novo` and
`/Users/ada/Library/Application Support/org.novo-lang.novo/config.toml`.
On Windows it prints `C:\Users\ada\AppData\Local\novo-lang\novo` and
`C:\Users\ada\AppData\Roaming\novo-lang\novo\config.toml`.

Build and test with `novo pkg build` and `novo test`.

## What the package contains

| Module | Contents |
| --- | --- |
| `dfplat` | The platform, the roles, the two tables, and the path arithmetic each platform needs: its separator, its join and its rule for an absolute path. |
| `dfenv` | `DfEnv`, the environment read once, the builders that make one by hand, and every fault this package can answer. |
| `dfbase` | The directories of the person running the program, as functions over a `DfEnv`. |
| `dfapp` | `DfApp`, the project path, and the same directories with this program's own folder inside them. |
| `dfuser` | Documents, Downloads, Pictures and the other directories a desktop gives a person, and the one file this package reads. |
| `dfmake` | The half that touches the disk: create a directory, check one exists, and verify its mode. |
| `dfnovo` | The novo toolchain's own layout under `NOVO_HOME`, as a worked example of a program that does not follow the conventions. |

## How to choose an entry point

**`dfbase` answers the person's directories.** `dfbase.config_dir` is
`/home/ada/.config`. Use it when you want the directory itself, or when
you are placing something that is not a program's private folder.

**`dfapp` answers this program's directories.** `dfapp.config_dir` is
`/home/ada/.config/novo`, with the project path already appended. Use
it for everything a program owns. `dfapp.config_file` adds a file name
on the end.

**`dfmake` is the only module that touches the disk.** Nothing in
`dfbase` or `dfapp` creates anything; they compute paths.

**`dfplat` is the tables themselves.** A program printing "your
configuration lives here because this variable is set" reads
`dfplat.override_variable` and `dfbase.source_of`.

## The rules a user needs

1. **`dfenv.capture` is the only environment read.** It is one of three
   functions in the package that declares `[io]`, together with
   `dfenv.capture_for` and `dfplat.current_platform`. Everything in
   `dfbase`, `dfapp`, `dfnovo` and the arithmetic half of `dfuser`
   declares nothing at all.
2. **An empty string means there is no such directory.** No read
   answers a `Result` or an optional. `dfbase.source_of` says which of
   six reasons applies, so a caller can tell "your platform has no
   runtime directory" from "you have no home directory".

   | Source | What happened |
   | --- | --- |
   | `DfFromOverride(name)` | A variable set it, and it is named. |
   | `DfFromDefault` | The platform's default under the home directory. |
   | `DfOverrideRejected(name)` | A variable set it and was ignored, because it is not absolute. |
   | `DfNoHome` | No home directory to hang a default off. |
   | `DfNoDefault` | The platform carries the role and defines no fallback. |
   | `DfRoleAbsent` | This platform has no such directory. |

3. **A relative override is ignored.** The XDG Base Directory
   Specification requires it, and `XDG_CONFIG_HOME=.config` is a thing
   people type into a shell profile. Honouring it would put a program's
   configuration under whatever directory it was started in.
   `dfenv.override_is_usable` is the rule and `DfOverrideRejected`
   reports it.
4. **Absoluteness is decided by the platform.** `/srv/data` is absolute
   on Linux and is not absolute on Windows, where it names no drive.
   Windows's absolute forms are `C:\...`, `C:/...` and
   `\\server\share`. `dfplat.is_absolute` is that predicate.
5. **An empty variable is unset.** `XDG_DATA_HOME=` is the
   specification's "unset" and not an empty path. `dfenv.is_set`
   preserves the distinction.
6. **`XDG_RUNTIME_DIR` has no fallback.** The specification gives none.
   `/tmp` is world-writable, so another user could replace the socket,
   and the cache directory is not emptied at logout, so a pid file
   would outlive the session. Unset, `dfbase.runtime_dir` answers the
   empty string and `source_of` answers `DfNoDefault`.
7. **The runtime and executable roles are absent on macOS and
   Windows.** `dfplat.role_is_absent` answers which, and
   `dfmake.ensure_app_dirs` skips a role the platform does not carry
   rather than refusing the whole call.
8. **A log goes in the state directory on Linux and in
   `Library/Logs` on macOS.** It is not the cache, which anything may
   delete.
9. **Check a program's name once, at startup.** `dfapp.check` refuses a
   qualifier, organization or application that is empty, is `.` or
   `..`, or contains a separator. The joins sanitise the name anyway,
   so skipping the check gives a strange directory name and never an
   escape from the parent.
10. **An unrecognised platform reads the XDG column.**
    `dfplat.platform_of` answers `DfOther` for anything that is not
    Linux, macOS or Windows, which is the right guess for every other
    Unix.
11. **`DfModePrivate` is verified and not merely requested.**
    `dfmake.ensure_dir` creates the directory, reads the mode back, and
    answers `DfModeTooWide(path, wanted, found)` when a umask or an
    inherited parent produced something wider.
12. **`DfModePrivate` is refused on Windows.** There are no POSIX mode
    bits there and this package does not read access control lists, so
    it answers `DfModeUnsupported`. Telling a credential store that its
    secret is protected on the one platform where it is not would be
    the worst available answer.
13. **Nothing here deletes anything.** The package creates directories
    and writes no files. A directories library that could remove a tree
    is a directories library that could remove the wrong tree.
14. **Paths come back as the environment gave them.** There is no `~`
    expansion and no resolution of symbolic links. `/tmp` is a link to
    `/private/tmp` on macOS, and resolving one path and not another
    would hand a caller two strings that name one directory and do not
    compare equal.
15. **`dfbase.config_search_path` is the list a layered configuration
    reader wants**, the person's directory first and the site's after
    it. `$XDG_CONFIG_DIRS` supplies the site's, split on `:`, and
    `/etc/xdg` is the default.

16. **The novo toolchain does not follow these conventions, and
    `dfnovo` is that layout.** Everything lives under one root,
    `$NOVO_HOME`, which defaults to `~/.novo`, so that an installation
    can be moved, copied or deleted by moving, copying or deleting one
    directory.

    | Part | Path under the root |
    | --- | --- |
    | Executables on the PATH | `bin`, a link to `current/bin` |
    | The selected release | `current`, a link to `versions/<version>` |
    | Every unpacked release | `versions/<version>` |
    | Runtime sources | `lib` |
    | Fetched package sources | `cache/registry/<host>/<name>/<version>` |
    | Compiled runtime objects | `cache/rt-obj` |

    `dfnovo.xdg_would_be` answers where each part would live under the
    conventions, so the decision not to follow them is checkable rather
    than implicit.

## What is not included

- **The Windows registry.** Documents and Downloads are stored there,
  and this package answers the default under `%USERPROFILE%` instead.
  `dfuser.default_user_dir` is the honest name for what it does.
- **Other users' directories.** Every answer is about the environment
  of the process asking.
- **Deleting anything, and writing any file.** See rule 13. Only
  directory creation is here.
- **Path resolution.** See rule 14.
- **A runtime directory where the platform has none.** See rule 6.
- **A dependency of any kind.** A package that answers where
  configuration lives is depended on by configuration readers, loggers
  and installers alike, and every dependency it took would be one they
  all pay for.

## Related packages

- [config-nv](https://novo-lang.org/packages/config-nv) stacks
  configuration from several sources in precedence order. It takes a
  list of directories, which is what `dfbase.config_search_path`
  answers. This package says where the files are; that one decides
  which value wins.
- [toml-nv](https://novo-lang.org/packages/toml-nv),
  [yaml-nv](https://novo-lang.org/packages/yaml-nv),
  [ini-nv](https://novo-lang.org/packages/ini-nv) and
  [dotenv-nv](https://novo-lang.org/packages/dotenv-nv) read the files
  that live in these directories.
- `std.env` in the standard library answers one variable, the home
  directory and the operating system name. It is what `dfenv.capture`
  calls. `std.fs` is what `dfmake` calls.

## Tests

```bash
novo test tests/dfplat_tests.nv        # 11 tests: the two tables, the joins, absoluteness
novo test tests/dfbase_tests.nv        # 12 tests: the three platform columns
novo test tests/dfapp_tests.nv         # 11 tests: the project path and the name rules
novo test tests/dfuser_tests.nv        # 12 tests: user-dirs.dirs and the defaults
novo test tests/dfmake_tests.nv        # 12 tests: the modes and the environment value
novo test tests/dfnovo_tests.nv        # 10 tests: the toolchain layout and the mode arithmetic
novo test tests/datafile_edge_tests.nv # 10 tests: every table arm, and the refusals a disk produces
```

The expected paths are `directories` 5.x's platform tables, the XDG
Base Directory Specification for the Linux column, and `platformdirs`
for the log directory, the version component and the site-wide
directories.

Almost no test reads the environment, a home directory or a disk.
Every assertion about a platform is made against a `DfEnv` built with
`dfenv.of`, so the macOS column and the Windows column are asserted
cell by cell on a Linux machine. The suites check that a relative
override is rejected and named, that an empty variable is unset, that
the runtime directory has no invented fallback, that a join produces
exactly one separator and never a leading one, that a Windows path is
absolute only with a drive or a share, that the project path is
spelled three different ways, and that a name with a separator in it is
refused.

The exceptions are the two suites that check the disk, which work
under `/tmp/novo-datafile-nv-tests` and read the machine's own
environment once to assert the shape of what comes back rather than
its values. Nothing is ever deleted: this package cannot delete, on
purpose.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
