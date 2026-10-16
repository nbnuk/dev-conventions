# Linting

Lint turns the mechanical parts of our conventions into checks that fail
`./gradlew check`. Judgement rules (simple linear code, no speculative
defences, meaningful names) stay as agent rules.

## Tools

| Project type | Formatter | Linters | Config |
|---|---|---|---|
| Java (nbn-recorder-workers, nbn-core) | Spotless with palantir-java-format | PMD, SpotBugs (main code only) | `config/pmd/ruleset.xml`, `config/spotbugs/exclude.xml` |
| Grails (nbn-recorder-grails, nbn-components-plugin) | Spotless: import order and whitespace only | CodeNarc | `config/codenarc/codenarc.groovy` |

Repos of the same type keep **identical** ruleset files. Change them together.

Reports are written to `build/reports/pmd/`, `build/reports/spotbugs/` and
`build/reports/codenarc/`.

## Where lint runs

- **Locally and by agents:** `./gradlew spotlessApply`, then `./gradlew check`.
  This is the only enforcement point (rule `lint-verification`).
- **Not in CI:** Travis and CodeBuild do not run the linters, so a lint failure
  never breaks a build or blocks publishing.

## Fix or suppress

Fix the code by default. Suppress only when the flagged code is deliberate, at
the narrowest scope, with a reason:

| Scope | Java (PMD) | Groovy (CodeNarc) |
|---|---|---|
| Line | `// NOPMD - RuleName: reason` | `// codenarc-disable-line RuleName` |
| Method | `@SuppressWarnings("PMD.RuleName")` | `@SuppressWarnings('RuleName')` |

Put a one-line comment above a method-level suppression saying why. Prefer the
method over the class, so new code added to the class is still checked. For
example, job runners and pollers suppress `AvoidCatchingGenericException`
on each boundary method rather than on the class.

If a rule is wrong across the whole codebase, change the ruleset in every repo
of that type instead of suppressing it repeatedly. Do this only with agreement.

## Agent rules now enforced by lint

| Rule | Enforced by | Still judgement |
|---|---|---|
| `readable-formatting` | Spotless (layout); `IfStatementBraces`, `ForStatementBraces` etc. (CodeNarc), `ControlStatementBraces` (PMD) | whitespace between logical steps, naming, wrapping style |
| `lean-controllers` | `MethodSize`, `AbcMetric`, `CyclomaticComplexity` (CodeNarc) | what belongs in a command object or service |
| `service-layer` (catch only at boundaries) | `CatchException`, `CatchRuntimeException` (CodeNarc), `AvoidCatchingGenericException` (PMD) | whether a catch is a real recovery path |
| `testing`, `tdd-design-review-loop` (no leftover debugging) | `Println` (CodeNarc), `SystemPrintln` (PMD) | |
| `simple-linear-code` | `CognitiveComplexity` (PMD), `NestedBlockDepth` (CodeNarc) as a backstop | when to extract a method |

`NPathComplexity` is deliberately not used: it penalises flat lists of guard
clauses, which is the style `simple-linear-code` asks for.
