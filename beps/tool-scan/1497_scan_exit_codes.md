# Introduce Build Gating and an Exit Code Contract for `bal scan`

- Authors - Charana Manawathilake
- Reviewed by - Tharmigan Krishnananthalingam
- Created date - 2026-10-26
- Updated date - 2026-10-28
- Issue - [1497](https://github.com/ballerina-platform/ballerina-spec/issues/1497)
- State - Submitted

## Summary

This proposal gives `bal scan` a documented set of exit codes and an opt-in gate, so whatever runs the scan, such as a script, a Git hook or a CI/CD pipeline, can fail on findings and on configuration errors and tool failures.

- `bal scan` uses the standard exit codes of Ballerina CLI tools: `0` for success, `1` for a runtime error, `2` for a usage error, `3` for an invalid target and `130` for an interrupt. A tripped gate exits with `1`.
- Every failure other than an interrupt is reported on standard error with a message that says what went wrong.
- Two new options, `--fail-on-severity <BLOCKER|HIGH|MEDIUM|LOW|INFO|NONE>` and `--fail-on-count <n>`, also settable in `Scan.toml`, define the gate. The scan fails when the number of issues at or above the severity reaches the count.

By default no gate is set, so findings never fail the scan and it exits with `0`. Errors, however, always exit with a non-zero code, so a broken `Scan.toml` no longer passes silently.

## Motivation

Today `bal scan` exits with `0` for almost every outcome, including real failures. A caller cannot tell a clean scan from a broken one without parsing standard output. The table lists what `ScanCmd.execute()` and `executeProject()` do in scan tool 0.12.0.

| Situation | What the scan tool does | Exit code |
| --- | --- | --- |
| Scan finished with no issues | Prints the results and returns | `0` |
| Scan finished with issues of any severity, `BLOCKER` included | Prints the results, writes `scan_results.json` or SARIF, and returns | `0` |
| Invalid project path, or a file inside a package | Prints a message to standard output and returns `Optional.empty()` | `0` |
| Project load diagnostics, such as an invalid `Ballerina.toml` | Prints the errors to standard output and returns | `0` |
| Empty package (`ERROR_002`) | Prints an error and returns, or skips the package in a workspace | `0` |
| `Scan.toml` cannot be read, or a remote configuration or JAR download fails (`ERROR_003` to `ERROR_007`) | `loadScanTomlConfigurations` returns empty and the scan stops | `0` |
| An external analyzer (compiler plugin) fails to load (`ERROR_008`) | Prints a message and returns | `0` |
| Both include and exclude rules are given (`ERROR_016`) | Prints an error and returns | `0` |
| A platform named in `--platforms` is missing | Prints a message and continues | `0` |
| Compilation errors (`ERROR_019`) | Throws `BLauncherException` | `1`, from the launcher |
| Writing the workspace report fails with an `IOException` | Wraps it in a `RuntimeException` | `1`, from the launcher's `catch (Throwable e)` |

Three gaps follow from this.

- **Issues never fail the scan.** There is no way to say "fail on any `BLOCKER`" or "fail on ten or more issues". The `Severity` enum (`BLOCKER`, `HIGH`, `MEDIUM`, `LOW`, `INFO`) already exists, but nothing reads it when the tool exits.
- **Tool and configuration errors look like success.** Errors are printed to `outputStream` (standard output), not `errorStream`, and the method returns normally. A typo in `Scan.toml` gives a successful run that scanned nothing.
- **Failures are not reported consistently.** Errors are written in different forms, mostly to standard output, so a caller cannot reliably tell a usage error or an invalid project from a successful run.

Anything that runs `bal scan` and acts on the result needs this contract to fail for the right reason and to report it clearly. CI/CD integrations, such as the [`scan-ballerina` GitHub Action](1507_scan_github_action.md), are one example.

## Goals

- Define a stable, documented exit code for every outcome of `bal scan`, using the standard exit codes of Ballerina CLI tools.
- Keep the scan exiting with `0` on findings unless a gate is set, so existing scripts and pipelines keep passing.
- Let users fail the scan on a number of issues, on issues at or above a severity, or on both, from the command line or from `Scan.toml`.
- Make every tool, configuration and project error exit with a non-zero code.
- Write errors and the gate result to standard error, as text.

## Non-Goals

- **New-code gating.** Gating only on issues introduced since a baseline or a commit is [future work](#future-work).
- **Per-severity count limits.** One gate, a severity and a count, applies to the whole scan. Different counts for different severities are [future work](#future-work).
- **Platform quality gates.** When the scan runs from inside a platform, such as SonarQube, that platform's quality gate decides whether the analysis fails.
- **Changes to reports.** Console output, `scan_results.json`, SARIF, the HTML report and platform uploads keep every issue, whatever the gate is set to.

## Design

The design has three parts: the standard exit codes for each failure, an opt-in gate made of a severity and a count, and errors that always exit with a non-zero code.

### Exit codes

`bal scan` uses only the following codes.

| Code | Name | When | Today |
| --- | --- | --- | --- |
| `0` | Success | The scan completed, and no gate is set or the gate was not tripped | `0` |
| `1` | Runtime error | The gate was tripped, or the scan could not be completed: a configuration, plugin, compilation, suppression or report error, or an unexpected exception | `0`, or `1` for compilation errors and report write failures |
| `2` | Usage error | Invalid command-line input: an invalid option value, such as an unknown `--format` (`ERROR_017`) or an invalid `--fail-on-severity` value, both include and exclude rules (`ERROR_016`), or an unknown rule ID (`ERROR_015`) | `0` |
| `3` | Invalid target | The path does not exist or cannot be loaded as a project, the project has load diagnostics such as an invalid `Ballerina.toml`, or the path is a `.bal` file inside a package | `0` |
| `130` | Interrupted | The scan was interrupted with SIGINT | `130` |

### Failures

Every failure other than an interrupt writes an error report to standard error and exits with the code below. An interrupt exits with `130` and writes no error report.

| Failure | Exit code |
| --- | --- |
| Invalid command-line input, as listed for code `2` | `2` |
| An invalid project, as listed for code `3` | `3` |
| The gate was tripped: the number of counted issues at or above `--fail-on-severity` is at least `--fail-on-count`, for the workspace or, with package gates, for any package | `1` |
| `Scan.toml` or a remote configuration cannot be read, a field is missing or invalid, a rules JSON file is invalid, or a `[suppression]` setting is not a boolean (`ERROR_003`, `ERROR_004`, `ERROR_009` to `ERROR_012`, `ERROR_020`, `ERROR_025`) | `1` |
| A remote configuration or a JAR cannot be downloaded (`ERROR_005` to `ERROR_007`) | `1` |
| The package has compilation errors (`ERROR_019`), including a malformed rule ID or an empty rule list in a `@scan:Suppress` annotation (`SCAN_SUPPRESS_101`, `SCAN_SUPPRESS_107`) | `1` |
| The package has no sources to scan (`ERROR_002`) | `1` |
| An analyzer plugin fails to load (`ERROR_008`), a platform named in `--platforms` is not available, or a platform plugin throws in `init` or `onScan` | `1` |
| A `@scan:Suppress` annotation names a rule that does not exist (`ERROR_024`), gives no reason while `requireReason` or `--require-suppression-reason` is set, or names no rule while `requireRuleId` or `--require-suppression-rule-id` is set. Never reported with `--ignore-suppressions` or `enabled = false` | `1` |
| A report cannot be written (`ERROR_014`, or the workspace report `IOException`) | `1` |
| An unexpected exception | `1` |

A script can test `== 0` for "the scan passed", and any other code for "the scan did not pass". The error message on standard error says which failure occurred.

### Error reports

Error reports follow a text format, described below, and are written to standard error. Scan tool 0.12.0 prints errors as plain text on standard output. A machine-readable form, such as JSON, is [future work](#future-work), pending a standard output format for Ballerina CLI tools.

An error report is `bal scan: error: <message>`. The message names the path, option or value involved, is one line, starts in lowercase (a number, path or name stays as written), and has no trailing period or stack trace.

```
bal scan: error: 6 issues at or above MEDIUM, limit 5 (BLOCKER: 1, HIGH: 2, MEDIUM: 3)
```

A problem in a file, such as `Scan.toml` or a rules JSON file, is a diagnostic. It is written as `<path>:<line>:<col>: error: <message>`, under the error report. A diagnostic for `Scan.toml` names the key, not the option.

```
bal scan: error: Scan.toml has 1 error
Scan.toml:2:18: error: invalid value 'CRITICAL' for 'failOnSeverity'; expected BLOCKER, HIGH, MEDIUM, LOW, INFO or NONE
```

A `@scan:Suppress` annotation that fails a check is a diagnostic at the annotation, under one error report.

```
bal scan: error: 2 invalid suppressions
main.bal:23:5: error: unknown rule 'ballerina/http:9': 'ballerina/http' does not define this rule
main.bal:41:1: error: suppression has no reason; requireReason is set in Scan.toml
```

A usage error is followed by a one-line synopsis and a pointer to the help, and not by the full help.

```
bal scan: error: invalid value '0' for --fail-on-count; expected a whole number of 1 or more
Usage: bal scan [OPTIONS] [<package>]
Try 'bal scan --help' for more information.
```

An unknown rule ID is a usage error that names the ID, with a suggestion to run `bal scan --list-rules`. The scan never suggests or guesses a near-match rule ID.

```
bal scan: error: unknown rule 'ballerina:99'
  Run 'bal scan --list-rules' to see the available rules.
Usage: bal scan [OPTIONS] [<package>]
Try 'bal scan --help' for more information.
```

Warnings, such as a deprecated option, are reported in the same format as error reports, as `bal scan: warning: <message>`. A warning never changes the exit code.

```
bal scan: warning: --example-option is deprecated; use --new-option
```

### Gate

The gate has two settings.

- `--fail-on-severity <severity>` selects the issues the gate counts: those at or above the severity. The values are `BLOCKER`, `HIGH`, `MEDIUM`, `LOW`, `INFO` and `NONE`, and are case-insensitive, so `high` and `High` are the same as `HIGH`.
- `--fail-on-count <n>` is the number of selected issues at which the scan fails. It is a whole number of `1` or more.

The scan fails with exit code `1` when the number of selected issues is greater than or equal to the count. The settings combine as follows.

| `--fail-on-severity` | `--fail-on-count` | Fails when |
| --- | --- | --- |
| Not set | Not set | Never. The gate is off |
| Not set | `n` | There are `n` or more issues of any severity |
| `X` | Not set | There is at least one issue at or above `X` |
| `X` | `n` | There are `n` or more issues at or above `X` |
| `NONE` | Any | Never. The gate is off |

In other words, a count without a severity counts every severity, as if the severity were `INFO`, and a severity without a count uses a count of `1`.

```bash
# Fail on one or more BLOCKER or HIGH issues
bal scan --fail-on-severity=HIGH

# Fail on 10 or more issues of any severity
bal scan --fail-on-count=10

# Fail on 5 or more BLOCKER, HIGH or MEDIUM issues
bal scan --fail-on-severity=MEDIUM --fail-on-count=5

# Never fail on issues, overriding Scan.toml
bal scan --fail-on-severity=NONE
```

The same settings can be given in a new `[gate]` table in `Scan.toml`, next to the existing `[[platform]]`, `[[analyzer]]` and `[rule]` sections.

```toml
[gate]
failOnSeverity = "MEDIUM"        # BLOCKER | HIGH | MEDIUM | LOW | INFO | NONE
failOnCount = 5                  # 1 or more
```

In a workspace, a new `Scan.toml` at the workspace root can hold the same `[gate]` table, and sets the gate for the whole workspace (rule 9). Today the scan tool reads `Scan.toml` only per package, and analyzers, platforms and rules stay per package, so only `[gate]` is read from the workspace `Scan.toml`. Any other section in it is ignored with a warning. Scanning a single member of a workspace, by giving its path, is a package scan, and does not read the workspace `Scan.toml`.

The gate follows these rules.

1. **Order.** Severities follow the existing `Severity` enum: `BLOCKER` > `HIGH` > `MEDIUM` > `LOW` > `INFO`. An issue is at or above `X` when its severity is `X` or more severe.
2. **Count.** The count is a threshold, not an allowance: `--fail-on-count=5` fails on the fifth issue, and allows four. The name says when the scan fails, matching `--fail-on-severity`.
3. **Precedence.** Each setting is resolved on its own: the command-line option overrides `Scan.toml`, which overrides the default of not set. So a `Scan.toml` that sets `failOnSeverity = "HIGH"`, run with `--fail-on-count=3`, fails on three or more `BLOCKER` or `HIGH` issues. `--fail-on-severity=NONE` on the command line turns off a gate set in `Scan.toml`. When the command line and `Scan.toml` set the same setting to different values, the scan writes a warning that names both, such as `bal scan: warning: --fail-on-severity=LOW overrides failOnSeverity = "HIGH" in Scan.toml`. The same value in both places, or each setting coming from a different place, gives no warning.
4. **Validation.** Severity values are case-insensitive. An invalid severity, or a count that is not a whole number of `1` or more, is a usage error (exit code `2`) on the command line and a configuration error (exit code `1`) in `Scan.toml`.
5. **Exit code only.** The gate changes only the exit code and the error report. Console output, `scan_results.json`, SARIF, the HTML report and platform uploads always contain every issue.
6. **Counted issues.** The gate counts the issues left after `--include-rules`, `--exclude-rules` and `@scan:Suppress` annotations ([proposal 1496](1496-in-source-scan-suppression.md)). Excluded and suppressed issues never count, unless `--ignore-suppressions` or `enabled = false` in `[suppression]` disregards the annotations, in which case suppressed issues count.
7. **Rules without a severity.** A plugin rule can have no severity (`rule.severity() == null`). Such a rule uses the same fallback as `resolveSarifLevel`: `BUG` counts as `HIGH`, `VULNERABILITY` as `MEDIUM` and `CODE_SMELL` as `LOW`. One mapping serves both SARIF and the gate.
8. **Errors win.** When both a tool error and a tripped gate occur, the error is reported instead of the gate failure. No gate setting, `NONE` included, hides an error.
9. **Workspaces.** In a workspace, every package is scanned. If any package had an error, the scan writes one error report and exits with `3` if any package is an invalid target, and `1` otherwise. The message gives the number of failing packages, and the error of each failing package is a diagnostic that names the package. Empty packages are reported as errors instead of being skipped silently. Otherwise the gate is evaluated in one of two ways, and the scan exits with `0` if it is not tripped.
   - **Workspace gate.** When the command line or the workspace `Scan.toml` sets `failOnSeverity`, `failOnCount` or both, one gate applies to the whole workspace, and the issues of all packages are counted together. Each setting is resolved as in rule 3, with the workspace `Scan.toml` in place of the package's, and a setting given in neither place takes its default. A `[gate]` table in a package's `Scan.toml` is ignored, with a warning that names the packages, such as `bal scan: warning: [gate] in pkg-a/Scan.toml and pkg-b/Scan.toml is ignored; the workspace gate applies`. `NONE` at the workspace level turns off the gate for every package.
   - **Package gates.** Otherwise each package is checked against the gate in its own `Scan.toml`, counting only its own issues, and a package without one is not gated. The scan fails with exit code `1` if any package trips its gate. The message gives the number of failing packages, and each failing package is a diagnostic that names it.

   ```
   bal scan: error: 2 packages failed the gate
   pkg-a: error: 2 issues at or above HIGH, limit 1 (BLOCKER: 1, HIGH: 1)
   pkg-b: error: 12 issues at or above LOW, limit 10 (HIGH: 2, MEDIUM: 4, LOW: 6)
   ```

10. **Standard error.** Every error diagnostic moves from `outputStream` to `errorStream`, and the error report is the last thing written to standard error.

A run that trips the gate looks like a run without a gate, followed by the gate error on standard error and exit code `1`. The scan results and report locations on standard output are unchanged.

```bash
$ bal scan --fail-on-severity=MEDIUM --fail-on-count=5
# Standard output: the scan results and report locations, as without a gate
...
# Standard error: the gate error, last
bal scan: error: 6 issues at or above MEDIUM, limit 5 (BLOCKER: 1, HIGH: 2, MEDIUM: 3)
$ echo $?
1
```

### Execution order

Each stage of a scan can stop it with its own exit code. The gate is evaluated last, after the reports are written, so a failing scan still leaves a full report behind.

1. Parse the command-line options. Fails with `2`.
2. Load the project. Fails with `3`.
3. Load each package's `Scan.toml`, the workspace `Scan.toml` in a workspace, remote configurations and rule files, and resolve the gate. Fails with `1` on a configuration or download error.
4. Compile the project. Fails with `1` on compilation errors or an empty package.
5. Load analyzer and platform plugins. Fails with `1`.
6. Run the analysis. Fails with `1` on a plugin error or an unexpected exception.
7. Apply `@scan:Suppress` annotations, check that the rules they name exist, and check the `[suppression]` requirements. A failure here does not stop the scan, so the reports in step 8 still hold every finding and suppression.
8. Write the reports, and send results to platforms. Fails with `1` on a plugin, report or unexpected error. Then, if step 7 found an invalid suppression, fails with `1`.
9. Evaluate the gate. Fails with `1`, or exits with `0`.

### Compatibility

- Findings still exit with `0` by default, because no gate is set.
- Scripts and pipelines that relied on a `0` exit when the project path is invalid, `Scan.toml` is broken, a plugin fails to load or a package is empty will start failing. This is the intent of the change.
- Compilation errors and report write failures keep exiting with `1`.

## Alternatives

**One exit code per severity.** The exit code would be the highest severity found, for example `1` for `INFO` up to `5` for `BLOCKER`. This is not possible under the Ballerina CLI Tool Design Guide.

- The guide's exit code table is closed, and a tool must not define other codes (section 8). Codes `2` and `3` already mean a usage error and an invalid target in every `bal` tool, `4` means overwrite refused, and `5` is not allowed at all.
- The design does not need it either. The scan report already says which severities were found and how many of each, and `--fail-on-severity` sets the fail point in the tool and in `Scan.toml`, where it is versioned and reviewed.

**A separate exit code for each type of failure.** A tripped gate, a configuration error, a plugin error and an internal error would each get their own code. This is not possible under the Ballerina CLI Tool Design Guide.

- The guide's exit code table is closed, and a tool must not define other codes (section 8). Every failure that is not a usage error, an invalid target, a refused overwrite or an interrupt exits with `1`, so a tripped gate, a configuration error, a plugin error and an internal error all share it.
- The design does not need it either. Shells, build tools and CI/CD systems act on zero and non-zero, and the error report on standard error says which failure occurred.

**Keep errors exiting with `0`.** This is not adopted. A broken `Scan.toml`, a failed JAR download or an empty package would keep giving a green run that scanned nothing, which is the problem the proposal fixes. A caller that wants to ignore every outcome can still do so itself, for example with `|| true` in a shell, or the equivalent setting of a CI/CD system.

**A bitmask exit code.** Each bit would stand for a severity, a category or a type of failure, so several outcomes could be combined in one code. This is not possible under the Ballerina CLI Tool Design Guide.

- The guide's exit code table is closed, and a tool must not define other codes (section 8). Combined codes such as `5` or `6` are not in the table, and single bits would reuse `1`, `2` and `4`, which already mean a runtime error, a usage error and a refused overwrite.
- The design does not need it either. The report and the severities are already there, and the gate provides the fail point, so the exit code only has to say pass or fail.

## Testing

- **Exit codes.** Unit tests replace the exit handler and assert the exit code and the error report for each situation in the [failures](#failures) table: invalid options, an invalid path, a `.bal` file inside a package, load diagnostics, each `Scan.toml` and remote configuration error, a failed download, compilation errors, an empty package, a plugin that fails to load, a missing platform, a platform plugin that throws, and a report write failure. An interrupt exits with `130` and writes no error report.
- **Gate.** Tests cover each row of the [gate](#gate) table against packages with issues of each severity, including counts just below, at and above the limit. They also cover precedence of each setting between the command line and `Scan.toml`, the override warning when both set the same setting to different values and its absence otherwise, `--fail-on-severity=NONE` overriding `Scan.toml`, case-insensitive severities, invalid severities and counts (`0`, negative and non-numeric) from both sources, issues excluded by `--include-rules`, `--exclude-rules` and inline suppressions, and plugin rules without a severity.
- **Suppressions.** An unknown rule in an annotation, a missing reason with `requireReason` or `--require-suppression-reason`, and a missing rule list with `requireRuleId` or `--require-suppression-rule-id` each exit with `1` after the reports are written, and win over a tripped gate. None of them fails the scan with `--ignore-suppressions` or `enabled = false`, and then suppressed issues count toward the gate. A malformed rule ID or an empty rule list exits with `1` as a compilation error, and a non-boolean `[suppression]` setting exits with `1` as a configuration error. Suppression warnings, such as an unmatched or unverifiable rule, leave the exit code unchanged.
- **Default.** A scan with issues of every severity and no gate exits with `0`.
- **Errors win.** A scan with both a tripped gate and an error reports the error, not the gate failure, including with `--fail-on-severity=NONE`.
- **Workspaces.** A workspace with a mix of clean packages, packages with issues, empty packages and packages with errors scans every package, counts issues across packages, and reports every failing package in the error report, exiting with `3` if any package is an invalid target and `1` otherwise. Gate tests cover a workspace gate set on the command line, in the workspace `Scan.toml` and in both, with package `[gate]` tables ignored and the warning written; `NONE` at the workspace level; package gates when no workspace setting is given, including packages without a gate and the per-package error report; other sections in the workspace `Scan.toml` ignored with a warning; and a member package scanned on its own, which does not read the workspace `Scan.toml`.
- **Reports.** A scan that fails on the gate still writes `scan_results.json`, SARIF and the HTML report with every issue.
- **Error reports.** Error reports follow the `bal scan: error: <message>` form, `Scan.toml` problems are reported as `<path>:<line>:<col>` diagnostics, and usage errors print the synopsis and help pointer but not the full help. Warnings follow the same format, as `bal scan: warning: <message>`.
- **Output streams.** Error diagnostics and the error report are written to standard error, not standard output.

## Risks and Assumptions

- Scripts and pipelines that pass today because errors exit with `0` will start failing after the upgrade. The release notes list the affected situations and their exit codes.
- A tripped gate and a tool error share exit code `1`. Until a machine-readable error output is added, a caller that treats them differently can only tell them apart from the error message, which is not a stable interface. The documentation must make this clear.
- Until the launcher can carry an exit code, the scan tool calls `Runtime.getRuntime().exit`, which bypasses the launcher's own exit handling. This assumes the launcher does no work after a tool command returns that the scan tool would skip.
- Some tools and wrappers may parse `bal scan` standard output for errors. Moving error diagnostics to standard error changes what they see.
- Many tools use a "maximum allowed" count, which fails above the number. `--fail-on-count` fails at the number. The name and the documentation must make this clear to avoid off-by-one gates.

## Dependencies

- The scan tool's existing `Severity` enum and the `resolveSarifLevel` fallback.
- [Proposal #1496: In-source suppression of scan findings](1496-in-source-scan-suppression.md), for the `@scan:Suppress` annotation, the `[suppression]` settings and their diagnostics. The suppression failures and the suppression step of the execution order apply once it is in place.
- The `bal` launcher in ballerina-lang, to let `BLauncherException` carry an exit code, as the guide expects (OUT-4). Until then the scan tool exits on its own.

## Future Work

- **Machine-readable error output.** An option, such as `--output-format json`, that writes error reports and warnings as JSON, once the Ballerina CLI Tool Design Guide settles the format. Each error would carry a stable identifier for the failure, so a caller can tell a tripped gate from a tool error without parsing the message.
- **Per-severity count limits.** A table in `Scan.toml` with a separate count for each severity, for example failing on one `BLOCKER` or on ten `MEDIUM` issues.
- **New-code gating.** A baseline file, or a `--new-code-since <commit>` option, so a team can gate on the issues a change introduces without first fixing every existing issue.
- **Per-rule severity overrides** in `Scan.toml`, so a team can raise or lower the severity of a rule for gating and reports.

## References

- [Issue #1497: Build gating and the exit code contract for Ballerina scan tool](https://github.com/ballerina-platform/ballerina-spec/issues/1497)
- [Proposal #1496: In-source suppression of scan findings](1496-in-source-scan-suppression.md)
- [Proposal #1507: Introduce a GitHub Action to run `bal scan` in CI/CD pipelines](1507_scan_github_action.md)
- [Ballerina scan tool documentation](https://ballerina.io/learn/scan-tool/)
- [ballerina-lang launcher `Main.java`](https://github.com/ballerina-platform/ballerina-lang/blob/master/cli/ballerina-cli/src/main/java/io/ballerina/cli/launcher/Main.java)
