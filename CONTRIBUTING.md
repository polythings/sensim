# Contributing to sensim

This is the code guide for everyone who changes this repository, people and AI agents alike. Formatters and linters
enforce the mechanical rules (see [Checks](#checks)). This guide covers what they can't: how code should read.
[SPEC.md](SPEC.md) holds the design, the requirements, and the roadmap.

## Principles

1. **Readable first.** Someone new should understand a function from its name, its signature, and one pass through
   the body. Fast code and clear code are not opposites: name the trick and explain why it's there.
2. **Idiomatic.** Rust should read like Rust, not like Java or C++.
3. **Modular.** Each crate has one job and a small public API. Each protocol is its own crate.
4. **One definition of each thing.** Before writing a helper, constant, or type, look for an existing one.
5. **Behaviour, spec, and tests change together.** A change users can see updates SPEC.md, adds or updates tests, and
   adds a line to [CHANGELOG.md](CHANGELOG.md).
6. **Every fault and behaviour is a setting.** It has an off state, a name for the mechanism it changes
   (`clock-drift-ppm`, not `bad-clock`), and docs saying what it costs and why its default is what it is.

## Comments and docs

- **Public items get a short doc comment**: what it does, and anything a caller must know (errors, units,
  invariants). One to three lines is usually enough.
- **Comments explain why, not what.** Explain a non-obvious choice, a workaround, or a number that came from a
  measurement.
- **No long comments.** If an explanation needs a page, it belongs in SPEC.md. Link to it from the code.

## Credit

sensim learns from other projects' ideas, never from copying their code. [PRIOR_ART.md](PRIOR_ART.md) lists the ideas
we took and the projects we're studying.

- **Never copy code**, whatever its license: not by pasting, and not by translating it into Rust.
- **Credit ideas from open-source projects** with a row in PRIOR_ART.md and a short comment where the idea is
  implemented, for example `// Scheduled from the intended start time, as in wrk2.`
- **Don't name open-core, dual-licensed, or source-available projects, or commercial products, as a source.** Naming
  them in comparisons is fine.

## Rust

- **Errors:** `thiserror` enums in libraries, `anyhow` in binaries. Keep the cause with `#[from]` or `#[source]`. No
  `unwrap()` outside tests; `expect("why this can't fail")` only when it truly can't.
- **Types over strings:** enums, newtypes, and `FromStr`/`Display` instead of strings and magic numbers. Parse
  configuration once at the boundary and pass typed values along.
- **Small functions:** split anything over roughly 60 lines, or with more than 4 or 5 parameters (use a struct).
- **The data path doesn't allocate per sample.** Reuse buffers; generate and encode frames in place.
- **Async:** never block inside async code. Don't hold a `Mutex` across an `.await`.
- **Suppressing a lint:** `#[expect(lint, reason = "...")]` with a real reason. Never `#[allow]`, and never relax the
  workspace lints to make a check pass.

## Tests

- **A bug fix starts with a test that fails**, then the fix.
- **Tests run offline** and on their own. Protocol tests talk to an in-process peer, never a network service.
- **Determinism is tested:** the same seed and configuration must give byte-identical output.
- **Benchmarks are not tests.** They run by hand, never in CI.

## Checks

Install once: `uv tool install prek && prek install`. Hooks then run on commit (formatters) and on push (clippy).
Before you push, or before calling work done:

```bash
prek run --all-files --hook-stage pre-push   # every formatter and linter
cargo test --workspace
```

CI runs the same on every push and pull request.

## Dependencies

Keep dependencies on their newest versions. [Dependabot](.github/dependabot.yml) opens a pull request each week per
ecosystem (Actions, Cargo, the Rust toolchain) with the minor and patch updates, and one per major update. It waits
until a version is 7 days old. Actions are pinned to a commit with the version in a comment
(`uses: actions/checkout@<sha> # v7.0.1`). The Rust version is in `rust-toolchain.toml`.

## Commits

- **Subject:** a short imperative sentence with the area first, for example `Signal: …`, `Faults: …`, `MQTT: …`,
  `CLI: …`, or `SPEC: …`. Add a body when the why isn't obvious.
- **Scope:** one logical change per commit.
- **Secrets:** never commit credentials or `.env*` files.

## Changelog

[CHANGELOG.md](CHANGELOG.md) is for users, not a list of commits. Anything a user can notice gets one line under
`## Unreleased`, in `Added`, `Changed`, `Deprecated`, `Removed`, `Fixed`, or `Security`, in the same commit as the
change.

## License

Contributions are licensed under the Apache License 2.0, like the rest of the project (see [LICENSE](LICENSE)).
