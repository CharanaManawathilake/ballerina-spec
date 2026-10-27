# Introduce Build Gating and an Exit Code Contract for `bal scan`

- Authors - Charana Manawathilake
- Reviewed by - TBD
- Created date - 2026-10-26
- Updated date - 2026-10-26
- Issue - [1497](https://github.com/ballerina-platform/ballerina-spec/issues/1497)
- State - Submitted

## Summary

This proposal gives `bal scan` a documented set of exit codes and an opt-in gate, so whatever runs the scan, such as a script, a Git hook or a CI/CD pipeline, can fail on findings and tell findings apart from configuration errors and tool failures.

- Exit code `0` means the scan succeeded, `1` means the gate was tripped, and `2` to `6` mean usage, configuration, project, plugin and internal errors.
- Two new options, `--fail-on-severity <BLOCKER|HIGH|MEDIUM|LOW|INFO|NONE>` and `--fail-on-count <n>`, also settable in `Scan.toml`, define the gate. The scan fails with `1` when the number of issues at or above the severity reaches the count.

By default no gate is set, so findings never fail the scan and it exits with `0`. Errors, however, always exit with a code of `2` or more, so a broken `Scan.toml` no longer passes silently.

## Motivation

Today `bal scan` exits with `0` for almost every outcome, including real failures. A caller cannot tell a clean scan from a broken one without parsing standard output. The table lists what `ScanCmd.execute()` and `executeProject()` do in scan tool 0.12.0.

| Situation | What the scan tool does | Exit code |
| --- | --- | --- |
| Scan finished with no issues | Prints the results and returns | `0` |
| Scan finished with issues of any severity, `BLOCKER` included | Prints the results, writes `scan_results.json` or SARIF, and returns | `0` |
| Invalid project path, or a file inside a package | Prints a message to standard output and returns `Optional.empty()` | `0` |
| Project load diagnostics, such as an invalid `Ballerina.toml` | Prints the errors to standard output and returns | `0` |
| Empty package (`STATIC_ANALYSIS_ERROR_002`) | Prints an error and returns, or skips the package in a workspace | `0` |
| `Scan.toml` cannot be read, or a remote configuration or JAR download fails (`ERROR_003` to `ERROR_007`) | `loadScanTomlConfigurations` returns empty and the scan stops | `0` |
| An external analyzer (compiler plugin) fails to load (`ERROR_008`) | Prints a message and returns | `0` |
| Both include and exclude rules are given (`ERROR_016`) | Prints an error and returns | `0` |
| A platform named in `--platforms` is missing | Prints a message and continues | `0` |
| Compilation errors (`ERROR_019`) | Throws `BLauncherException` | `1`, from the launcher |
| Writing the workspace report fails with an `IOException` | Wraps it in a `RuntimeException` | `1`, from the launcher's `catch (Throwable e)` |

Three gaps follow from this.

- **Issues never fail the scan.** There is no way to say "fail on any `BLOCKER`" or "fail on ten or more issues". The `Severity` enum (`BLOCKER`, `HIGH`, `MEDIUM`, `LOW`, `INFO`) already exists, but nothing reads it when the tool exits.
- **Tool and configuration errors look like success.** Errors are printed to `outputStream` (standard output), not `errorStream`, and the method returns normally. A typo in `Scan.toml` gives a successful run that scanned nothing.

Anything that runs `bal scan` and acts on the result needs this contract to fail for the right reason and to report it clearly. CI/CD integrations, such as the [`scan-ballerina` GitHub Action](1507_scan_github_action.md), are one example.

## Goals

- Define a stable, documented set of exit codes for `bal scan`.
- Keep the scan exiting with `0` on findings unless a gate is set, so existing scripts and pipelines keep passing.
- Let users fail the scan on a number of issues, on issues at or above a severity, or on both, from the command line or from `Scan.toml`.
- Make every tool, configuration and project error exit with a code of `2` or more.
- Write errors and the gate result to standard error.

## Non-Goals

- **New-code gating.** Gating only on issues introduced since a baseline or a commit is [future work](#future-work).
- **Per-severity count limits.** One gate, a severity and a count, applies to the whole scan. Different counts for different severities are [future work](#future-work).
- **Platform quality gates.** When the scan runs from inside a platform, such as SonarQube, that platform's quality gate decides whether the analysis fails.
- **Changes to reports.** Console output, `scan_results.json`, SARIF, the HTML report and platform uploads keep every issue, whatever the gate is set to.

## Design

The design has three parts: a fixed set of exit codes from `0` to `6`, an opt-in gate made of a severity and a count, and errors that always exit with a code of `2` or more.

### Exit codes

| Code | Name | When | Today |
| --- | --- | --- | --- |
| `0` | `SUCCESS` | The scan completed, and no gate is set or the gate was not tripped | `0` |
| `1` | `GATE_FAILED` | The scan completed, and the number of counted issues at or above `--fail-on-severity` is at least `--fail-on-count` | `0` |
| `2` | `USAGE_ERROR` | Invalid command-line input: an unknown `--format` (`ERROR_017`), an invalid `--fail-on-severity` or `--fail-on-count` value, both include and exclude rules (`ERROR_016`), or an unknown rule ID (`ERROR_015`) | `0` |
| `3` | `CONFIG_ERROR` | `Scan.toml` or a remote configuration cannot be read, a field is missing or invalid, a JAR download fails, or a rules JSON file is invalid (`ERROR_003` to `ERROR_007`, `ERROR_009` to `ERROR_012`, `ERROR_020`) | `0` |
| `4` | `PROJECT_ERROR` | The project cannot be analyzed: an invalid path, load diagnostics, an empty package (`ERROR_002`) or compilation errors (`ERROR_019`) | `0`, or `1` for compilation errors |
| `5` | `PLUGIN_ERROR` | An analyzer plugin fails to load (`ERROR_008`), a platform named in `--platforms` is not available, or a platform plugin throws in `init` or `onScan` | `0` |
| `6` | `INTERNAL_ERROR` | An unexpected exception, or a report write failure (`ERROR_014`, or the workspace report `IOException`) | `1` |

The codes are chosen so that:

- A script can test `== 1` for "the code did not pass the gate" and `>= 2` for "the scan tool could not do its job".
- Each error code points to who must act: whoever runs the command (`2`), the owner of `Scan.toml` (`3`), the code author (`4`), the plugin owner (`5`) or the scan tool team (`6`).
- The codes are a small fixed enum, not a bitmask or a count, and stay below `64` to avoid the `sysexits` range and the codes shells use for signals (`128` and above).

These codes are part of the documented scan tool contract. Adding a new code is a minor change. Changing the meaning of an existing code is a breaking change.

### Gate

The gate has two settings.

- `--fail-on-severity <severity>` selects the issues the gate counts: those at or above the severity. The values are `BLOCKER`, `HIGH`, `MEDIUM`, `LOW`, `INFO` and `NONE`.
- `--fail-on-count <n>` is the number of selected issues at which the scan fails. It is a whole number of `1` or more.

The scan exits with `1` when the number of selected issues is greater than or equal to the count. The settings combine as follows.

| `--fail-on-severity` | `--fail-on-count` | Exits with `1` when |
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

The gate follows these rules.

1. **Order.** Severities follow the existing `Severity` enum: `BLOCKER` > `HIGH` > `MEDIUM` > `LOW` > `INFO`. An issue is at or above `X` when its severity is `X` or more severe.
2. **Count.** The count is a threshold, not an allowance: `--fail-on-count=5` fails on the fifth issue, and allows four. The name says when the scan fails, matching `--fail-on-severity`.
3. **Precedence.** Each setting is resolved on its own: the command-line option overrides `Scan.toml`, which overrides the default of not set. So a `Scan.toml` that sets `failOnSeverity = "HIGH"`, run with `--fail-on-count=3`, fails on three or more `BLOCKER` or `HIGH` issues. `--fail-on-severity=NONE` on the command line turns off a gate set in `Scan.toml`.
4. **Validation.** Severity values are case-insensitive. An invalid severity, or a count that is not a whole number of `1` or more, exits with `2` on the command line and `3` in `Scan.toml`.
5. **Exit code only.** The gate changes only the exit code. Console output, `scan_results.json`, SARIF, the HTML report and platform uploads always contain every issue.
6. **Counted issues.** The gate counts the issues left after `--include-rules`, `--exclude-rules` and inline suppressions (`ExcludedIssue`). Excluded issues never count.
7. **Rules without a severity.** A plugin rule can have no severity (`rule.severity() == null`). Such a rule uses the same fallback as `resolveSarifLevel`: `BUG` counts as `HIGH`, `VULNERABILITY` as `MEDIUM` and `CODE_SMELL` as `LOW`. One mapping serves both SARIF and the gate.
8. **Errors win.** When both a tool error and a tripped gate occur, the error code is returned, because the issue list may be incomplete. No gate setting, `NONE` included, hides codes `2` to `6`.
9. **Workspaces.** In a workspace, every package is scanned, and the issues of all packages are counted together against the gate. The tool exits with the highest error code if any package had an error, otherwise `1` if the gate was tripped, otherwise `0`. Empty packages are reported with `4` instead of being skipped silently.
10. **Standard error.** Every error diagnostic moves from `outputStream` to `errorStream`. When the exit code is not `0`, a summary line is written to standard error, for example:

    ```
    error: scan failed: 6 issues at or above MEDIUM, limit 5 (BLOCKER: 1, HIGH: 2, MEDIUM: 3)
    ```

### Execution order

Each stage of a scan can stop it with its own exit code. The gate is evaluated last, after the reports are written, so a failing scan still leaves a full report behind.

1. Parse the command-line options. Fails with `2`.
2. Load `Scan.toml`, remote configurations and rule files, and resolve the gate. Fails with `3`.
3. Load the project and compile it. Fails with `4`.
4. Load analyzer and platform plugins. Fails with `5`.
5. Run the analysis, write the reports, and send results to platforms. Fails with `5` for a platform plugin error, or `6` for a report or unexpected error.
6. Evaluate the gate. Exits with `1` or `0`.

### Compatibility

- Findings still exit with `0` by default, because no gate is set.
- Scripts and pipelines that relied on a `0` exit when `Scan.toml` is broken, a plugin fails to load or a package is empty will start failing. This is the intent of the change.
- Compilation errors move from `1` to `4`. Scripts that test for a non-zero exit are unaffected.

## Alternatives

**One exit code per severity.** The exit code would be the highest severity found, for example `1` for `INFO` up to `5` for `BLOCKER`. This is not adopted.

- The scan report already says which severities were found and how many of each, in `scan_results.json`, SARIF, the HTML report and the summary line. An exit code that repeats part of it adds nothing.
- Most callers ask one question: pass or fail. Shells, build tools and CI/CD systems treat any non-zero exit as a failure, so a team that wants to tolerate `LOW` but block `HIGH` would need a wrapper around every call. With `--fail-on-severity`, the fail point is set in the tool and in `Scan.toml`, where it is versioned and reviewed.
- Severity codes and error codes would share the same small range, which loses the simple `== 1` and `>= 2` rule.
- A non-zero exit on any finding would break every existing script and pipeline, so a gate would still be needed underneath.

**No separate codes for tool failures.** Errors would keep exiting with `0`, as today, or every failure, whether findings or errors, would share one non-zero code. This is not adopted.

- With `0`, a broken `Scan.toml`, a failed JAR download or an empty package gives a green run that scanned nothing. This is the problem the proposal fixes.
- With one shared code, a caller cannot tell "the code did not pass the gate" from "the scan tool could not run", so it cannot report the right reason, retry a failed download, or decide who should act. Separate codes `2` to `6` answer that without parsing output.
- A caller that wants to ignore every outcome can still do so itself, for example with `|| true` in a shell, or the equivalent setting of a CI/CD system. The tool reports what happened, and the caller decides what to do with it.

**A bitmask exit code.** Each bit would stand for a severity, a category or an error kind, so several outcomes could be combined in one code. This is not adopted.

- A scan needs three things: a report of every issue, a severity for each issue, and a fail point. The report and the severities are already there, and the gate provides the fail point, so the exit code only has to say pass, fail or error.
- Bitmasks need bit tests in every script that calls the scan, such as `$(( rc & 4 ))`, which are easy to get wrong, and a few bits cannot say how many issues of each severity were found.
- Error kinds and severities in one bitmask make it unclear which outcome wins, while a fixed enum with "errors win" gives one answer.

## Testing

- **Exit codes.** Unit tests replace the exit handler and assert the code for each situation in the [exit codes](#exit-codes) table: invalid options, each `Scan.toml` and remote configuration error, an invalid path, load diagnostics, an empty package, compilation errors, a plugin that fails to load, a missing platform, a platform plugin that throws, and a report write failure.
- **Gate.** Tests cover each row of the [gate](#gate) table against packages with issues of each severity, including counts just below, at and above the limit. They also cover precedence of each setting between the command line and `Scan.toml`, `--fail-on-severity=NONE` overriding `Scan.toml`, case-insensitive severities, invalid severities and counts (`0`, negative and non-numeric) from both sources, issues excluded by `--include-rules`, `--exclude-rules` and inline suppressions, and plugin rules without a severity.
- **Default.** A scan with issues of every severity and no gate exits with `0`.
- **Errors win.** A scan with both a tripped gate and an error exits with the error code, including with `--fail-on-severity=NONE`.
- **Workspaces.** A workspace with a mix of clean packages, packages with issues, empty packages and packages with errors exits with the most severe outcome, scans every package, and counts issues across packages.
- **Reports.** A scan that exits with `1` still writes `scan_results.json`, SARIF and the HTML report with every issue.
- **Platform-triggered runs and the language server.** A `--platform-triggered` run exits with `0` when the gate would trip and with the error code on errors. `ScanLanguageServerTool` never exits the process.
- **Output streams.** Error diagnostics and the summary line are written to standard error, not standard output.
- **SARIF.** `INFO` rules have `level: "note"`, and `VULNERABILITY` rules have the expected `security-severity`.

## Risks and Assumptions

- Scripts and pipelines that pass today because errors exit with `0` will start failing after the upgrade. The release notes list the affected situations and the new codes.
- Calling `Runtime.getRuntime().exit` from a tool bypasses the launcher's own exit handling. This assumes the launcher does no work after a tool command returns that the scan tool would skip.
- Some tools and wrappers may parse `bal scan` standard output for errors. Moving error diagnostics to standard error changes what they see.
- Many tools use a "maximum allowed" count, which fails above the number. `--fail-on-count` fails at the number. The name and the documentation must make this clear to avoid off-by-one gates.

## Dependencies

- The scan tool's existing `Severity` enum and the `resolveSarifLevel` fallback.
- The `bal` launcher in ballerina-lang, only if the launcher is changed to let `BLauncherException` carry an exit code.
- The SonarQube language and platform plugins in `sonar-ballerina`, which run `bal scan` and act on its exit code.

## Future Work

- **Per-severity count limits.** A table in `Scan.toml` with a separate count for each severity, for example failing on one `BLOCKER` or on ten `MEDIUM` issues.
- **New-code gating.** A baseline file, or a `--new-code-since <commit>` option, so a team can gate on the issues a change introduces without first fixing every existing issue.
- **Per-rule severity overrides** in `Scan.toml`, so a team can raise or lower the severity of a rule for gating and reports.
- **Launcher support for exit codes**, so `BLauncherException` can carry a code and the scan tool no longer exits the process itself.

## References

- [Issue #1497: Build gating and the exit code contract for Ballerina scan tool](https://github.com/ballerina-platform/ballerina-spec/issues/1497)
- [Proposal #1507: Introduce a GitHub Action to run `bal scan` in CI/CD pipelines](1507_scan_github_action.md)
- [Ballerina scan tool documentation](https://ballerina.io/learn/scan-tool/)
- [ballerina-lang launcher `Main.java`](https://github.com/ballerina-platform/ballerina-lang/blob/master/cli/ballerina-cli/src/main/java/io/ballerina/cli/launcher/Main.java)
- [GitHub: SARIF support for code scanning](https://docs.github.com/en/code-security/code-scanning/integrating-with-code-scanning/sarif-support-for-code-scanning)
