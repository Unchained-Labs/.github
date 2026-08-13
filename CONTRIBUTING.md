# Contributing

These are small, opinionated tools. The bar is not high effort — it is
**checkability**.

## The three rules

1. **Every finding needs evidence.** A rule that fires without being able to
   quote the source span that triggered it is a guess. Guesses train people to
   pass `--no-verify`, and a linter nobody runs is worse than no linter.

2. **Every number needs a measurement.** No "significantly faster", no
   "roughly 40% cheaper" without the run that produced it. If you do not have the
   number yet, say so — `decorrelate`'s statistics are tested against published
   worked examples for exactly this reason, and `branding` has a script that
   asserts the ratios written in its own docs, because two of them were wrong.

3. **A new rule needs a fixture that fails without it.** Both directions: a case
   in `test/fixtures/bad` that the rule catches, and confirmation that
   `test/fixtures/good` still lints clean. A rule with no false-positive test is
   a rule that will produce false positives.

## Practicalities

```sh
pnpm install
pnpm build
pnpm test
```

Node 20+. pnpm 9. No other setup.

- **Commits**: imperative, lowercase, no type prefix. Say what the commit does.
  If it needs a paragraph, write one — the reasoning is worth more than the
  summary line.
- **Comments**: explain *why*, never *what*. A comment restating the next line is
  noise the moment the PR merges. A comment explaining why a regex has no
  trailing `\b`, or why a barrier is justified here, earns its place.
- **No new dependencies** without a reason in the PR description. `preflight`'s
  GitHub Action is deliberately dependency-free; a cost bot that needs a 40MB
  install to post one comment is a cost bot nobody adopts.

## Adding a rule to graphlint

1. Write the failing case into `test/fixtures/bad/`.
2. Add the rule to `src/rules/index.ts` with an `id`, a `severity`, a `summary`,
   and a `reference` naming the section it enforces.
3. Give every finding a `detail` (why this costs money or correctness) and a
   `fix` (what to do instead). Both are asserted by the test suite.
4. Confirm `test/fixtures/good/` still lints clean.

Severity convention: `error` costs money or does not terminate; `warning` will at
scale; `info` is a smell. Only `error` fails CI by default.

## Adding a workflow to workflow-hub

It must pass `graphlint check` with zero findings, and it must teach a shape the
registry does not already cover. Add the curation entry in `src/registry.ts` —
the tests check that every documented arg is read and every read arg is
documented.

## What gets declined

- A rule that cannot state why it fired.
- A benchmark with no reproduction.
- A feature that adds a runtime dependency to `workflow-hub` — the entire design
  is that you own the file after it lands.
- Marketing language in a README. See
  [voice.md](https://github.com/Unchained-Labs/branding/blob/main/brand/voice.md).

## Disagreeing

If a rule is wrong for your codebase, turn it off in `.graphlintrc.json` and open
an issue saying which architecture it misreads. That is more useful than a
workaround, and several rules exist in their current form because someone did
exactly that.
