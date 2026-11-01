# ZuoKit

A single-file toolkit for [Zuo](https://docs.racket-lang.org/zuo/index.html) build scripts, covering commands, dependencies, builds, tests, and benchmarks.

## Features

- **Commands:** Run processes with arguments, environment overrides, and output capture (`run`, `cmd`).
- **Dependencies:** Manage Git repositories, SHA-256-verified archives, and local sources with caching, patches, and a lockfile (`deps`, `deps-sync`, `deps-upgrade`).
- **Builds:** Define incremental steps and integrate CMake or Chez Scheme projects (`step`, `cmake-project`, `chez-project`).
- **Testing:** Run isolated test and benchmark suites with machine-readable reports (`test-suite`, `bench-suite`).

## Usage

Install Zuo and put `zuokit.zuo` in your project. Create `main.zuo`:

```scheme
#lang zuo
(require "zuokit.zuo")
```

External tools such as Git, curl, bsdtar, CMake, and Chez Scheme are required only for features that invoke them.

### Commands

`run` executes a command, `cmd` constructs a reusable command, and `run/cmd` executes a constructed command.

```scheme
(run option value ... program arg ...)
(cmd option value ... program arg ...)
(run/cmd option value ... command)
```

Command arguments support a shell-like syntax:

- `word`, `"text"`, `123`: literal arguments.
- `$x`, `($ expr)`: interpolate a value as one argument.
- `@xs`, `(@ expr)`: splice a list of arguments.

For example:

```scheme
(define flags '("-O2" "-Wall"))
(define source "main.c")

(run gcc @flags -c $source -o main.o)

(define compile (cmd gcc @flags -c $source -o main.o))
(run/cmd compile)
```

Options precede the executable:

| Option | Description |
| --- | --- |
| `'dir` | Working directory |
| `'env` | Add, override, or remove environment variables |
| `'env!` | Replace the environment |
| `'stdin`, `'stdout`, `'stderr` | Stream redirection |
| `'check?` | Raise an error on nonzero exit (default `#t`) |
| `'thread?` | Wait using thread-aware process waiting |
| `'capture?` | Capture stdout and stderr |
| `'directory` | Temporary directory parent for captured output |
| `'with` | Merge a hash of process options |

```scheme
(run 'dir "src" 'env (hash 'CC "clang") make)
(run 'capture? #t echo hello) ; => (hash 'exit 0 'stdout "hello\n" 'stderr "")
(run 'check? #f false)        ; => nonzero exit status
```

Without capture, `run` returns an exit status. With capture, it returns a hash containing `exit`, `stdout`, and `stderr`.

### Dependencies

The `deps` syntax declares named dependencies:

```scheme
(deps [name source]
  ...)
```

Three source constructors are available:

```scheme
(git url 'commit commit)
(git url 'ref branch)
(archive url 'sha256 digest 'root 'single)
(local path)
```

Example:

```scheme
(define dependencies
  (deps
    [foo (git "https://example.org/foo.git" 'ref "main")]
    [bar (archive "https://example.org/bar.tar.gz" 'sha256 "<sha256>" 'root 'single)]
    [baz (local "vendor/baz")]))
```

Source options:

- `git`: `'commit` or `'ref` (defaults to `"HEAD"`).
- `archive`: `'sha256` and optional `'root` (`'keep`, `'single`, or a relative subdirectory).
- All sources: `'patches`, a list of patch files applied with `git apply`.

Dependency operations:

```scheme
(deps-sync dependencies "build/_deps")
(deps-upgrade dependencies "build/_deps")
(deps-upgrade dependencies "build/_deps" 'names '(foo))
(deps-targets dependencies "build/_deps")
```

- `deps-sync`: resolve and materialize dependencies, returning a hash of paths.
- `deps-upgrade`: update locked Git revisions.
- `deps-targets`: create Zuo build targets for dependencies.

Dependencies support caching, lockfiles, SHA-256 verification, and patches. Common options include `'base`, `'cache`, and `'lock`. `deps-sync` and `deps-upgrade` also accept `'thread?`.

### Incremental Builds

`step` defines a Zuo build target:

```scheme
(step output inputs action option value ...)
```

- `output`: a file path, a list of co-outputs, or a symbol for a phony target.
- `inputs`: a list of file paths or targets, or a thunk returning such a list.
- `action`: a `cmd`, a thunk, or a list of actions.
- Options: `'config`, `'needs`, `'db-dir`, `'precious?`.

```scheme
(define object
  (step "build/main.o"
        '("main.c")
        (cmd cc -c main.c -o build/main.o)
        'config (hash 'mode 'release)))

(step 'all (list object) (lambda () (void)))
```

Project helpers:

```scheme
(cmake-project source build-dir
  'generator "Ninja"
  'configuration "Release"
  'define (hash 'BUILD_TESTING #f)
  'jobs 4)

(chez-project sources build-dir
  'scheme "scheme"
  'optimize-level 2)
```

`cmake-project` provides configure/build steps and installs only when `'prefix` is set. `chez-project` compiles Chez Scheme libraries, scanning `.sls`, `.ss`, `.scm`, and `.sch` sources.

Both integrate with incremental build targets and accept additional configuration and dependencies.

### Tests and Benchmarks

Suites are declared using `test-suite` and `bench-suite`:

```scheme
(test-suite
  [name action option value ...]
  ...
  option value ...)

(bench-suite
  [name action option value ...]
  ...
  option value ...)
```

Suites can be nested. Each case runs in a separate worker process.

#### Tests

A test action can be a thunk, a `cmd`, or a `diff`.

```scheme
(define tests
  (test-suite
    [smoke (cmd echo hello) 'tags '(fast)]
    [compare (diff (cmd echo hello) (cmd echo hello))
      'needs '("smoke")]
    [unit (test-suite [simple (lambda () (void))])]))
```

Test helpers:

```scheme
(diff expected-cmd actual-cmd 'compare comparator)
(defer (lambda () action))
```

- `diff` compares command stdout, or uses a custom comparator that receives two output paths and returns a command.
- `defer` prepares an action inside the worker process.

Test options include `'exit` (expected exit status) and `'skip-exit` (skip status).

#### Benchmarks

A benchmark action can be a thunk, `batch`, or `measured`.

```scheme
(define benchmarks
  (bench-suite
    [hash
      (lambda () (string-sha256 "hello"))
      'target-ms 50 'warmup 3 'samples 15]))
```

Additional measurement forms:

```scheme
(batch (lambda (n) nanoseconds) 'clock 'reported)
(measured (lambda (n result-path) command) 'clock 'reported)
```

- `batch` measures `n` operations and directly returns their total duration in nanoseconds.
- `measured` constructs a command that writes the batch duration (an S-expression integer in nanoseconds) to `result-path`.
- Ordinary thunks use wall-clock timing.

Benchmark options:

| Option | Default | Description |
| --- | --- | --- |
| `'target-ms` | `50` | Target batch duration |
| `'warmup` | `3` | Warmup samples |
| `'samples` | `15` | Recorded samples |
| `'max-n` | `1073741824` | Maximum batch size |

Reports include calibration data, samples, minimum, median, and p90 nanoseconds per operation.

#### Suite options

Both suite forms support:

- `'tags`: tags for filtering cases.
- `'needs`: prerequisite cases or nested suites.
- `'defaults`: inherited child defaults (suite level only).

The functional `suite` API is also available:

```scheme
(suite
  (list (list "smoke" (cmd echo ok) 'tags '(fast)))
  'tags '(ci))
```

### Running Suites

A manifest exports `tests` and/or `benchmarks`:

```scheme
#lang zuo

(require "zuokit.zuo")
(provide tests benchmarks)

(define tests
  (test-suite [smoke (cmd echo hello)]))

(define benchmarks
  (bench-suite [hash (lambda () (string-sha256 "hello"))]))
```

Run from the command line:

```sh
zuo zuokit.zuo test checks.zuo
zuo zuokit.zuo test checks.zuo smoke
zuo zuokit.zuo bench checks.zuo
```

For a dependency manifest exporting `dependencies`:

```sh
zuo zuokit.zuo upgrade deps.zuo
zuo zuokit.zuo upgrade deps.zuo foo
```

Programmatic APIs:

```scheme
(test-run "checks.zuo" 'names '("smoke") 'tags '(fast))
(bench-run "checks.zuo")
(report (test-run "checks.zuo"))
```

`test-run` and `bench-run` return structured results. They support `'export`, `'names`, `'tags`, `'directory`, `'config`, and `'supervise`.

A manifest may export a suite directly or a procedure that accepts the run configuration.

`suite-targets` integrates suites into Zuo's build system:

```scheme
(suite-targets "checks.zuo" 'mode 'test)
(suite-targets "checks.zuo" 'mode 'bench)
```

It supports per-case targets and custom groups.

Reports are stored in per-run directories under `build/zuokit/`, including a machine-readable `report.sexp` and worker output files.
