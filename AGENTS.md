# Agent instructions

sensim: a field of simulated IoT devices for testing and benchmarking. A Rust workspace (`crates/`). The project is in
design: [SPEC.md](SPEC.md) lists every requirement and feature and where each one stands.

## Before changing code

- Follow [CONTRIBUTING.md](CONTRIBUTING.md). It is the code guide, and it applies to you.
- For anything beyond a small fix, check SPEC.md. Agree on the design with the user before building something new,
  and record the decision in SPEC.md's decision log.

## Rules

- **Done means checked.** Before you report work as finished, run `prek run --all-files --hook-stage pre-push` and
  `cargo test --workspace`. Report failures as failures.
- **Fix the code, not the checks.** Don't loosen lint or format configs. Don't add `#[allow]` to get a check passing.
  Use `#[expect(..., reason = "...")]` only when the tool is genuinely wrong.
- **Keep things in sync:** a change in behaviour updates SPEC.md (requirement, feature status), the tests, and
  `CHANGELOG.md` (under `Unreleased`) in the same commit.
- **Every fault and behaviour is a setting** with an off state, named for its mechanism (SPEC.md §15).
- **Ideas, not code.** Never copy or translate another project's code. Credit ideas in `PRIOR_ART.md`
  (CONTRIBUTING.md, "Credit").
- **Tests run offline.** No network or external services in tests; protocol tests use in-process peers.
- **Never commit secrets** or `.env*` files.

## Commands

```bash
cargo build                                  # debug build
cargo test --workspace                       # tests
prek run --all-files --hook-stage pre-push   # every formatter and linter
```
