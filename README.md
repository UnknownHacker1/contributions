# Open-Source Contributions

Real fixes I've landed (or am landing) in tools I actually use. Smallest common
thread: find a genuine correctness bug in a large unfamiliar codebase, trace it,
and fix it the way the maintainers would.

| # | Project | Language | What I fixed | PR | Status |
| :-: | --- | --- | --- | :-: | --- |
| 1 | [emscripten](https://github.com/emscripten-core/emscripten) ⭐27k | C / JS | Added the `desynchronized` WebGL context attribute | [#27163](https://github.com/emscripten-core/emscripten/pull/27163) | ✅ Merged |
| 2 | [vyperlang/vyper](https://github.com/vyperlang/vyper) ⭐5k | Python | Codegen: raise instead of silently returning `None` in `safe_pow()` | [#5134](https://github.com/vyperlang/vyper/pull/5134) | ✅ Merged |
| 3 | [rust-lang/rust-analyzer](https://github.com/rust-lang/rust-analyzer) ⭐16k | Rust | Guarded an index panic in flycheck handling | [#22634](https://github.com/rust-lang/rust-analyzer/pull/22634) | 🟢 Approved, about to merge |
| 4 | [denoland/deno](https://github.com/denoland/deno) ⭐100k+ | Rust | Fixed a broken-pipe panic in `deno lint --rules` | [#35479](https://github.com/denoland/deno/pull/35479) | 🔄 In progress |

---

## 1. emscripten — `desynchronized` WebGL context attribute  ·  ✅ Merged

**PR:** [emscripten-core/emscripten#27163](https://github.com/emscripten-core/emscripten/pull/27163)
· **Issue:** [#8406](https://github.com/emscripten-core/emscripten/issues/8406)

### Why I chose this issue

I picked issue #8406, "Support the `desynchronized` (low-latency) canvas
attribute," in emscripten-core/emscripten. The browser `getContext()` API has
supported a `desynchronized` WebGL attribute for years, it lets the canvas skip
the browser's normal compositing/double-buffering for lower latency, which matters
for things like drawing apps and emulators. emscripten's C API just didn't expose
it, so anyone compiling to WebGL through emscripten had no way to ask for it.

A few reasons it stood out:

1. The scope is clear and self-contained. It's one boolean flag threaded through an
   existing, well-established plumbing path (C struct -> struct-info offsets -> JS
   library -> the `getContext()` call), with a matching read-back path. No new
   subsystem, no API redesign.
2. It's a real capability gap, not a style nit. The feature exists in every modern
   browser; emscripten was simply missing the pass-through.
3. emscripten is a heavily used toolchain (27k stars) and the change touches an
   interesting seam: handwritten C headers, a JSON struct-info file that generates
   byte offsets, and a JS "library" that the compiler stitches into the output. Good
   place to learn how that layering fits together.
4. I wanted to do an end-to-end feature on a large C/JS codebase: header, codegen
   glue, docs, and a browser test that actually verifies the round-trip.

My understanding of the gap: `EmscriptenWebGLContextAttributes` (in
`system/include/emscripten/html5_webgl.h`) had no `desynchronized` field, and the
JS context-creation code in `src/lib/libhtml5_webgl.js` never passed
`desynchronized` into `getContext()`. So even if a user wanted a low-latency
context there was no field to set. The plan was to add the field, register it in
the struct-info so offsets generate correctly, set it on the JS attributes object,
read it back in `emscripten_webgl_get_context_attributes()`, document it, and prove
it round-trips with a browser test.

### Phase II: Reproduction and Plan

#### Steps to Reproduce
1. Clone and set up emscripten:
   - `git clone https://github.com/emscripten-core/emscripten.git && cd emscripten`
   - install/activate a matching LLVM via emsdk so `emcc` works.
2. Open `system/include/emscripten/html5_webgl.h` and look at
   `struct EmscriptenWebGLContextAttributes`. There is no `desynchronized` member,
   so C code has no field to request a desynchronized context.
3. Open `src/lib/libhtml5_webgl.js` and find where the attributes object is built
   for `getContext()` (the object with `alpha`, `depth`, `powerPreference`,
   `failIfMajorPerformanceCaveat`, ...). There is no `desynchronized` key, so the
   flag is never forwarded to the browser even if it existed in the struct.
4. Confirm the read-back path in the same file (the
   `emscripten_webgl_get_context_attributes` write-out) has no `desynchronized`
   entry, so there's nothing to round-trip.
5. Net effect: a user who wants the documented browser behavior (lower-latency
   canvas) cannot get it through emscripten's WebGL API at all.

#### Reproduction Evidence
- Working branch on my fork: `fix/8406-desynchronized-canvas`
- Upstream issue: https://github.com/emscripten-core/emscripten/issues/8406

#### Implementation Plan
- Add `bool desynchronized` to `EmscriptenWebGLContextAttributes` in
  `html5_webgl.h`, next to the other booleans.
- Register the new field in `src/struct_info.json` so the generated offset tables
  (`struct_info_generated*.json`) pick it up and the `C_STRUCTS` macros resolve.
- In `src/lib/libhtml5_webgl.js`: pass `'desynchronized'` into the `getContext()`
  attributes object, and write it back in the get-attributes path so it round-trips.
- Document the field in `site/source/docs/api_reference/html5.h.rst`.
- Add a browser test that sets `desynchronized = true`, creates the context, reads
  the attributes back, and asserts the round-trip.
- Use the real web attribute name `desynchronized` (not a `lowLatency` alias),
  since the oldest Chrome emscripten supports (85) already ships `desynchronized`.

### Phase III: Implementation

#### Implementation Notes
The core change is small and follows the existing pattern for every other context
attribute. The interesting wrinkles were all around it: keeping the generated
struct-info in sync, deciding the attribute name, handling OffscreenCanvas, and
rebaselining the code-size expectation tests (adding a field changes the generated
JS, so the byte-exact `codesize` snapshots move).

#### Code Changes
- Pull request: https://github.com/emscripten-core/emscripten/pull/27163
- Branch: `fix/8406-desynchronized-canvas`
- Header — `system/include/emscripten/html5_webgl.h`:
  ```c
  bool renderViaOffscreenBackBuffer;
  bool desynchronized;
  ```
- Struct info — `src/struct_info.json` (so offsets generate into the `*_generated*`
  files and `C_STRUCTS.EmscriptenWebGLContextAttributes.desynchronized` resolves):
  ```json
  "renderViaOffscreenBackBuffer",
  "desynchronized"
  ```
- JS glue — `src/lib/libhtml5_webgl.js`, forward it into `getContext()` and write it
  back on read:
  ```js
  'desynchronized': !!HEAP8[attributes + {{{ C_STRUCTS.EmscriptenWebGLContextAttributes.desynchronized }}}],
  ```
  ```js
  {{{ makeSetValue('a', C_STRUCTS.EmscriptenWebGLContextAttributes.desynchronized, 't.desynchronized', 'i8') }}};
  ```
- Docs — added a `desynchronized` entry to the `EmscriptenWebGLContextAttributes`
  reference in `site/source/docs/api_reference/html5.h.rst`.

#### Testing Strategy
- Round-trip browser test in `test/browser/html5_webgl.c`: set
  `attrs.desynchronized = true`, create the context, call
  `emscripten_webgl_get_context_attributes()`, and assert the value came back as
  expected. Guarded by a compile-time `EXPECT_DESYNCHRONIZED` (`-1` = skip).
- Wired into `test/test_browser.py`: expect `1` on Chrome (which honors it) and `0`
  on browsers that report it back as false, and only on a normal canvas — not
  `OffscreenCanvas`, which doesn't honor the attribute (`is_chrome()` gate; skip when
  extra args select OffscreenCanvas).
- Rebaselined the `codesize` / minimal-runtime code-size expectation JSONs that
  shifted because the generated WebGL JS now includes the extra attribute
  (`test_codesize_hello_dylink_all`, the `hello_webgl*` minimal-runtime snapshots).

### Phase IV: Pull Request

- **Pull request:** https://github.com/emscripten-core/emscripten/pull/27163
- **Summary of what I contributed:** Closes #8406. Adds the `desynchronized` WebGL
  context attribute end to end: the C struct field, the generated struct-info
  offsets, the JS pass-through into `getContext()` and the read-back, reference
  docs, and a browser round-trip test. Lets emscripten programs opt into a
  lower-latency canvas, matching the browser API.
- **Feedback received and addressed:**
  - Naming: discussion about whether to also expose the legacy `lowLatency` name.
    Resolved by exposing only the real attribute `desynchronized`, since the oldest
    supported Chrome (85) already uses it; dropped the `lowLatency` mention from the
    docs too.
  - OffscreenCanvas: the round-trip assert failed there because OffscreenCanvas
    doesn't honor `desynchronized`. Gated the check to run only on a normal canvas.
  - Code size: the byte-exact `codesize` expectation tests flagged the change, and
    they kept drifting against emscripten's tip-of-tree LLVM. Rebaselined the
    affected snapshots and re-synced with `upstream/main` until CI was green; the
    maintainer helped land it through the moving baseline.
- **Status:** Merged into `main`.

---

## 2. vyperlang/vyper — `safe_pow()` raises instead of returning `None`  ·  ✅ Merged

**PR:** [vyperlang/vyper#5134](https://github.com/vyperlang/vyper/pull/5134)
· **Issue:** [#5026](https://github.com/vyperlang/vyper/issues/5026)

This was my CodePath open-source deliverable, documented end to end below.

### Why I chose this issue

I picked issue #5026, "safe_pow() silently returns None for two-variable
exponentiation instead of raising TypeCheckFailure," in vyperlang/vyper. It's a
small, contained bug in the compiler's code-generation path, written in Python,
which lines up well with what I already do. I work mostly in Python, and I'm
comfortable reading and modifying logic-heavy code, so tracing a value through a
compiler's codegen feels approachable to me.

A few reasons it stood out:

1. The scope is clear. It's about replacing a silent `None` return with an
   explicit compiler error, using an exception class the file already imports.
   There's no new feature, no API change, and no design work needed.
2. It's a genuine correctness problem. The `else` branch returns `None`, and the
   caller passes that straight into IR construction, so a bypassed guard upstream
   would cause a cryptic downstream crash instead of a clean, actionable error.
3. Vyper is a notable, actively maintained project (5.2k stars) and the main
   smart-contract language for Ethereum, which makes it a great place to learn how
   a real-world compiler is organized and how a team handles contributions.
4. Personally, I want to get better at jumping into a large unfamiliar codebase,
   following a value from one function down to its caller, and reusing what's
   already there instead of reinventing it.

After reading through the issue, here's my understanding of the problem. When both
the base and the exponent of a `**` expression are runtime variables, `safe_pow()`
in `vyper/codegen/arithmetic.py` hits an `else` branch that just returns `None`.
The caller in `expr.py` feeds that `None` directly into IR construction, so instead
of a clear compiler error you'd get a confusing downstream crash if the type guard
upstream were ever bypassed. The branch is currently unreachable by design, so this
is defensive hardening. My plan is to replace the silent `return` with an explicit
`raise TypeCheckFailure(...)` with a clear message (the class is already imported
and used elsewhere in the file), and to check whether the newer venom codegen
backend has the same pattern so I can keep the two consistent.

### Phase II: Reproduction and Plan

#### Steps to Reproduce
1. Clone the repo and set up the dev environment:
   - `git clone https://github.com/vyperlang/vyper.git && cd vyper`
   - `python -m venv .venv && source .venv/Scripts/activate` (Windows) or `source .venv/bin/activate` (macOS/Linux)
   - `pip install -e .`
2. Open `vyper/codegen/arithmetic.py` and find `safe_pow(x, y)`. Look at the final
   `else` branch (when neither the base nor the exponent is a compile-time
   constant). It runs a bare `return`, which silently returns `None`.
3. Open `vyper/codegen/expr.py` and find the `Pow` handler (around line 447). It
   feeds `safe_pow(...)`'s return value straight into `IRnode.from_list(...)` with
   no `None` check, so a `None` here propagates into IR construction.
4. Confirm the branch is currently guarded by the front end. In a Python shell:
   ```python
   from vyper import compile_code
   compile_code("@external\ndef g(a: uint256, b: uint256) -> uint256:\n    return a ** b",
                output_formats=["bytecode"])
   ```
   This is rejected with `InvalidOperation` before codegen, which is why the
   `else` branch is unreachable today.
5. To see the latent bug directly, temporarily relax that front-end guard and
   recompile `a ** b`. Instead of a clean compiler error you get a cryptic crash
   from the `None` flowing into IR construction. That is exactly the problem the
   issue describes: a silent `None` instead of a clear error.

#### Reproduction Evidence
- My working branch on my fork: https://github.com/UnknownHacker1/vyper/tree/fix/5026-safe-pow-raise-typecheckfailure
- Upstream issue: https://github.com/vyperlang/vyper/issues/5026

#### Implementation Plan
- Replace the silent `return` in `safe_pow()`'s final `else` branch with an
  explicit raise, so the impossible state fails loudly instead of returning `None`.
- Match existing codebase conventions: raise a panic-style exception and mark the
  unreachable branch with `# pragma: nocover`, consistent with the defensive raise
  already a few lines above and with the venom backend's `safe_pow`.
- No new feature, no API change, and no behavior change for valid programs.
- Verify: existing exponent tests pass, valid `**` still compiles, `a ** b` is
  still rejected at the front end, and pre-commit (isort/black/flake8/mypy) is clean.

### Phase III: Implementation

#### Implementation Notes
Implemented the fix in `vyper/codegen/arithmetic.py`. Replaced the bare `return`
in `safe_pow()`'s final `else` branch with a clear, loud failure for the
impossible state, and added a one-line comment noting the type checker guarantees
at least one literal operand. The approach evolved through maintainer review (see
Phase IV): it started as `TypeCheckFailure`, switched to `CodegenPanic` at the
reviewer's suggestion, and was then simplified to match the codebase's style.

#### Code Changes
- Pull request: https://github.com/vyperlang/vyper/pull/5134
- Branch: https://github.com/UnknownHacker1/vyper/tree/fix/5026-safe-pow-raise-typecheckfailure
- Final change in `vyper/codegen/arithmetic.py`:
  ```python
  else:  # pragma: nocover
      # type checker guarantees pow has at least one literal operand
      raise CodegenPanic("unreachable")
  ```

#### Testing Strategy
- Compile checks: `a ** 2` (literal exponent) still compiles to identical
  bytecode; `a ** b` (both runtime) is still rejected at the front end with
  `InvalidOperation`, confirming the `else` branch stays unreachable.
- Test suite: `pytest tests/functional/codegen/types/numbers/test_exponents.py`
  passes (23 passed).
- Lint and types: ran the project's exact pre-commit hooks (isort, black, flake8,
  mypy) at their pinned versions, all pass.
- Confirmed no existing test referenced `safe_pow` or relied on the old `None` return.

### Phase IV: Pull Request

- **Pull request:** https://github.com/vyperlang/vyper/pull/5134
- **Summary of what I contributed:** Fixes #5026. In the classic codegen backend,
  `safe_pow()` silently returned `None` for the (currently unreachable) case where
  neither exponentiation operand is a compile-time constant. The PR raises
  `CodegenPanic("unreachable")` instead, so the impossible state fails loudly rather
  than propagating a `None` into IR construction. Marked `# pragma: nocover` since
  the front end already rejects this case.
- **Feedback received and addressed:**
  - Reviewer (Sporarum) suggested raising `CodegenPanic` instead of
    `TypeCheckFailure`, since the case is already caught by the type checker. Done.
  - Reviewer suggested a simpler comment and message and moving `# pragma: nocover`
    onto the `else:` line. Applied their exact suggestion.
  - Reviewer asked me not to force-push the branch; switched to normal follow-up commits.
- **Status:** Merged (approved by maintainer Sporarum and merged into `master`).

---

## 3. rust-lang/rust-analyzer — guarded an index panic in flycheck  ·  🟢 Approved, about to merge

**PR:** [rust-lang/rust-analyzer#22634](https://github.com/rust-lang/rust-analyzer/pull/22634)
· **Issue:** [#21638](https://github.com/rust-lang/rust-analyzer/issues/21638)

### Why I chose this issue

I picked issue #21638, a panic in rust-analyzer's flycheck (on-save check)
handling. rust-analyzer is the Rust language server behind the VS Code / editor
experience that basically every Rust developer uses, so a panic in its
save-notification path is the kind of bug that silently kills people's diagnostics
mid-session. The fix is a one-line defensive guard, which made it a clean way to
work inside a large, real-world LSP codebase.

A few reasons it stood out:

1. The scope is tightly bounded: an unchecked slice index that can panic on an
   empty list. The fix is to guard it. No behavior change for the normal case.
2. It's a genuine robustness bug. A soft assertion right above it (`always!`) only
   *logs* in release builds, it doesn't stop execution, so the very next line
   indexes `world.flycheck[0]` and panics when the list is actually empty.
3. rust-analyzer is a prestigious, fast-moving rust-lang project. Good place to
   learn how an industrial LSP server is structured and how that team reviews.
4. It fit a pattern I like: read the surrounding code, notice the existing idiom
   for "optional state," and reuse it instead of inventing something.

My understanding of the bug: in `run_flycheck`
(`crates/rust-analyzer/src/handlers/notification.rs`), under the "once" invocation
strategy, the code asserts there is exactly one flycheck handle and then indexes
`world.flycheck[0]` to restart it. The assertion is `stdx::always!`, which in
release is non-fatal, it logs and keeps going. So when `world.flycheck` is empty
(the condition the assert is meant to catch), execution falls straight through to
`world.flycheck[0]`, which panics and takes down the handler.

### Phase II: Reproduction and Plan

#### Steps to Reproduce
1. Clone and build rust-analyzer:
   - `git clone https://github.com/rust-lang/rust-analyzer.git && cd rust-analyzer`
   - `cargo build` (toolchain per `rust-toolchain.toml`).
2. Open `crates/rust-analyzer/src/handlers/notification.rs` and find `run_flycheck`.
   Under the "once" invocation strategy you'll see:
   ```rust
   stdx::always!(
       world.flycheck.len() == 1,
       "should have exactly one flycheck handle when invocation strategy is once"
   );
   let saved_file = vfs_path.as_path().map(ToOwned::to_owned);
   world.flycheck[0].restart_workspace(saved_file);
   ```
3. Note that `stdx::always!` is a soft assertion: on release builds it logs the
   message but does **not** return or abort. So if `world.flycheck` is empty, the
   guard does not actually stop the function.
4. The next statement, `world.flycheck[0]`, indexes an empty slice and panics. The
   panic is reached exactly in the state the assertion was warning about, an empty
   flycheck list when a save fires under the "once" strategy (as reported in #21638).

#### Reproduction Evidence
- Working branch on my fork: `fix/21638-flycheck-index-panic`
- Upstream issue: https://github.com/rust-lang/rust-analyzer/issues/21638

#### Implementation Plan
- Replace the unchecked `world.flycheck[0]` index with a `first()` guard so an
  empty list is a safe no-op instead of a panic.
- Keep the existing `always!` assertion in place, it's still useful as a signal in
  debug builds that the "exactly one handle" invariant was violated.
- No behavior change in the normal (one-handle) case; the guard only affects the
  previously-panicking empty case.

### Phase III: Implementation

#### Implementation Notes
One-line, surgical change in `run_flycheck`. The codebase already uses
`if let Some(..) = ...first()` style guards for optional state elsewhere, so this
matches the existing idiom rather than introducing a new pattern. The soft
assertion stays so the invariant is still flagged in debug builds.

#### Code Changes
- Pull request: https://github.com/rust-lang/rust-analyzer/pull/22634
- Branch: `fix/21638-flycheck-index-panic`
- Change in `crates/rust-analyzer/src/handlers/notification.rs`:
  ```rust
  // before
  world.flycheck[0].restart_workspace(saved_file);

  // after
  if let Some(flycheck) = world.flycheck.first() {
      flycheck.restart_workspace(saved_file);
  }
  ```

#### Testing Strategy
- `cargo build` / `cargo check` clean.
- Reasoned through both paths: with exactly one handle, `first()` yields it and
  behavior is identical to the old index; with an empty list, the old code panicked
  and the new code is a no-op, which is the intended robust behavior.
- No existing test depended on the panic; the change is a strict superset of the
  previous safe behavior.

### Phase IV: Pull Request

- **Pull request:** https://github.com/rust-lang/rust-analyzer/pull/22634
- **Summary of what I contributed:** Fixes rust-lang/rust-analyzer#21638. Guards the
  `world.flycheck[0]` index in `run_flycheck` with a `first()` check so an empty
  flycheck list under the "once" invocation strategy is a no-op instead of a panic.
  The pre-existing `always!` soft assertion is kept for debug-build signal.
- **Feedback received and addressed:**
  - rustbot flagged the issue reference format; amended the commit to use the
    canonical `Fixes rust-lang/rust-analyzer#21638` so it links correctly.
  - Kept the change minimal per the project's preference for tightly-scoped fixes.
- **Status:** Approved on the rust-lang side, queued to merge.



---

## 4. denoland/deno — broken-pipe panic in `deno lint --rules`  ·  🔄 In progress

**PR:** [denoland/deno#35479](https://github.com/denoland/deno/pull/35479)
· **Issue:** [#30248](https://github.com/denoland/deno/issues/30248)

### Why I chose this issue

I picked issue #30248: `deno lint --rules` panics with a broken-pipe error when its
output is piped into a program that closes the pipe early, like
`deno lint --rules | head`. It's a classic Unix papercut, a CLI that doesn't
tolerate a closed stdout, and deno is a 100k-star runtime, so it's a high-visibility
place to fix a real, reproducible bug. deno is also Rust, which I wanted more reps in.

A few reasons it stood out:

1. The bug is concrete and trivially reproducible from the shell, and the fix has a
   clear, idiomatic shape.
2. deno already had the right tool for this, a `write_to_stdout_ignore_sigpipe`
   helper used elsewhere, so the fix is "use the existing pattern here too" rather
   than inventing error handling.
3. Touching a real CLI's output path on a huge project is good practice for writing
   a change that matches house style and passes a strict review/CI process.

My understanding of the bug: `print_rules_list` in `cli/tools/lint/mod.rs` writes
the rule listing directly with `println!` (and `write_json_to_stdout(...).unwrap()`
in the JSON branch). When stdout is a pipe that's already closed by the reader,
each `println!` hits a `BrokenPipe` error; `println!` panics on write failure, and
the `.unwrap()` in the JSON path panics too. So instead of exiting cleanly, the
command crashes with a panic.

### Phase II: Reproduction and Plan

#### Steps to Reproduce
1. Build deno from source (`cargo build`) or use a release binary.
2. Run the rules listing piped into a reader that closes early:
   ```
   deno lint --rules | head -n 1
   ```
3. `head` reads one line and closes the pipe. The next `println!` in
   `print_rules_list` writes to a closed stdout, hits `BrokenPipe`, and panics
   instead of exiting cleanly. (The JSON path, `deno lint --rules --json | head`,
   panics the same way via `write_json_to_stdout(...).unwrap()`.)

#### Reproduction Evidence
- Working branch on my fork: `fix/30248-lint-rules-broken-pipe`
- Upstream issue: https://github.com/denoland/deno/issues/30248

#### Implementation Plan
- Stop writing directly with `println!` / `write_json_to_stdout().unwrap()` inside
  the loop.
- Build the entire listing (both the text and JSON branches) into a single
  `String` buffer using `writeln!`.
- Write the buffer once through deno's existing
  `display::write_to_stdout_ignore_sigpipe`, which treats `BrokenPipe` as
  `Ok(())`, the same helper deno already uses for pipe-safe stdout.
- No change to the actual output content; only how it's written.

### Phase III: Implementation

#### Implementation Notes
Converted `print_rules_list` to accumulate into a buffer and flush once. Added
`use std::fmt::Write as _;` for `writeln!` into a `String`, replaced every
`println!` / `write_json_to_stdout(...).unwrap()` with `writeln!(output, ...)`, and
ended the function with a single pipe-safe write. This matches the pattern deno
already uses elsewhere for stdout that may be closed early.

#### Code Changes
- Pull request: https://github.com/denoland/deno/pull/35479
- Branch: `fix/30248-lint-rules-broken-pipe`
- In `cli/tools/lint/mod.rs`:
  ```rust
  use std::fmt::Write as _;
  ```
  ```rust
  let mut output = String::new();
  // ... build the listing with writeln!(output, ...) in both the json and
  // text branches instead of println! / write_json_to_stdout().unwrap() ...
  writeln!(output, "- {} {}", rule.code(), colors::green(enabled)).unwrap();
  // ...
  // Write all at once and ignore broken pipe errors, e.g. when the output is
  // piped to a program that closes the pipe early like `head` (#30248).
  let _ = display::write_to_stdout_ignore_sigpipe(output.as_bytes());
  ```

#### Testing Strategy
- Manual repro before/after: `deno lint --rules | head -n 1` panics on the old
  code and exits cleanly on the new code; same for `--json | head`.
- Confirmed the visible output is byte-for-byte the same when not piped (the buffer
  just batches what the `println!`s used to emit).
- Relying on deno's CI for the full lint/test matrix.

### Phase IV: Pull Request

- **Pull request:** https://github.com/denoland/deno/pull/35479
- **Summary of what I contributed:** Fixes #30248. `print_rules_list` now collects
  the rule listing into a single buffer and writes it through
  `write_to_stdout_ignore_sigpipe`, so `deno lint --rules | head` (and the `--json`
  variant) exit cleanly instead of panicking on a broken pipe.
- **Disclosure:** per deno's contributing policy, the PR description discloses use of
  AI assistance.
- **Status:** Open, working through review and CI.
