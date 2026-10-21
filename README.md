# php-migration

Migrate a PHP application from **5.x to 8.5** — and prove nothing broke.

- **Audit first.** For every version step, the agent goes through the official PHP migration notes change by change and checks your code against each one.
- **Baseline before any edit.** It records how your app behaves today — test results, coverage, configuration, and snapshots of the tricky parts — so "nothing broke" is something you can show, not something you hope.

Then it fixes things one at a time. For each change: write the test first, make the change, run the whole suite again, and compare the snapshots against the ones taken at the start.

> Any difference in those snapshots has to match a change the audit already justified. A difference with no explanation means something broke, so the agent stops and works it out — it never updates a snapshot just to make the difference disappear.

```
Understand the app → check every documented change → record how it behaves today
→ fix one change at a time → tighten the types → test on every PHP version
→ deploy safely → write it up
```

Runs on any AI coding agent: Claude Code, Cursor, Codex, Gemini CLI, GitHub Copilot, OpenCode, or anything else that reads [Agent Skills](https://agentskills.io).

---

## Install

One command:

```bash
gh skill install enegalan/php-migrator-skill --agent <your-agent> --scope user
```

`--agent` accepts `claude-code`, `cursor`, `codex`, `gemini-cli`, `github-copilot`, `opencode`, `cline`, `goose`, `windsurf`, `universal` and ~30 more — run `gh skill install --help` for the list. Drop `--scope user` to install it into the current repository instead of your machine.

Prefer to copy it yourself? It is a single markdown file. Drop it in your agent's skills folder — `~/.agents/skills/php-migration/SKILL.md` is read by most of them, and each agent's own path is listed in [Technical details](#technical-details).

## Use it

Just ask your agent:

> Migrate this application to PHP 8.5

The agent picks the skill up on its own. You can also force it with `@php-migration`.

## What happens

| Step                      | What the agent does                                                                                                    |
|---------------------------|--------------------------------------|
| **Look around**           | Works out your PHP version, framework, test setup and CI, and scans for code that will not survive the jump. |
| **Set up the runtimes**   | Installs the target PHP next to the current one in `composer.json`. |
| **Sort out dependencies** | Finds every package that blocks the target PHP, and upgrades or replaces it. |
| **Audit**                 | Goes through the official migration notes for each version, change by change, and checks your code against each one. Each one gets an answer — does it affect you or not — and the proof behind it. |
| **Baseline**              | Records how the app behaves *today*: test results, coverage, runtime config, and snapshots of the risky parts. |
| **Fix, one at a time**    | For each change: write the test, make the change, run the whole suite, compare the snapshots. |
| **Harden types**          | Turns on strict types, adds type declarations, and replaces loose comparisons. |
| **Test & CI**             | Runs your test suite on every PHP version you support, so you find out when a change breaks one of them. Deprecations count as failures. |
| **Deploy & roll back**    | Canary rollout, health checks, and a rollback that has actually been rehearsed. |
| **Clean up & report**     | Removes the scaffolding and writes the final report. |

Nothing gets edited until the audit and the baseline are done. If either turns up something you did not expect, you find out before the code changes — not after.

## What you get

A folder in your repository:

```
docs/migration/
├── 00-baseline.md        how the app looked before
├── 01-environment.md     the runtimes used
├── 02-dependencies.md    every dependency decision and why
├── audit/                one file per version step: every documented
│                         change, whether it applies to you, and the proof
├── action-plan.md        the work items, one per applicable change
├── golden/               snapshots of current behaviour — the reference
└── MIGRATION-REPORT.md   the final summary
```

---

## Technical details

<details>
<summary><strong>Acceptance gates — what each change has to satisfy</strong></summary>

1. Pre-existing tests still pass **unchanged** (except tests explicitly updated for that item, each linked to an audit entry).
2. A new or updated test pins the new behaviour.
3. Coverage does not drop below the baseline.
4. Any difference in the snapshots is either expected (it matches an audit entry) or it is a problem to fix.

</details>

<details>
<summary><strong>Tools it leans on</strong></summary>

| Tool                                                                                         | What for                                                          |
|----------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| [`rector/rector`](https://github.com/rectorphp/rector)                                       | Automated version upgrades                                        |
| [`phpcompatibility/php-compatibility`](https://github.com/PHPCompatibility/PHPCompatibility) | Finds code using features that do not exist in the target version |
| [`phpstan/phpstan`](https://github.com/phpstan/phpstan-src)                                  | Static analysis and type checking                                 |
| [`phpunit/phpunit`](https://github.com/sebastianbergmann/phpunit)                            | The test suite                                                    |
| [`roave/backward-compatibility-check`](https://github.com/Roave/BackwardCompatibilityCheck)  | Guards your own public API if you ship a library                  |

The skill lists the versions it was built against. Check the current ones with `composer show <package> --all` before pinning them in your project.

</details>

## License

MIT — see [LICENSE](LICENSE).
