# datafile-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

Where a program's configuration, data, cache, state, runtime and log
directories live — on Linux, on macOS and on Windows.  A novo-lang port
of Rust's [directories](https://docs.rs/directories) and Python's
[platformdirs](https://platformdirs.readthedocs.io), cut to the layer:
the environment is read **once**, into a value, and every directory is
arithmetic over it.

- `dfplat` — the two tables.  Which environment variable overrides a
  role on a platform, and where the role goes when nothing does.
- `dfenv` — `DfEnv`, the snapshot, and the only `[io]` a program has to
  perform.
- `dfbase` — the base directories, `[]` over a `DfEnv`.
- `dfapp` — `DfApp` and the project path, which is spelled differently
  on all three platforms.
- `dfuser` — Documents, Downloads, Pictures, and the one file this
  package reads.
- `dfmake` — the `[fs]` half: create the directory, and verify the mode
  when a secret is going into it.
- `dfnovo` — the novo toolchain's own layout, as a worked example.

```
novo pkg add datafile-nv
novo pkg build
novo test
```

## The one example that will work

```novo ignore
use dfenv
use dfapp
use dfmake

fn main() [io, fs]
    // One read of the environment.  Everything after this is arithmetic.
    let e = dfenv.capture()
    let me = dfapp.qualified("org", "novo-lang", "novo")

    match dfapp.check(me)
        Some(f) => println(f.message())
        None    =>
            match dfmake.ensure_app_dir(e, me, DfCache, DfModeDefault)
                Err(f) => println(f.message())
                Ok(dir) =>
                    println(dir)
                    println(dfapp.config_file(e, me, "config.toml"))
```

On Linux with an ordinary home directory that prints
`/home/ada/.cache/novo` and `/home/ada/.config/novo/config.toml`; on
macOS `/Users/ada/Library/Caches/org.novo-lang.novo` and
`/Users/ada/Library/Application Support/org.novo-lang.novo/config.toml`;
on Windows `C:\Users\ada\AppData\Local\novo-lang\novo` and
`C:\Users\ada\AppData\Roaming\novo-lang\novo\config.toml`.

## The load-bearing interface: `DfEnv`, the environment read once

```novo ignore
pub struct DfEnv
    platform: dfplat.DfPlatform
    home: Str
    values: [(Str, Str)]

pub fn capture() -> DfEnv [io]
pub fn of(p: dfplat.DfPlatform, home: Str, values: [(Str, Str)]) -> DfEnv []
```

`capture` is the whole `[io]` surface a program has to touch.  Every
directory — `dfbase.config_dir`, `dfapp.cache_file`, `dfnovo.bin` — is
`[]` arithmetic over the value it returns.  Three things follow, and
they are why the design is worth the extra call.

**The platform table is testable on any machine.**  `of(DfWindows,
"C:/Users/ada", pairs)` builds an environment no machine in this
project has, and `tests/dfbase_tests.nv` then asserts the Windows
column — cell by cell, on every commit, on a Linux runner.  `directories`
and `platformdirs` both read the environment *inside* each accessor, so
their macOS column is checked on the day somebody has a Mac and their
Windows column is checked by their users.  That is the difference this
package exists to make, and it is worth one line at startup.

**The answers are consistent.**  A program that calls `config_dir` and
then `cache_dir` against a live environment can be handed answers from
two different environments — a `setenv` between them, a plugin that
changed a variable.  One snapshot makes that impossible, and it makes
the whole layout one value a program can log, print, or hand to a
subprocess.

**The cost is one pass.**  `capture` reads at most eleven variables plus
the home directory and the platform name.  The per-accessor shape reads
one or two variables *per call*, so a program with twelve accessors
reads the environment twelve times.

### The second decision: `""` means "there is no such directory"

Nothing in the read path returns a `Result` or an optional.  Every
accessor answers a `Str`, and `""` means the directory does not exist as
a concept here.  `dfbase.source_of` says which of six reasons applies:

| source | what happened |
| --- | --- |
| `DfFromOverride(name)` | a variable set it, and it is named |
| `DfFromDefault` | the platform's default under the home directory |
| `DfOverrideRejected(name)` | a variable set it and was **ignored** — it is not absolute |
| `DfNoHome` | no home directory to hang a default off |
| `DfNoDefault` | the platform carries the role but defines no fallback (`XDG_RUNTIME_DIR`) |
| `DfRoleAbsent` | this platform has no such directory (runtime on macOS) |

The standard library set the precedent: `env.home()` answers `""` rather
than `?Str`, "because there is nothing a caller does differently about
it".  A package where twelve accessors each returned an optional pushes
a `??` into every call site and each one spells the fallback
differently.  A package that answered a *relative* path instead is the
defect where a service account with no `HOME` writes its configuration
into whatever directory it was started in.

### Three rules almost nothing implements

**A relative override is ignored.**  The XDG specification says so
outright, and `XDG_CONFIG_HOME=.config` is a thing people type into a
shell profile.  Honouring it puts a program's configuration under
whatever directory it was started in, and the configuration vanishes the
next time it starts somewhere else.  `dfenv.override_is_usable` is that
rule, `DfOverrideRejected` reports it, and it is checked per platform —
`/srv/data` is absolute on Linux and is **not** absolute on Windows,
where it names no drive.

**An empty variable is unset.**  `XDG_DATA_HOME=` is the specification's
"unset", not an empty path.

**`XDG_RUNTIME_DIR` has no fallback.**  The specification gives none, and
the two obvious inventions are both wrong: `/tmp` is world-writable, so
another user can replace the socket; the cache directory is not emptied
at logout, so the pid file outlives the session.  This answers `""` and
`DfNoDefault`, and a program that needs a socket chooses in its own
code, where the decision is visible.

### The project path is the fact a naive port gets wrong

Given the qualifier `org`, the organization `novo-lang` and the
application `novo`:

| platform | project path |
| --- | --- |
| Linux | `novo` |
| macOS | `org.novo-lang.novo` |
| Windows | `novo-lang\novo` |

A port that used the application name on all three writes a macOS
directory the platform's own tools do not recognise, and a Windows
directory that collides with any other vendor's program of the same
name.  `dfapp.project_path` is that table, it is `[]`, and all three
rows are asserted in `tests/dfapp_tests.nv`.

Names are **checked and sanitised**, both.  `dfapp.check` refuses a name
with a separator, a `.` or a `..` in it, and a program calls it once at
startup where the message can name the manifest field that is wrong.
The joins sanitise anyway, so the failure mode of skipping `check` is a
strange directory name and never an escape.  One without the other is a
guard that gets bypassed, or a silence.

### `dfmake` verifies the mode it asked for

`DfModePrivate` does not merely request `0700`; it creates the directory
and then **reads back** the mode, answering `DfModeTooWide(path, wanted,
found)` when a umask or an inherited parent produced something wider.
A caller that wrote a token into a directory it believed was private has
no way to discover otherwise afterwards, so the check belongs here.

On Windows there are no POSIX mode bits and this package does not read
ACLs, so `DfModePrivate` is `DfModeUnsupported` — a refusal rather than
a silent success, because telling a credential store its secret is
protected on the one platform where it is not is the worst of the
available answers.

Nothing in this package deletes.  A directories library that could
remove a tree is a directories library that could remove the wrong tree;
`tempdir-nv` owns removal, behind a root value that says what it is
entitled to delete.

## The worked example: the novo toolchain's own layout

`novo` does **not** follow XDG, and `dfnovo` is the honest example a
directories library ought to carry — one that shows the rule and shows a
program that does not follow it, with the reason written down.

```
$NOVO_HOME  (default ~/.novo)
├── bin                → current/bin          on PATH
├── current            → versions/<version>
├── versions/<version>                        one unpacked release each
├── lib                                       runtime sources
└── cache
    ├── registry/<host>/<name>/<version>      fetched package sources
    └── rt-obj                                compiled runtime objects
```

Everything lives under one root because a person must be able to move,
copy or delete an installation by moving, copying or deleting one
directory — and because the installer's relative symbolic links stop
being relative the moment the tree is split across `~/.config`,
`~/.cache` and `~/.local/share`.

`dfnovo.xdg_would_be` answers where each part *would* live under the
conventions, so the decision not to is visible and checkable rather than
implicit, and so the table exists on the day the toolchain moves.

## The layer, and why

`host`, and two effects only.

`[io]` is the **environment** — `env.os`, `env.home`, `env.get`.  SPEC
§ 5.1 puts the environment under `[io]` alongside pipes and processes;
there is no `[env]` label to declare, and a package that wanted one
would still be charged `[io]`.  It appears on exactly three functions:
`dfplat.current_platform`, `dfenv.capture` and `dfenv.capture_for`.

`[fs]` is the disk, and it appears in exactly two places: the three
functions in `dfuser` that read `user-dirs.dirs`, and `dfmake`.

Everything else — `dfplat`, `dfbase`, `dfapp`, `dfnovo`, and the parse
and default halves of `dfuser` — is `[]`.  That is not a technicality:
it is the reason the whole platform table can be asserted from a test
with no environment, no home directory and no disk.

No dependencies.  A package that answers "where does my configuration
live" is depended on by configuration readers, loggers and installers
alike, and every dependency it took would be one they all pay for.

## What it does not do

- **No Windows registry.**  The user directories (Documents, Downloads)
  are stored in the registry there, and this package answers the default
  under `%USERPROFILE%` instead.  `dfuser.default_user_dir` is the
  honest name for what it is doing.  A `winreg-nv` row is what would
  fix it.
- **No `~` expansion and no `realpath`.**  Paths are returned as the
  environment gave them.  `/tmp` is a symbolic link to `/private/tmp` on
  macOS, and a package that resolved one path and not another would hand
  a caller two strings that name the same directory and do not compare
  equal.
- **No `UserDirs` for other users.**  Every answer is about the process's
  own environment.
- **No deletion, no writing of files** — only directory creation.

## The reference implementation

[directories](https://docs.rs/directories) 5.x for the role set, the
project path and the platform tables; `directories`' own
`BaseDirs`/`ProjectDirs` split is where `dfbase` and `dfapp` come from.
[platformdirs](https://platformdirs.readthedocs.io) for the version
component, the log directory and the site-wide directories, which
`directories` does not carry.  The [XDG Base Directory
Specification](https://specifications.freedesktop.org/basedir-spec/) is
the authority for the Linux column, including the two rules above that
both references skip.

## Status

Every body is `todo()`.  The interface is **122 `pub` items across seven
modules** — 3 structs, 6 enums and 113 functions, plus one `impl Error`:

| module | surface |
| --- | --- |
| `dfplat` | 3 enums, 14 functions |
| `dfenv` | 1 struct, 1 enum, 16 functions, `impl Error for DfFault` |
| `dfbase` | 19 functions, all `[]` |
| `dfapp` | 1 struct, 24 functions |
| `dfuser` | 1 enum, 11 functions |
| `dfmake` | 1 enum, 14 functions |
| `dfnovo` | 1 struct, 15 functions |

Effect rows across the whole package: `[io]` on three functions, `[fs]`
on sixteen, and `[]` on the other ninety-four.
