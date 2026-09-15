# Changelog

All notable changes to datafile-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
