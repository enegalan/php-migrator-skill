---
name: php-migration
description: >-
  Run a complete, evidence-based PHP version upgrade from PHP 5.x up to PHP 8.5:
  discovery, multi-version environment, dependency matrix, a point-by-point audit
  against the official PHP migration documentation, baseline and characterization
  tests, execution of a per-item action plan, type hardening, CI matrix, staged
  deployment and cleanup. Use when the user asks to migrate, upgrade, modernize or
  bump the PHP runtime of a codebase (e.g. "migrate this app to PHP 8.5",
  "upgrade from PHP 5.6", "we need to get off deprecated PHP APIs").
license: MIT
compatibility: Requires git, Composer 2, a PHP 5.x and a PHP 8.5 runtime available side by side, and network access to php.net for the documentation audit.
metadata:
  audience: developer
---

# PHP Migration (5.x → 8.5)

A migration is only complete when **every behavior change is justified by an item
in the official PHP documentation** and **proven not to break the application**.

This skill therefore runs a strict pipeline:

```
discover → set up env → resolve dependencies → AUDIT (doc item by doc item)
        → BASELINE (freeze current behavior) → EXECUTE (red→green, one item at a time)
        → harden types → test & CI → deploy → clean up
```

The audit comes **before** any code is touched, and the baseline comes **before**
any code is changed. Never reorder these.

---

## Contents

