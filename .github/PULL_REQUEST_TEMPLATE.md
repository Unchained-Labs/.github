## What this changes

<!-- One or two sentences. What behaviour is different after this merges? -->

## Why

<!-- The reasoning. If this fixes a misfire, name the architecture the rule
     misread. If it adds a rule, name the line from the reference architecture
     it enforces. -->

## Checklist

- [ ] `pnpm test` passes
- [ ] `pnpm build` and `pnpm typecheck` pass
- [ ] If this adds or changes a rule: a fixture in `test/fixtures/bad/` fails
      without it, **and** `test/fixtures/good/` still lints clean
- [ ] If this changes a number in the docs: the number came from a run, and the
      run is in the description
- [ ] No new dependencies, or the reason one was needed is above

## Evidence

<!-- Paste the before/after output. For a cost or statistics change, the actual
     figures, not a description of them. This is the part reviewers read. -->

```
```
