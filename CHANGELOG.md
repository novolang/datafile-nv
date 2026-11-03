# Changelog

All notable changes to datafile-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-10-11

The **implementation**.  Every `pub fn` under `src/` has a body, the
seven suites are green, and the interface release's stamps are gone.
`stability = "experimental"`.

### Added

- `tests/datafile_edge_tests.nv` — ten tests over the arms and the
  refusals the first six suites did not reach: every role name, a
  source that names its own variable, the Windows and macOS columns of
  the site-directory lists, the mode text, the faults `dfmake` answers
  when what is at a path is a file rather than a directory, and a
  directory that already exists with a wider mode being tightened
  rather than believed.

### Changed

- **Twenty-two `== false` comparisons across the suites are spelled
  `not`.**  The 0.20.2 linter raises `redundant-bool` on each and
  suggests exactly that rewrite, which is the same predicate.  All
  twenty-two are pre-existing — the released copies carry the same
  counts, `dfapp` 8, `dfplat` 5, `dfnovo` 4, `dfmake` 3 and `dfbase` 2
  — and the assertions' meanings are untouched.
- Every module under `src/`: the bodies, with no signature and no
  effect row changed.  A doc sentence that had become false is gone
  from four of them — `dfmake.octal` no longer claims novo-lang has no
  octal integer literal, `dfbase.executable_dir` no longer names
  `$XDG_BIN_HOME`, and `dfenv` and every module header no longer say
  the bodies panic.  `dfplat`, `dfenv` and `dfbase` were reformatted by
  `novo fmt`.
- The mode half is built on `std.fs.file_mode` and
  `std.fs.set_file_mode`, and the sources spell a mode as `0o700`.  The
  two 0.0.1 rows that said those primitives did not exist are closed by
  the toolchain; nothing in this package works around them.
- Four `// cov: skip` markers, one for each line no suite can run: the
  three refusals in `dfmake.apply_mode` and `default_component`'s
  `DfFonts` arm.  `novo pkg publish` prints each region and its reason.
- README: the interface-release status block and the
  implementation-status table are gone, the test section lists the
  seventh suite, and its claim that no test reads the environment or a
  disk now names the two suites that do and the directory they use.
- `novo = ">= 0.20.2"`, the version whose `0o` literals and mode
  primitives the implementation needs.

### Known

- **`NOVO_HOME` is not in `variable_names`.**  The list is the seven
  XDG variables and `HOME` on Linux, `HOME` on macOS and the four
  Windows variables, and `capture` reads exactly it.  A captured
  snapshot therefore never carries `NOVO_HOME`, and
  `dfnovo.layout(dfenv.capture())` answers `~/.novo` however the
  variable is set in the process.  Measured: with `NOVO_HOME=/opt/novo`
  exported, `dfenv.variable(e, "NOVO_HOME")` is `""`.  A caller that
  wants the variable honoured builds the snapshot with
  `dfenv.with_variable(e, dfnovo.home_variable(), path)`, which is what
  `tests/dfnovo_tests.nv` does.  Widening the list is a change to
  `capture`'s documented environmental footprint and is held for a
  later release.
- **The layout's own parts are joined with `/`.**  `DfNovoLayout`
  carries no platform, so `dfnovo.bin` and its neighbours append a part
  with a forward slash whatever the platform was, and a Windows root
  reads `C:\Users\ada\.novo/bin`.  Windows accepts it; a caller that
  wants the platform's separator on every component must not use this
  module.
- **The Windows system font directory is written out as
  `C:\Windows\Fonts`**, because no variable this package reads names
  the system root.  `%WINDIR%` would be the honest source and is not
  read.
- **Four lines of `src/` are not reached by any suite** — 606 of 610
  merged lines, 99.3%.  The publish prints 99% with four excused
  regions.  Three are `apply_mode`'s refusals — a platform that will
  not report a mode for a directory this process owns, a filesystem
  that will not take one, and a filesystem that takes the call and
  keeps other bits than the ones asked for.  A directory this process
  owns on a Linux filesystem always reports a mode and always takes
  one, so reaching the first two from a test would mean changing a
  directory the package does not own, which a suite run as root would
  really do, and the third needs a filesystem that takes the call and
  lies.  The fourth is `default_component`'s `DfFonts` arm, which
  exists for the match to be exhaustive and cannot be reached:
  `default_user_dir` answers a font directory before the table is
  consulted.

