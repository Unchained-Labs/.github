# Security policy

## Reporting

Email **erwin.lejeune15@gmail.com** with `SECURITY` in the subject, or open a
[private vulnerability report](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing-information-about-vulnerabilities/privately-reporting-a-security-vulnerability)
on the affected repository.

Please do not open a public issue for a vulnerability.

Include what you have: the repository, a version or commit, what an attacker
gains, and a reproduction if you have one. A report with a reproduction gets
fixed considerably faster than one without.

**Expect a first response within 5 working days.** These are side projects
maintained by one person — that is the honest number, not a target I will quietly
miss.

## Scope

In scope:

- Arbitrary code execution, path traversal, or file writes outside the target
  directory in any CLI here.
- The `preflight` GitHub Action leaking a token, or posting to a repository other
  than the one that invoked it.
- `authsweep` or `graphlint` reading or transmitting files outside the paths they
  were given. None of these tools make network calls; if one does, that is a bug
  worth reporting.
- Anything that makes a tool report a **false clean** — an `authsweep` scan that
  silently skips a file and reports no findings, or a `graphlint` rule that
  stops firing. A security tool that quietly under-reports is worse than one that
  crashes.

Out of scope:

- False positives and false negatives in the heuristics. These are documented
  limitations, not vulnerabilities — `authsweep` is explicitly not a taint
  analyser, and its README says so. Open a normal issue.
- Cost estimates being wrong. `preflight` is an estimator with published
  assumptions; a wrong estimate is a calibration issue.
- Dependency advisories with no reachable path in these packages. Report them,
  but they are not urgent.

## What these tools do and do not touch

Worth stating plainly, because it bounds the blast radius:

- **No network calls.** Every tool here is local and offline. `graphlint`,
  `preflight`, `decorrelate` and `authsweep` read files and print. The only
  exception is `preflight`'s Action, which posts one comment to the PR that
  invoked it.
- **No writes outside an explicit target.** `workflow-hub add` writes one file to
  a directory you name, and refuses to overwrite without `--force`.
- **No model calls.** None of these tools spend money. `authsweep` emits a graph
  for you to run rather than running it — deliberately, because verification
  costs money and that decision is yours.

## Supported versions

Alpha, `0.x`. Fixes land on `main` and in the next release; there are no
backports. Pin exact versions.