1. [Modes](#modes)
2. [Operating principles](#operating-principles)
3. [Artifacts produced](#artifacts-produced)
4. [Phase 0 — Discovery](#phase-0--discovery)
5. [Phase 1 — Environment](#phase-1--environment)
6. [Phase 2 — Dependency matrix](#phase-2--dependency-matrix)
7. [Phase 3 — Documentation-driven audit](#phase-3--documentation-driven-audit)
8. [Phase 4 — Baseline & characterization](#phase-4--baseline--characterization)
9. [Phase 5 — Execute the action plan](#phase-5--execute-the-action-plan)
10. [Phase 6 — Type hardening](#phase-6--type-hardening)
11. [Phase 7 — Test & CI matrix](#phase-7--test--ci-matrix)
12. [Phase 8 — Deploy & rollback](#phase-8--deploy--rollback)
13. [Phase 9 — Cleanup & report](#phase-9--cleanup--report)
14. [Appendix A — Error → cause → fix](#appendix-a--error--cause--fix)
15. [Appendix B — Official sources](#appendix-b--official-sources)
16. [Appendix C — Framework notes](#appendix-c--framework-notes)
17. [Portability](#portability)
18. [Quick reference](#quick-reference)

---

## Modes

| Mode                   | Path                                                                                 | When                                                                               |
|------------------------|--------------------------------------------------------------------------------------|------------------------------------------------------------------------------------|
| **stepwise** (default) | `5.6 → 7.0 → 7.4 → 8.0 → 8.1 → 8.2 → 8.3 → 8.4 → 8.5`, green checkpoint at each step | Any production application. Safe, incremental, reversible.                         |
| **big-bang**           | `5.x → 8.5` in one branch, running every checklist in order                          | Legacy/prototype code, hard deadlines, or a codebase with near-zero test coverage. |

### Which jumps you cannot skip

Within a major version (`7.0` → `7.4`) PHP only adds features and warns about
removals — nothing that worked stops working. Across a major version
(`7.4` → `8.0`) functions are removed and existing behaviour changes.

So two boundaries are unavoidable: **`5.x → 7.0`** and **`7.4 → 8.0`**. What
each one removes is not written down here — that is what the audit in Phase 3
is for.

The steps in between (`7.0 → 7.4`, `8.0 → 8.5`) are optional, but stepping
through them is cheaper: each minor version prints the deprecations the next
major will remove, and each step ends green, so a break is pinned to one
version instead of a six-version range.

Use the full sequence unless the user opts out.

> **Big-bang mode still audits every step.** Compressing the implementation
> does not compress the verification.

---

## Operating principles

These are rules, not suggestions. If a rule cannot be satisfied, stop and report
why — do not work around it silently.

1. **The PHP manual is the source of truth.** This skill carries no summary of
   it — the agent reads the pages and cites them. When this skill and php.net
   disagree, php.net wins. Every action must cite the doc item that justifies
   it.
2. **Audit before you edit.** No code change without a corresponding audit row.
3. **Baseline before you edit.** No code change before the current behavior is
   captured and reproducible.
4. **One item at a time.** Fix one audit row, run its test, run the suite, then
   move on. Do not batch unrelated fixes; you will not know which one broke it.
5. **Red → green.** Write or update the test *first*, see it fail for the right
   reason, then change the code.
6. **Tests may change, but only to assert documented new behavior.** Never edit
   a test just to make it pass. If a test must change, the change is explained
   by an `ACCEPTED-DIFF` audit row.
7. **Never edit `vendor/`.** If a dependency is incompatible, upgrade or replace
   the dependency — that is a separate action-plan item.
8. **Never weaken CI to get green.** No `@`, no silenced deprecations, no
   `--ignore-platform-reqs` in committed config, no skipping test suites.
9. **Never guess.** Package versions, PHP behavior, API signatures: verify them
   (php.net, `composer show`, `php -r`). If unverified, mark it unverified.
10. **Coverage must not regress.** Each accepted item keeps coverage at or above
    the baseline.
11. **Every step is reversible.** One commit per phase; commits are small and
    independently revertible.
12. **Report honestly.** If a golden diff cannot be explained by an audit row,
    say so and stop. Do not regenerate the golden file to hide it.

---

## Artifacts produced

All artifacts live in the repository being migrated:

```
docs/migration/
├── 00-baseline.md              # inventory + current state (Phase 0)
├── 01-environment.md           # runtime matrix used (Phase 1)
├── 02-dependencies.md          # dependency decisions (Phase 2)
├── audit/
│   ├── 5.6-7.0.md              # one file per ladder step
│   ├── 7.0-7.4.md
│   ├── 7.4-8.0.md
│   └── ...
├── action-plan.md              # one task per APPLIES/CONDITIONAL row
├── golden/                     # the recorded contract (Phase 4)
│   ├── behaviour.golden.txt    # must not change during the migration
│   └── api.golden.txt          # changes at every version step, on purpose
└── MIGRATION-REPORT.md         # final summary (Phase 9)
```

And the machinery that produces it, in the repository being migrated:

```
tests/golden/harness.php                    # the machinery
tests/golden/record.php                     # the recorder: boots the app, runs the cases
tests/Golden/BehaviourUnchangedTest.php     # the guarantee
```

Keep these under version control. They are the audit trail for the migration.

---

## Phase 0 — Discovery

**Goal:** know exactly what is being migrated before touching anything.

### 0.1 Runtime and stack inventory

```bash
php -v
php -m                       # extensions available on the CURRENT runtime
php -i | grep -E '^(Loaded Configuration|Scan this dir|date.timezone|error_reporting|serialize_precision)'
composer --version
```

Then inspect the repository:

```bash
# direct dependencies — the framework is usually one of them
composer show --direct 2>/dev/null || cat composer.json

# the layout
ls -1

# the tooling already in use
ls -1 composer.json composer.lock phpunit.xml* phpstan.neon* .php-cs-fixer* phpcs.xml* .github/workflows/ 2>/dev/null
```

Identify the framework from what `composer.json` requires and from the project
layout. Do not match against a fixed list of framework names — some projects
use none, some use several. `composer.json`'s `type` field narrows it down.

Record the exact version of each framework: `composer show <package>` for
Composer projects, the version constant in the source for the rest (e.g.
`wp_version` in `wp-includes/version.php`).

Record, in `docs/migration/00-baseline.md`:

| Field                                     | Value |
|-------------------------------------------|-------|
| Current PHP version                       |       |
| Target PHP version                        |       |
| Framework + version                       |       |
| Test runner + version                     |       |
| Static analysis tools                     |       |
| CI provider + config files                |       |
| Deploy mechanism (Docker? FPM? platform?) |       |
| Production PHP version(s) in use          |       |
| `composer.json` `require.php` constraint  |       |
| Test suite size (tests, assertions)       |       |
| Coverage %                                |       |
| Current failing tests (baseline)          |       |

### 0.2 Size the job

Before choosing the mode, see how much each step involves. The number of
sections on each migration page is in
[Appendix B](#appendix-b--official-sources) — that is the size of the work the
documentation describes.

How many of those this repo actually touches is what Phase 3 measures. Do not
estimate it from memory: read the page, probe the repo, count the hits.

### 0.3 Gate

- `docs/migration/00-baseline.md` written and complete.
- The user has confirmed the **target version** and the **mode** (stepwise / big-bang).
- If there are zero tests, say so loudly — Phase 4 will need characterization
  tests written from scratch, which changes the plan materially.

---

## Phase 1 — Environment

**Goal:** be able to run the codebase on the old and the new runtime, side by side.

### 1.1 Install the runtimes you need

Do not uninstall the current PHP. You need both.

```bash
# macOS (Homebrew): current stable is 8.5.x; versioned formulae for the ladder
brew install php@8.1 php@8.2 php@8.3 php@8.4 php
brew --prefix php          # e.g. /opt/homebrew/opt/php

# Linux / CI / any host: Docker is the most reproducible route
docker run --rm -v "$PWD":/app -w /app php:8.5-cli php -v
docker run --rm -v "$PWD":/app -w /app php:8.4-cli php -v
```

Use absolute paths to each binary when running different versions, e.g.

```bash
/opt/homebrew/opt/php@8.3/bin/php -v
/opt/homebrew/opt/php/bin/php -v        # 8.5
```

### 1.2 Branch

```bash
git checkout -b chore/php-upgrade-to-8.5
git status --porcelain      # must be empty before starting
```

If the working tree is dirty, stop and ask. Do not stash the user's work
without being told to.

### 1.3 Pin the platform in `composer.json`

Three keys, and they are committed:

- `require.php` — the range the project supports, e.g. `>=8.5 <8.6`.
- `config.platform.php` — the target PHP, and specifically the **lowest patch
  version you support**, not the newest one you tested with. The value is the
  floor for resolution: a dependency that requires `php >= 8.5.5` is acceptable
  when the platform says `8.5.12` and is not when it says `8.5.0`.
- `config.platform-check` — keep it enabled so the deployed app fails fast if
  the runtime does not match. It defaults to `php-only` since Composer 2.2; set
  it to `true` to also check the extensions.

**This is committed, and it stays.** `composer.json` and `composer.lock` travel
together: a lock file resolved under one platform, without that platform in
`composer.json`, means nothing to whoever reads it later.

`config.platform` makes Composer resolve dependencies **as if** running on the
target PHP, which is how you get a lock file that installs cleanly in
production.

Because the platform is faked, the local checks no longer tell the truth about
the real runtime. Run `composer check-platform-reqs` on the production image too
(Phase 8).

The one thing in this area that is **never** committed is
`--ignore-platform-req=php+` (2.2) — that one is a diagnostic.

### 1.4 Verify the install

```bash
composer install
composer check-platform-reqs
```

### 1.5 Gate

- Old runtime and new runtime both run `php -v` and `php -m`.
- `composer install` completes with no platform errors.
- `docs/migration/01-environment.md` records the exact binary paths used.

---

## Phase 2 — Dependency matrix

**Goal:** make every dependency support the target PHP before touching
application code.

### 2.1 Find what blocks the target

```bash
composer why-not php 8.5            # alias of `composer prohibits`
composer why-not php 8.5 --locked   # against the locked versions
```

The output lists every package whose constraint excludes PHP 8.5, with the
versions that would work. Work top-down: a package that cannot run on 8.5 at all
must be replaced, not patched.

### 2.2 Diagnostic-only escape hatch

If a package only caps the **upper** bound of PHP (e.g. `"php": "^7.4"`), you can
inspect the resolution without committing to it:

```bash
composer update --ignore-platform-req=php+ --dry-run
```

`--ignore-platform-req=php+` ignores only the PHP upper bound. Never commit it,
never put it in CI, and treat the result as a diagnostic: it tells you the
package would probably work, not that it does.

### 2.3 Upgrade in a controlled way

```bash
# raise the root constraint first
composer require "symfony/symfony:^6.4" --no-update
composer update --with-all-dependencies     # -W updates root requirements too
composer audit --locked
```

Upgrade order that minimizes churn: PHP constraint → framework → major
dependencies → minor dependencies.

### 2.4 Verify

```bash
composer validate
composer check-platform-reqs
composer install
composer audit --locked
```

### 2.5 Gate

- `composer why-not php 8.5` no longer lists any package that must be replaced.
- `composer install` and `composer check-platform-reqs` both pass.
- `composer audit --locked` reports no known vulnerabilities in required packages.
- `docs/migration/02-dependencies.md` records each upgrade decision and why.

---

## Phase 3 — Documentation-driven audit

**This is the heart of the migration.** For each ladder step, go through the
official PHP documentation **item by item** and decide, with evidence, whether
that item applies to this repository. Nothing is changed in this phase.

### 3.1 Fetch the authoritative pages

For the step `X.Y → Z.W`, fetch:

| Document                      | URL                                                          |
|-------------------------------|--------------------------------------------------------------|
| Backward Incompatible Changes | `https://www.php.net/manual/en/migrationZW.incompatible.php` |
| Deprecated Features           | `https://www.php.net/manual/en/migrationZW.deprecated.php`   |

Full per-version URL table: [Appendix B](#appendix-b--official-sources).

Fetch them with whatever tool the harness provides — a dedicated web-fetch
tool, or plain shell:

```bash
curl -sL https://www.php.net/manual/en/migration80.incompatible.php -o /tmp/migration80.incompatible.html
```

Keep the fetched pages (or their markdown conversion) next to the audit file
for that step, e.g. `docs/migration/audit/7.4-8.0/migration80.incompatible.md`,
so a reviewer can re-read exactly what you read. If the environment has no
network access, ask the user for the page contents before continuing — the
audit is only valid against the real documentation.

### 3.2 Enumerate the items

Each page is a list of sections. A section is titled after an extension or
topic (`CURL`, `GD`, `PHP Core`, `Standard`) and holds either subsections
titled after one specific change (`String to Number Comparison`, `Other
incompatible Changes`), or a bulleted list of changes.

The auditable items are the subsections and the bullets. Do not summarize or
skip items — an item you skipped is an item you did not verify.

Each item has a permanent link, and the audit must cite it:

- the id is in the HTML, in the `id` attribute of the section's wrapper
  (`<div class="sect3" id="migration80.incompatible.core.other">`);
- ids are hierarchical — `a.b.c` sits under `a.b` — so cite the longest id
  that covers the finding;
- the bullets inside a section share that section's id, so probe each bullet
  separately and cite the section.

Open the link you wrote. If it does not resolve, the id is wrong.

### 3.3 Probe each item against the repository

Use the probe that matches the item's category. Every probe must produce output
that can be pasted into the audit table as evidence — including the empty result.

| Item category                                                                   | Probe                                                                                                                                  | Evidence to capture                   |
|---------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------|
| Removed / renamed function, class, constant                                     | `grep -rnE '\bSYMBOL\s*\(' --include='*.php' .` **and** `php -r 'var_dump(function_exists("SYMBOL"));'` against the **target** binary  | file:line hits, or "0 hits" + `false` |
| Changed signature / return type                                                 | AST search: PHPStan (see Phase 6) + `grep` for the call sites                                                                          | list of call sites                    |
| Removed or changed INI directive                                                | `grep -rn 'DIRECTIVE' php.ini .htaccess docker* Dockerfile* .github/ 2>/dev/null` + `php -i \| grep DIRECTIVE` on old and new runtimes | where it is set                       |
| Removed extension                                                               | `php -m` on target vs `ext-*` in `composer.json` vs actual usage of the extension's functions                                          | extension list + usage                |
| Removed language syntax                                                         | `find src tests -name '*.php' -exec php -l {} +` using the **target** binary                                                           | parse-error list                      |
| Silent semantic change (comparison, sorting, escaping, numeric strings, locale) | pattern search + runtime probe (see 3.5)                                                                                               | pattern list + probe output           |
| Resource → object migration                                                     | `grep -rn 'is_resource(' --include='*.php' .` + the affected extension calls                                                           | call-site list                        |
| Warning → exception change                                                      | runtime probe that triggers the path                                                                                                   | probe output                          |
| Error-model change (new exception classes)                                      | `grep -rnE 'catch\s*\(\s*(Exception\|Error\|Throwable)' --include='*.php' .`                                                           | catch sites that may no longer match  |

A powerful generic probe for "does this function/method still exist and behave
the same" is:

```bash
php -r 'var_dump(function_exists("the_function"), (new ReflectionFunction("the_function"))->getNumberOfParameters());'
```

Run it with the **target** binary. A `false` plus a non-zero grep hit count
means the item applies.

### 3.4 Verdicts

Every item gets exactly one verdict:

| Verdict         | Meaning                                                                                                                                                |
|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| `APPLIES`       | The feature is reachable in this repo. Evidence: file:line.                                                                                            |
| `CONDITIONAL`   | Applies only under specific runtime configuration (an INI value, an SAPI, an extension being loaded). Record the condition and where it is configured. |
| `N/A`           | The feature is not used. Evidence must be the probe output showing zero hits.                                                                          |
| `ALREADY-FIXED` | The repo already migrated this item. Evidence: where.                                                                                                  |
| `ACCEPTED-DIFF` | The behavior changes and the change is intended. Requires a test pinning the new behavior.                                                             |

`N/A` without evidence is not a verdict. "I did not find it" is a hypothesis,
not a fact.

### 3.5 Runtime probes for silent changes

Some items cannot be found by grep — they change the behavior of code you
already have. Probe them explicitly.

Write one probe per audit item that needs one. A good probe:

- calls the function with the inputs that hit the changed behavior — the edge
  cases, not the happy path: malformed UTF-8, `null`, the empty string,
  numeric strings, the last element of an array;
- prints the result **with its type**, so `false`, `"false"` and `0` are never
  confused;
- is deterministic: same output every run — no timestamps, no random values,
  no ids;
- prints nothing else, so the diff shows only what changed.

Run it on the old runtime and on the new one. Every difference is a candidate
`ACCEPTED-DIFF`: record it in the audit **before** you change any code.

Keep the probe. In Phase 4 it becomes a case in the recorder (4.4), so it keeps
running for the rest of the migration.

### 3.6 Items that change what the code means

Most items change what the code *does*. A few change what it *means*, and those
are the ones that bite, because the fix is not mechanical.

The clearest case is a comparison between a number and a string whose type you
do not control. Since PHP 8.0, a number compared loosely with a non-numeric
string is compared as a string: `0 == "foo"` and `0 == ""` were `true`, now
they are `false`. The same applies to `!=`, `<` and `<=`. Since PHP 8.5, an
object compared loosely with a boolean held in a variable (`$object == $flag`)
always returned `false`; now it follows `(bool) $object`.

**There is no single correct edit.** What the branch was trying to ask decides
the change:

| The branch meant                              | The change                                                                                                                |
|-----------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| "is this the number zero, whatever its type?" | cast on purpose: `(float) $var === 0.0`, or `$var === 0 \|\| $var === "0"`                                                |
| "is this value empty?"                        | `$var === ""` — and decide whether `null` and `"0"` count                                                                 |
| "is this value present?"                      | `$var !== null && $var !== ""` — and decide about `"0"`                                                                   |
| "is this a number other than zero?"           | `is_numeric($var) && (float) $var !== 0.0`                                                                                |
| "is this truthy?"                             | spell out which values count as false — [the manual lists them](https://www.php.net/manual/en/language.types.boolean.php) |

**The intent is not in the code.** Read the caller, the tests, the git history,
the ticket. If you cannot tell what the branch meant, ask the user. Do not
guess, and do not reach for `===` and move on: a wrong guess is a silent
change in production, and no test catches it, because the test would encode
the guess.

Two things make it safer:

- **Probe the values the branch really sees.** Not `0` and `""` in the
  abstract — the string a form sends, the value a database column returns, the
  result of a `json_decode()` that failed.
- **Write the intent into the audit row**, next to the action. One sentence:
  what the branch means and why. That is what lets a reviewer tell a
  considered change from an accident.

The doc items where this bites:
[PHP 8.0 — String to Number Comparison](https://www.php.net/manual/en/migration80.incompatible.php#migration80.incompatible.core.string-number-comparision)
and
[PHP 8.5 — Loosely comparing uncomparable objects](https://www.php.net/manual/en/migration85.incompatible.php#migration85.incompatible.core.loosely-comparing-object).

### 3.7 The audit table

One file per ladder step: `docs/migration/audit/<from>-<to>.md`.

```markdown
# Audit: PHP 7.4 → 8.0

Source: https://www.php.net/manual/en/migration80.incompatible.php
        https://www.php.net/manual/en/migration80.deprecated.php
Doc items: 46 (39 incompatible + 7 deprecated)

| # | Doc item                                    | Source link                                                                                              | Probe                                               | Evidence                      | Verdict       | Action                                           | Test                               | Status |
|---|---------------------------------------------|----------------------------------------------------------------------------------------------------------|-----------------------------------------------------|-------------------------------|---------------|--------------------------------------------------|------------------------------------|--------|
| 1 | `implode()` reversed argument order removed | [standard](https://www.php.net/manual/en/migration80.incompatible.php#migration80.incompatible.standard) | `grep -rn "implode(" src/`                          | src/Report.php:88             | APPLIES       | A-01 swap to `implode($glue, $parts)`            | tests/Unit/ReportTest::testImplode | TODO   |
| 2 | `each()` removed                            | [other](https://www.php.net/manual/en/migration80.incompatible.php#migration80.incompatible.core.other)  | `grep -rnE "\beach\s*\(" src/`                      | 0 hits                        | N/A           | —                                                | —                                  | CLOSED |
| 3 | `htmlspecialchars()` default flags          | [standard](https://www.php.net/manual/en/migration81.incompatible.php#migration81.incompatible.standard) | `grep -rn "htmlspecialchars(" src/` + runtime probe | src/View.php:31, probe output | ACCEPTED-DIFF | A-02 pin `ENT_QUOTES\|ENT_SUBSTITUTE` explicitly | tests/Unit/ViewTest::testEscaping  | TODO   |
```

Rules for the table:

- **Every row cites a doc source.** A row without a source link is invalid. The
  link is the page URL plus the anchor of the section that documents the item
  — take the id from the page HTML (3.2).
- The row count must equal the number of doc items enumerated in 3.2. Keep the
  count in the file header so a reviewer can verify completeness.
- `N/A` rows keep their probe output (collapsed in a `<details>` block if long).
- `APPLIES` and `ACCEPTED-DIFF` rows get an `Action` id (`A-01`, `A-02`, …).

### 3.8 The action plan

`docs/migration/action-plan.md` — one task per `APPLIES` / `CONDITIONAL` /
`ACCEPTED-DIFF` row.

```markdown
| Id   | Audit row | Area           | Affected files    | Change                                           | Test that proves it                      | Status |
|------|-----------|----------------|-------------------|--------------------------------------------------|------------------------------------------|--------|
| A-01 | 5.6-7.0#3 | src/Report.php | src/Report.php:88 | `implode($parts, ',')` → `implode(', ', $parts)` | tests/Unit/ReportTest::testImplode (new) | TODO   |
```

Order the plan by risk × effort: language-level removals first (they are parse
errors and stop everything), then semantic changes (silent bugs), then
deprecations (warnings), then framework/config items.

### 3.9 Gate

- Every doc item for the step has a row with a verdict and evidence.
- `docs/migration/action-plan.md` exists and covers every non-`N/A` row.
- The user has reviewed the plan and its size. Report the item counts per
  verdict — that number is the real size of the migration.

---

## Phase 4 — Baseline & characterization

**Goal:** freeze the current behavior so you can prove it did not change.

This phase changes no production code.

### 4.1 Test baseline

```bash
composer test 2>&1 | tee docs/migration/golden/baseline-tests.txt
```

Record:
- total tests, assertions, failures, errors, skipped;
- the **exact list** of failing tests (they are pre-existing; you are not
  responsible for them, but you must not add to them);
- runtime warnings and deprecations already emitted.

### 4.2 Coverage baseline

```bash
vendor/bin/phpunit --coverage-clover docs/migration/golden/baseline-coverage.xml
```

Keep the XML. Later you diff per-file coverage to prove no new dead code was
introduced.

### 4.3 Runtime configuration baseline

```bash
php -i > docs/migration/golden/baseline-php-info.txt
php -m  > docs/migration/golden/baseline-modules.txt
```

For every `CONDITIONAL` audit row, copy the relevant INI values here.

### 4.4 Record what the code does

The guarantee you need: the same functions return the same values, and the
application's globals end up in the same state, before and after every change.

Make it a **test**, not a manual diff. A test that fails on its own is the only
thing that still holds three steps later.

This is the project's own code, written in the project's own test framework.
The file names below are a convention — keep them, or follow whatever the
project already uses. Four pieces:

| Piece    | File                                      | Job                                                         |
|----------|-------------------------------------------|-------------------------------------------------------------|
| Harness  | `tests/golden/harness.php`                | The machinery: renders values, collects globals and statics |
| Recorder | `tests/golden/record.php`                 | Boots the app, runs the cases, prints what happened         |
| Contract | `docs/migration/golden/*.golden.txt`      | That output, committed                                      |
| Test     | `tests/Golden/BehaviourUnchangedTest.php` | Fails when the output no longer matches                     |

The recorder prints two records:

- **behaviour** — the result of every case, the application's globals, the
  static properties. This one must stay **identical** for the whole migration.
- **api** — the PHP surface: every internal function, the loaded extensions, the
  ini values. This one is *meant* to change at every version step. The diff
  between the old and the new one **is** the list of what the new version took
  away.

Two rules: the behaviour must not change; the api change must be explained.

#### The interface

Whatever the code looks like, the recorder answers to the same three commands,
because the test and the audit both call it:

```bash
php tests/golden/record.php behaviour   # print the behaviour record
php tests/golden/record.php api         # print the PHP surface
php tests/golden/record.php --write     # write both golden files
```

A **case** is a named piece of code that returns a value. One case per behavior
family. The record prints the case name and the value.

#### The rules that keep the record honest

- **Bootstrap the application first**, at global scope, or you record an empty
  shell.
- **Name everything the recorder defines with a leading `__`**, or it shows up
  as an application global and pollutes the record.
- **A case must be deterministic.** No timestamps, no random values, no ids.
  Assert on the shape (`strlen(random_bytes(8))`), not on the value.
- **Skip the superglobals.** They hold the environment and the request, not the
  application.
- **Pin what the runtime would otherwise decide**: `serialize_precision` to
  `-1`, the timezone, the locale. Otherwise the way a float is written depends
  on the runtime, and the record differs for no reason.
- **Print the value with its type**, so `false`, `"false"` and `0` are never
  confused.

#### What the test does

- It runs the recorder as a subprocess, with the same PHP binary, and compares
  the output with the committed file, byte for byte. Running it in-process
  instead would give a different scope, and the comparison would mean nothing.
- On a mismatch in the behaviour record it fails with the **exact lines that
  differ**, not a generic "not equal".
- On a mismatch in the api record it lists what was **removed** and what was
  **added**, and says the file is regenerated on purpose at every version step.
- It never regenerates the golden file to make itself pass. A difference with
  no audit row is a regression — see 5.2.

#### Generating the contract

Run the recorder once on the **current** runtime and commit the result. From
then on the test *is* the contract.

### 4.5 Characterization tests for untested behavior

If a behavior family has no test, write one **before** changing anything: a
test that asserts the **exact** value the code produces today — not a loose
comparison, not a type check.

A characterization test pins the *current* behavior. If it later fails after a
migration change, that change either broke something or needs an explicit
`ACCEPTED-DIFF` row. Either way you find out.

A case in the recorder (4.4) is the same thing, but it compares the whole run
instead of one value. Prefer the recorder for behavior families: one case per
family, one file, one assertion.

### 4.6 Gate

- Baseline test output and coverage XML stored under `docs/migration/golden/`.
- `docs/migration/golden/behaviour.golden.txt` and `api.golden.txt` exist, were
  produced by the **current** runtime, and are committed.
- `tests/Golden/BehaviourUnchangedTest.php` passes on the current runtime.
- Every risky behavior family from the audit has a case in the recorder or a
  characterization test.
- The suite is green (or has a known, recorded set of pre-existing failures).

---

## Phase 5 — Execute the action plan

**Goal:** implement every action, proving each one safe.

### 5.1 Per-item protocol

For each action-plan row, in order:

1. **Read the audit row again.** Open the doc item. Do not work from memory.
2. **Write the test first.**
   - For a behavior change: assert the *new documented* behavior. Run it — it
     must fail (red) for the expected reason.
   - For a pure refactor: do not touch the test. The existing test must keep
     passing (green) both before and after.
3. **Apply the change.** Smallest possible diff.
4. **Run the focused test.** Green.
5. **Run the full suite.** No new failures.
6. **Run the golden test.** `tests/Golden/BehaviourUnchangedTest.php` must pass:
   the behaviour record has to be byte-identical, and any difference in the api
   record has to be explained in the audit.
7. **Update the action plan status** to `DONE` and commit.

```bash
vendor/bin/phpunit --filter testEscaping
composer test
vendor/bin/phpunit tests/Golden/BehaviourUnchangedTest.php
```

When a fix has to run on **both** the old and the new version — the usual
case during the ladder — write it so it works on both, and prefer asking the
environment over asking the version:

- ask whether the feature is there: `function_exists('json_validate')`,
  `extension_loaded('intl')`, `class_exists('SomeClass')`;
- ask the version only when there is no feature to detect:
  `version_compare(PHP_VERSION, '8.5.0', '>=')` or `PHP_VERSION_ID >= 80500`
  (`PHP_VERSION_ID` is `major*10000 + minor*100 + release`).

A version check answers "is the feature there?" the wrong way, and it goes
stale the moment the project supports one more version.

### 5.2 The equivalence rule

> **Every difference in the golden records must map 1:1 to an `ACCEPTED-DIFF`
> row in the audit.**

A difference with no audit row is a **regression**. When that happens:

1. Stop. Do not regenerate the golden files.
2. Find the cause: usually an undocumented side effect, a dependency change, or
   a different code path.
3. Either fix the code so the behavior is preserved, or promote the item to an
   `ACCEPTED-DIFF` audit row with a citation to the documentation and a test
   pinning the new behavior.
4. Record the reasoning in the migration report.

Never regenerate the golden files to make the failure disappear. That is the
single fastest way to ship a silent regression.

### 5.3 Acceptance gate for one item

All four must hold:

- [ ] Pre-existing tests still pass **unchanged** (except tests explicitly
      updated for this item, each linked to an `ACCEPTED-DIFF` row).
- [ ] A new or updated test pins the new behavior.
- [ ] Coverage does not drop below baseline for the touched files.
- [ ] The golden test passes: the behaviour record is identical, and every
      difference in the api record is explained by an audit row.

### 5.4 Using the automated tools (without skipping the audit)

Rector and PHPCompatibility are **amplifiers**, not substitutes for the audit.
They may only be applied to items the audit already identified.

**Rector** — the Composer package is `rector/rector` (the GitHub repository is
`rectorphp/rector`).

1. Write a `rector.php` that targets the **new** PHP version and covers the
   project's own paths. The 2.x config is a fluent chain built on
   `RectorConfig::configure()`; the calls to look for are `withPaths()`,
   `withPhpSets()`, `withPhpLevel()`, `withPreparedSets()` and `withSkip()`.
   **Read the tool's own docs for the version you install** — the calls and the
   set names change between releases.
2. Enable the sets the **audit rows** ask for, not the ones that look good. A
   set that rewrites code nobody reviewed is exactly what this phase forbids.
3. On a large legacy codebase, raise the level one step at a time
   (`withPhpLevel()`) instead of taking the whole set.
4. Preview, then apply:

```bash
vendor/bin/rector process --dry-run --clear-cache    # preview the diff
vendor/bin/rector process                            # apply
```

`--dry-run` exits non-zero when it would change files, so it doubles as a CI
gate.

Know its limits before you trust the output:

- It resolves types with PHPStan, so a PHPStan baseline makes it blind.
- It does not boot the framework or the container, so magic methods and
  runtime-generated classes are invisible to it.
- It does not understand Twig or Blade templates.
- It does not format code.

On a large legacy codebase, get PHPStan to level 3–4 **without** a baseline
before trusting Rector's output.

Run Rector **after** the manual audit rows are fixed, then re-run the audit
probes: Rector may reveal additional call sites that the grep probes missed.

**PHPCompatibility** — a PHP_CodeSniffer standard that detects usage of
features that do not exist in the target version.

```bash
composer config allow-plugins.dealerdirect/phpcodesniffer-composer-installer true
composer require --dev phpcompatibility/php-compatibility
vendor/bin/phpcs -i     # must list PHPCompatibility
```

Two things to get right. **Allow the plugin first**: without it the standard
never registers and the tool reports nothing — which looks exactly like "the
code is clean". And **install the release line that knows about the target
version**: the stable one dates from 2019 and predates PHP 8 entirely, so the
PHP 8 knowledge lives in the 10.x development line.

```bash
vendor/bin/phpcs -ps src tests \
  --standard=phpcs.xml.dist \
  --runtime-set testVersion 5.6-8.5
```

The `testVersion` range is the versions the code still has to support; a
reference to something that does not exist in that range is the finding.

**Coding standard** — if the project uses PHP-CS-Fixer, run it after the
rewrites. It has one migration ruleset per PHP version, named after it
(`@PHP85Migration` and the earlier ones), so the fixer can apply the same
syntax the target version allows.

### 5.5 Gate for the whole step

- Every action-plan row is `DONE` or explicitly `DEFERRED` (with a reason).
- The full suite is green (plus the recorded pre-existing failures only).
- Coverage ≥ baseline.
- All golden diffs are explained.
- The step is committed and the branch is green.

---

## Phase 6 — Type hardening

Optional, and its own plan. Do not mix it into the migration commits.

Target: `declare(strict_types=1)` in every entry point, typed properties, typed
return types, explicit nullability, and the replacement of loose comparisons.

### 6.1 Strict types

Add `declare(strict_types=1);` as the first statement of the file, file by
file, starting with leaf/utility code, and fix what it breaks.
`declare(strict_types=1)` affects calls **made from** that file, not calls into
it, so adding it to one file changes the behavior of that file's calls only —
which makes incremental adoption tractable.

### 6.2 Loose comparison audit

The PHP 8.0 change to loose comparison is the most common silent regression in
migrations. Audit `==` explicitly:

```bash
grep -rnE '[^=!<>]==[^=]' --include='*.php' src | grep -vE '===' | head -100
grep -rnE '!=[^=]' --include='*.php' src | head -100
```

For each hit decide explicitly: is `==` intended (type juggling) or accidental?

**Do not replace `==` with `===` by reflex.** When the right side is a literal
and the left side is an internal value, a comparison against an unexpected
type is usually a bug, and `===` is the fix. But when the value comes from
outside — a form field, a database column, a JSON payload — `===` may change
what the branch means instead of repairing it.

That case is not a mechanical fix. Read the branch, work out what it meant,
and change that: 3.6. Add a characterization test before you touch it.

### 6.3 Static analysis as the safety net

Point PHPStan at the project's own `src` and `tests`, at the highest level, and
at the target version. Two settings matter for a migration:

- `phpVersion` — the version PHPStan should assume, in the `major*10000 +
  minor*100 + release` form (`80500` for 8.5.0). Without it, PHPStan keeps
  accepting constructs the target version removed.
- `treatPhpDocTypesAsCertain` — set it to `false`, so PHPStan does not trust a
  `@param` that the code does not enforce.

Exclude the folders you are not migrating yet, one path per entry, and keep the
exclusions shrinking as the migration advances.

```bash
vendor/bin/phpstan analyse --no-progress
```

Do **not** introduce a baseline to make it green during a migration. A baseline
hides exactly the type errors the migration is trying to surface. If you need
one for legacy code, keep it in a separate, clearly named file and track it as
an action-plan item with an owner.

Two Composer packages earn their place here: one that flags usage of
`@deprecated` API, and one that provides the symbol map for the target
version, so PHPStan sees the functions and classes that exist there.

### 6.4 Gate

- `declare(strict_types=1)` in all entry points.
- PHPStan at the target level with no baseline (or an explicitly accepted,
  tracked baseline).
- No new loose comparisons.

---

## Phase 7 — Test & CI matrix

**Goal:** the project stays green on every PHP version it claims to support.

### 7.1 PHPUnit configuration

Take the project's existing `phpunit.xml` — or create `phpunit.xml.dist` — and
make it fail on everything that matters while migrating:

| Setting                          | Value                    | Why                                          |
|----------------------------------|--------------------------|----------------------------------------------|
| `failOnDeprecation`              | `true`                   | a deprecated API call fails the build        |
| `failOnPhpunitDeprecation`       | `true`                   | a deprecated PHPUnit feature fails the build |
| `failOnWarning`                  | `true`                   | a warning fails the build                    |
| `beStrictAboutOutputDuringTests` | `true`                   | output during a test is an error, not noise  |
| `error_reporting` (ini)          | `-1`                     | everything is reported, nothing is filtered  |
| `bootstrap`                      | the project's autoloader | the tests load the application               |

For Symfony, also set the `SYMFONY_DEPRECATIONS_HELPER` env to
`max[total]=0` — zero deprecations tolerated.

`failOnDeprecation` + `error_reporting=-1` turns every deprecation into a test
failure — which is exactly what you want while migrating, so that no deprecated
API survives the migration unnoticed.

If the project still has an old PHPUnit config, migrate it rather than hand-rewriting it:

```bash
vendor/bin/phpunit --migrate-configuration
```

### 7.2 CI matrix

One job per supported version, in the CI the project already uses. On GitHub
Actions, `shivammathur/setup-php@v2` covers PHP 5.4 to 8.5 — it takes the PHP
version, the extensions, the coverage driver and `tools: composer:v2`.

**Pin the PHPUnit version to the PHP version of the job.** PHPUnit 13 needs PHP
8.4 or later, 12 needs 8.3, 11 needs 8.2. `setup-php` can pick it for you.

Each job runs:

```bash
composer install --no-progress --no-interaction
vendor/bin/phpunit --fail-on-deprecation --fail-on-warning
vendor/bin/phpstan analyse --no-progress
vendor/bin/phpcs -ps src tests --standard=phpcs.xml.dist --runtime-set testVersion 5.6-<the php version of the job>
vendor/bin/rector process --dry-run --clear-cache
```

Two details worth keeping:

- Install with `composer update --prefer-stable --prefer-lowest
  --with-all-dependencies` in the job that tests the oldest dependency set, so
  the floor is tested and not just the latest.
- Set `error_reporting=-1` in the job, so the deprecations the tests do not
  catch still show up in the log.

`rector process --dry-run` exits non-zero when it would change files, so it acts
as a "no pending automated migrations" gate.

### 7.3 Deprecations as errors at runtime

For non-Symfony projects, a small bootstrap turns deprecations into
exceptions: install an error handler that throws on `E_DEPRECATED` and
`E_USER_DEPRECATED`, and let every other severity through to PHP's own
handler.

With `symfony/error-handler` you additionally get `DebugClassLoader`, which
reports usage of `@deprecated` / `@internal` API that never triggers a runtime
notice. It is turned on by calling the `Debug` class of that package
(`Symfony\Component\ErrorHandler\Debug`) and its `enable()` method, early in
the bootstrap.

### 7.4 Gate

- CI green on every version in the matrix.
- Zero deprecations in the test output.
- Coverage at or above the Phase 4 baseline.

---

## Phase 8 — Deploy & rollback

**Goal:** change the runtime in production without an outage.

### 8.1 Pre-flight

- [ ] `composer check-platform-reqs` passes on the production image.
- [ ] `config.platform` in `composer.json` matches the **lowest** PHP version
      the production image runs.
- [ ] The production image pins the PHP version explicitly
      (`FROM php:8.5.12-fpm-alpine`, not `php:fpm`).
- [ ] Extensions required by `composer.json` (`ext-*`) are present in the image.
- [ ] CI built the exact artifact that will be deployed.
- [ ] A rollback artifact from the previous release is available.

### 8.2 Runtime behavior changes to check in production

A reminder, not a list to work through: the complete set of version-specific
changes comes from the audit and from the deprecation notes of the version you
deployed. These are the infrastructure-level items a code audit tends to miss.

| Area                 | Change                                                                    | Action |
|----------------------|---------------------------------------------------------------------------|---------------------------------------------------------------------------------------|
| Password hashing     | `PASSWORD_BCRYPT` default cost raised 10 → 12 in 8.4                      | Verify login still works; existing hashes are unaffected (cost is stored in the hash) |
| Opcache              | Built into the binary and always loaded in 8.5                            | Remove `zend_extension=opcache.so` from ini — it now warns                            |
| JIT                  | `opcache.jit` default changed to `disable`, buffer to 64M                 | Set explicitly if you rely on JIT                                                     |
| `disable_classes`    | INI removed in 8.5                                                        | Remove from ini files                                                                 |
| `htmlspecialchars()` | Default flags changed in 8.1                                              | Verify rendered HTML and any stored/escaped data                                      |
| Session IDs          | `session.sid_length` / `sid_bits_per_character` changes deprecated in 8.4 | Move to 32-char hex IDs; invalidate old sessions deliberately                         |
| Intl                 | Requires ICU ≥ 57.1                                                       | Confirm the base image ships a compatible ICU                                         |
| `date.timezone`      | Unconfigured apps get UTC                                                 | Set `date.timezone` explicitly in the image                                           |
| CLI                  | `-z` / `--zend-extension` removed in 8.5                                  | Use `-d zend_extension=...`                                                           |

### 8.3 Rollout

1. **Canary** — deploy to a small slice (one instance, or 5% traffic).
2. **Soak** — watch error rate, latency, and deprecation logs for a meaningful
   window. Compare against the pre-deploy baseline.
3. **Ramp** — 25% → 50% → 100%.
4. **Hold** — keep the previous version deployable for at least one release
   cycle.

### 8.4 Rollback

The rollback must be a *version* rollback, not a code rollback. Keep the previous
PHP image tagged and deployable, and keep database migrations backward
compatible (expand/contract) for the duration of the soak window.

### 8.5 Gate

- [ ] Canary healthy for the agreed window.
- [ ] Error rate and latency at or better than the pre-migration baseline.
- [ ] No deprecation warnings in production logs.
- [ ] Rollback rehearsed (documented, and actually tried once).

---

## Phase 9 — Cleanup & report

### 9.1 Remove the scaffolding

- [ ] Remove compatibility shims that are no longer reachable.
- [ ] Remove `composer.json` `config.platform` if you no longer need to fake the
      platform — or keep it, but make it match production exactly.
- [ ] Remove temporary `--ignore-platform-req` usage from CI (it must not be there).
- [ ] Remove Rector/PHPCompatibility dev dependencies **only if** you are not
      keeping them as permanent gates. Recommended: keep them; they are cheap
      insurance.
- [ ] Delete temporary probes that were only used for the audit, unless they
      were promoted to regression tests.
- [ ] Update `README.md`, `CONTRIBUTING.md`, and the changelog with the new
      minimum PHP version (if files exists in the repository).

### 9.2 Backward compatibility of your own public API

If the project is a library:

```bash
vendor/bin/roave-backward-compatibility-check --format=github-actions
```

This compares the public API against the last tagged release and fails on any
breaking change. It needs git tags in `x.y.z` form and `fetch-depth: 0` in CI.

### 9.3 The final report

`docs/migration/MIGRATION-REPORT.md`:

```markdown
# PHP Migration Report

- From: PHP 5.6.40
- To: PHP 8.5.12
- Mode: stepwise
- Date: 2026-03-11

## Audit summary
| Step      | Items audited | APPLIES | CONDITIONAL | N/A | ALREADY-FIXED | ACCEPTED-DIFF |
|-----------|---------------|---------|-------------|-----|---------------|---------------|
| 5.6 → 7.0 | 58            | 31      | 2           | 24  | 1             | 3             |
| ...       |

## Behavior changes accepted
| Item | Old behavior | New behavior | Justification | Test |
|------|--------------|--------------|---------------|------|
| ...  |

## Golden diffs
Every diff listed here maps to an ACCEPTED-DIFF row.

## Test results
Baseline: N tests, M assertions, K failures.
Final:    N tests, M assertions, K failures.
Coverage: X% → Y%.

## Deferred / follow-up
| Item | Reason | Owner | Tracking |
|------|--------|-------|----------|
| ...  |

```

### 9.4 Gate

- [ ] All phases closed.
- [ ] Report written and reviewed.
- [ ] `composer.json` `require.php` reflects the new minimum version.
- [ ] Production is on the new version and stable.

---

## Appendix A — Error → cause → fix

Not a checklist. A few traps where the obvious fix is wrong; for everything else
use Appendix B and the versioned migration pages.

| Symptom                                                                 | Trap                                 | Fix                                                                             |
|-------------------------------------------------------------------------|--------------------------------------|---------------------------------------------------------------------------------|
| `0 == "foo"` now `false`                                                | Swapping `==` for `===` by reflex    | Decide what the branch meant — see 3.6                                          |
| `PDOException` / `mysqli_sql_exception` where code checked `=== false`  | Silencing or forcing old error modes | Catch the exception, or set `ATTR_ERRMODE` / `mysqli_report` explicitly         |
| Values compared with `===` no longer match after DB fetch               | Re-enabling `ATTR_STRINGIFY_FETCHES` | Cast explicitly; native types are the contract since 8.1                        |
| `TypeError: implode(): Argument #1 ($separator) must be of type string` | Guessing argument order              | `implode($glue, $parts)` — glue first                                           |
| `TypeError: Cannot unpack array with string keys`                       | Stripping keys blindly               | Named args explicitly, or pass a list                                           |
| `Deprecated: Passing null to parameter #N of type string`               | `@` or ignoring the warning          | `(string) $x` (or fix the null source)                                          |
| `Deprecated: Creation of dynamic property`                              | Blanket `#[\AllowDynamicProperties]` | Declare the property; attribute only when the class is intentionally open       |
| HTML now contains `&#039;` / malformed UTF-8 becomes `�`                | Changing only the test expectation   | Pass `ENT_QUOTES \| ENT_SUBSTITUTE` explicitly and update expectations together |

---

## Appendix B — Official sources

Always re-fetch rather than trusting a cached copy. The manual is versioned and
these pages are stable per release.

### Migration guides

The section count is how many sections the page has. It is the size of that
step's audit: if your audit has fewer rows than the count, you skipped
something.

| Step      | Incompatible                   | Deprecated                   | Sections (incompat. / deprec.) |
|-----------|--------------------------------|------------------------------|--------------------------------|
| 5.6 → 7.0 | `migration70.incompatible.php` | `migration70.deprecated.php` | 59 / 5                         |
| 7.0 → 7.1 | `migration71.incompatible.php` | `migration71.deprecated.php` | 25 / 2                         |
| 7.1 → 7.2 | `migration72.incompatible.php` | `migration72.deprecated.php` | 21 / 14                        |
| 7.2 → 7.3 | `migration73.incompatible.php` | `migration73.deprecated.php` | 17 / 10                        |
| 7.3 → 7.4 | `migration74.incompatible.php` | `migration74.deprecated.php` | 22 / 22                        |
| 7.4 → 8.0 | `migration80.incompatible.php` | `migration80.deprecated.php` | 39 / 7                         |
| 8.0 → 8.1 | `migration81.incompatible.php` | `migration81.deprecated.php` | 16 / 24                        |
| 8.1 → 8.2 | `migration82.incompatible.php` | `migration82.deprecated.php` | 5 / 7                          |
| 8.2 → 8.3 | `migration83.incompatible.php` | `migration83.deprecated.php` | 15 / 14                        |
| 8.3 → 8.4 | `migration84.incompatible.php` | `migration84.deprecated.php` | 44 / 20                        |
| 8.4 → 8.5 | `migration85.incompatible.php` | `migration85.deprecated.php` | 37 / 28                        |

Base: `https://www.php.net/manual/en/`

Every section has its own permanent link. Take the id from the page HTML and
cite it — see 3.2.

### Other authoritative sources

- `https://github.com/php/php-src/blob/PHP-8.5/UPGRADING` — the canonical
  change list per branch, including notes the manual omits.
- `https://wiki.php.net/rfc` — the RFCs behind each change.
- `https://www.php.net/releases/index.php` — current supported releases and
  end-of-life dates.
- `https://www.php.net/manual/en/migration85.incompatible.php` for the current
  stable release.

> PHP 6 was abandoned; there is no `migration6.*` page. The manual jumps from
> 5.6 to 7.0.

---

## Appendix C — Framework notes

Notes for the frameworks that break most often. **Not a framework registry** —
if the project runs another one, that framework's own upgrade guide belongs in
the audit, in the same files and under the same evidence rules as the PHP
documentation.

### C.1 Symfony

- Symfony 4.4 LTS supports PHP 7.1–8.x; Symfony 5.4 LTS supports PHP 7.2.5–8.x;
  Symfony 6.4 LTS requires PHP ≥ 8.1; Symfony 7.x requires PHP ≥ 8.2.
- `symfony/debug-bundle`, `symfony/profiler` and `symfony/web-profiler` are
  development-only dependencies — they must not be in `require` for production.
- `symfony/phpunit-bridge` provides `simple-phpunit` and the
  `SYMFONY_DEPRECATIONS_HELPER` mechanism. `max[total]=0` means "zero
  deprecations tolerated".
- `symfony/error-handler`'s `DebugClassLoader` surfaces usage of `@deprecated`
  and `@internal` API that produces no runtime notice.
- Symfony's `config` package validates configuration at build time — a config
  referencing a removed option fails at cache warmup, not at request time. Warm
  the cache in CI to catch it early.
- Framework-level deprecations are separate from PHP deprecations: audit
  `symfony/*` changelogs for the target version too, and record them in the same
  audit files with the Symfony RFC/changelog link as the source.

### C.2 Laravel

- Laravel 8 supports PHP 7.3–8.x; Laravel 9 requires PHP ≥ 8.0.2; Laravel 10
  requires PHP ≥ 8.1; Laravel 11 requires PHP ≥ 8.2; Laravel 12 requires PHP
  ≥ 8.2.
- `php artisan config:cache` and `route:cache` mean configuration errors surface
  at build time — run them in CI.
- `php artisan optimize` plus `composer dump-autoload --optimize` for production.
- The framework's own deprecations are surfaced through the Laravel log channel
  (`deprecations` channel). Route them and treat them as failures during the
  migration window.

### C.3 WordPress and plugins

- The `php-stubs/wordpress-stubs` package gives PHPStan/Psalm the WordPress
  function and class map, which is essential for meaningful static analysis.
- `wp_die()` and `wpdb` are the usual integration points to re-check.
- WordPress plugins are distributed as zips; a plugin must be verified on the
  minimum PHP version it claims to support, not only on the newest.

---

## Portability

This skill is written against the [Agent Skills](https://agentskills.io)
standard, so it loads the same way in Claude Code, Cursor, Codex, Gemini CLI,
GitHub Copilot, OpenCode, and any other harness that discovers skills. It
assumes only three capabilities:

| Need                     | If your harness names it differently                          |
|--------------------------|---------------------------------------------------------------|
| **Run shell commands**   | `Bash`, `shell`, `terminal`, `run_command`, `execute_command` |
| **Fetch a URL**          | `WebFetch`, `webfetch`, `web.run`, `WebSearch`, `browse`      |
| **Read and write files** | `Read`/`Write`, `Edit`, `apply_patch`, `str_replace_editor`   |

Rules for adapting:

- **No web tool?** Use `curl -sL https://www.php.net/manual/en/migration80.incompatible.php`.
  If the machine has no network at all, ask the user to paste the page content
  (or point at a local checkout of the manual) and continue — the audit is
  worthless without the real documentation, so do not substitute a summary of
  it, however plausible, for the page itself.
- **No shell?** The audit and baseline phases are impossible without one. Say so
  and stop; do not improvise a migration you cannot verify.
- **No git?** Phases 1, 5 and 9 assume a branch and one commit per item. If git
  is unavailable, still keep one change per step and record what you changed in
  `docs/migration/action-plan.md`.
- **Interactive chat only (no repo access)?** Run the audit as a
  *review*: read the code the user pastes, answer with the audit table and the
  action plan, and let them apply the changes. Keep the same verdicts, probes
  and evidence discipline — that is the value of this skill.

Everything else in this document is plain shell, PHP, and markdown on purpose:
no harness-specific tool names, no plugins, no MCP servers.

---

## Quick reference

```bash
# Where am I?
php -v && composer why-not php 8.5 && composer check-platform-reqs

# Audit helpers
vendor/bin/phpcs -ps src tests --standard=phpcs.xml.dist --runtime-set testVersion 5.6-8.5
vendor/bin/rector process --dry-run --clear-cache
vendor/bin/phpstan analyse --no-progress

# Prove nothing broke
vendor/bin/phpunit --fail-on-deprecation --fail-on-warning
git diff --stat docs/migration/golden/
```
