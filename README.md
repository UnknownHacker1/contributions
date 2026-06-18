# CodePath Open-Source Contribution

## Issue
**Project:** [vyperlang/vyper](https://github.com/vyperlang/vyper)
**Issue:** [#5026 – safe_pow() silently returns None instead of raising a TypeCheckFailure](https://github.com/vyperlang/vyper/issues/5026)

## Why I Chose This Issue

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

I've also left a comment on the issue introducing myself as a CodePath student and
asking to be assigned. I'll update this section once a maintainer replies.