This release requires novo 0.23.0 or later.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `dfplat` — `DfPlatform`, `DfRole` and `DfSource`; the two tables as
  public functions, `override_variable` and `default_suffix`;
  `role_is_absent`, `roles`, `separator`, `join`, `is_absolute`,
  `to_platform`, and the name accessors.
- `dfenv` — `DfEnv`, the snapshot; `capture` and `capture_for` (the
  only `[io]` in the package beside `dfplat.current_platform`); `of`
  and `bare`, the constructors that need no environment;
  `override_is_usable`, `is_set`, `with_variable`, `without_variable`,
  `with_home`, `with_platform`, `variables`, `variable_names`; and
  `DfFault` with `impl Error for DfFault`.
- `dfbase` — `base_dir` and `source_of`, plus the eleven named
  accessors, `layout`, the two site-directory lists and the two search
  paths. Every row `[]`.
- `dfapp` — `DfApp`; `app`, `qualified`, `versioned`, `check`,
  `is_component`; `project_path` (the three-platform table); `app_dir`,
  the ten named accessors, `app_file` and the four named file
  accessors; `config_search_path` and `config_file_candidates`.
- `dfuser` — `DfUserDir`; `parse_user_dirs` and `user_dir_from` as
  `[]`; `user_dir`, `all_user_dirs` and `read_user_dirs` as `[fs]`;
  `default_user_dir`, `user_dirs_path`, `xdg_key` and `font_dirs`.
- `dfmake` — `DfMode`; `mode_bits`, `octal`, `mode_octal`,
  `mode_is_private`; `ensure_dir`, `ensure_app_dir`, `ensure_app_dirs`,
  `ensure_parent`; `dir_exists`, `dir_mode`, `dir_is_private`,
  `make_private`, `first_existing`, `existing`.
- `dfnovo` — `DfNovoLayout` and the toolchain's own tree: `root`,
  `bin`, `current`, `versions`, `version_dir`, `lib`, `cache`,
  `registry_cache`, `package_dir`, `runtime_object_cache`, `tree`, and
  `xdg_would_be`.

### Known

- **The environment is read once, into a value.** `dfenv.capture` is the
  whole `[io]` surface and every directory is `[]` arithmetic over its
  result. That is what makes the macOS and Windows columns testable on a
  Linux runner, and `tests/dfbase_tests.nv` asserts both of them cell by
  cell.
- **There is no `[env]` effect.** SPEC § 5.1 puts the environment under
  `[io]`, so reading `XDG_CONFIG_HOME` and writing to a pipe carry the
  same label. The manifest says so rather than inventing a narrower
  claim.
- **`""` means "there is no such directory", and `source_of` says why.**
  Six reasons, including the two the references skip: an override
  ignored for being relative, and a role the platform defines with no
  fallback.
- **A relative override is ignored**, which is the XDG specification's
  own rule and is what stops `XDG_CONFIG_HOME=.config` following a
  person's working directory. Absoluteness is the *platform's* rule:
  `/srv/data` is not absolute on Windows.
- **`std.fs` has no mode, no `chmod` and no `stat`**, so `dfmake`'s
  whole mode half — `DfModePrivate`, `dir_mode`, `dir_is_private`,
  `make_private` — needs primitives that do not exist yet. Recorded
  here and reported upstream against the toolchain rather than worked
  around.
- **novo-lang has no octal integer literal**, so `dfmake.octal(text)`
  is on the public surface: without it a caller cannot write `0700` in
  a way a reviewer can check against `chmod`.
- **No Windows registry**, so the user directories there are the
  defaults under `%USERPROFILE%` rather than the registry's values. A
  `winreg-nv` row is what would fix it.
- **No dependencies**, and no device claim: this package is `[io]` and
  `[fs]` by its nature and has nothing to say at `@tier(embedded)`.
