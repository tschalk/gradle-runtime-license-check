# gradle-runtime-license-check

A single Gradle init script for scanning runtime dependencies for obvious license risks without adding a Gradle plugin or source file to the target project.

## Start Here

Copy `runtime-license-scan.init.gradle` into the Gradle project you want to check, then run:

```sh
./gradlew -I runtime-license-scan.init.gradle scanRuntimeLicenses
```

Then open the generated reports in that same Gradle project:

```text
build/reports/runtime-license-scan/runtime-license-report.md
build/reports/runtime-license-scan/runtime-license-report.tsv
build/reports/runtime-license-scan/runtime-license-report.json
```

If the project does not have `./gradlew`, use `gradle`:

```sh
gradle -I runtime-license-scan.init.gradle scanRuntimeLicenses
```

To make the build fail when a blocked license is found:

```sh
./gradlew -I runtime-license-scan.init.gradle scanRuntimeLicenses -PfailOnBlocked=true
```

## Copy Into A Project

From a checkout of this repository:

```sh
cp runtime-license-scan.init.gradle /path/to/your-gradle-project/
cd /path/to/your-gradle-project
./gradlew -I runtime-license-scan.init.gradle scanRuntimeLicenses
```

No plugin is applied and no Gradle build file needs to be edited. If you do not want to copy the file, pass Gradle the path to the script:

```sh
./gradlew -I /path/to/gradle-runtime-license-check/runtime-license-scan.init.gradle scanRuntimeLicenses
```

## What It Does

`runtime-license-scan.init.gradle` registers a `scanRuntimeLicenses` task in the Gradle build where you run it. Nothing is installed into that build. The task scans every project and subproject with a resolvable `runtimeClasspath`, walks direct and transitive dependencies, reads Maven POM license metadata where available, and writes:

- `build/reports/runtime-license-scan/runtime-license-report.md`
- `build/reports/runtime-license-scan/runtime-license-report.tsv`
- `build/reports/runtime-license-scan/runtime-license-report.json`

The report includes dependency paths where Gradle exposes them, so you can see which runtime dependency pulled in a transitive module.

The reports are evidence-oriented: they show the POM license metadata, the POM lookup status, optional local JAR license-file evidence, dependency paths, dependencyInsight commands for non-OK findings, the risk category, and the suggested next action. Use Markdown for reading, TSV for spreadsheet filtering, and JSON for automation or follow-up tooling.

## Why It Exists

Sometimes you need a quick runtime dependency license check across a Gradle build without changing that build, adding a plugin, or committing policy tooling into an application repository. This repository is just a home for the reusable init script; copy the script or reference it with `-I`.

## Run From Any Gradle Project

Use a copied script in the current project:

```sh
./gradlew -I runtime-license-scan.init.gradle scanRuntimeLicenses
```

Or use the script from another local checkout:

```sh
./gradlew -I /path/to/gradle-runtime-license-check/runtime-license-scan.init.gradle scanRuntimeLicenses
```

To fail the build when a `BLOCK` dependency is found:

```sh
./gradlew -I runtime-license-scan.init.gradle scanRuntimeLicenses -PfailOnBlocked=true
```

If the target project does not use the Gradle wrapper, use `gradle` instead of `./gradlew`.

To collect local license-file evidence from resolved runtime JARs for `BLOCK`, `REVIEW`, and `UNKNOWN` findings:

```sh
./gradlew -I runtime-license-scan.init.gradle scanRuntimeLicenses -PincludeJarLicenseEvidence=true
```

## Exclude Subprojects

To skip projects before dependency resolution, edit the list near the top of `runtime-license-scan.init.gradle`:

```groovy
static final List<String> EXCLUDED_PROJECTS = [
    'com.heise.*',
    'hallo-welt-*',
]
```

`*` is a wildcard. Patterns are matched against the Gradle project path and project name, so `com.heise.*` can match `:com:heise:app` and `hallo-welt-*` can match a subproject named `hallo-welt-api`.

## Risk Categories

- `BLOCK`: AGPL, or GPL without LGPL and without Classpath Exception.
- `REVIEW`: LGPL, MPL, EPL, CDDL, or GPL with Classpath Exception.
- `UNKNOWN`: missing, unreadable, or empty POM license metadata.
- `OK`: no obvious copyleft risk found in Maven POM metadata.

These categories are triage signals, not legal conclusions. Some findings can be false positives, especially when POM metadata contains broad parent licenses, dual-license choices, or incomplete Classpath Exception details.

Review the Markdown report first. Use the TSV report for sorting and filtering in spreadsheet tools, and use the JSON report for CI, automation, or follow-up tooling.

## Local JAR Evidence

By default, the scan reads Maven POM metadata only. With `-PincludeJarLicenseEvidence=true`, the task also resolves matching runtime JAR artifacts for `BLOCK`, `REVIEW`, and `UNKNOWN` findings and copies common license files such as `LICENSE`, `NOTICE`, `COPYING`, and `COPYRIGHT` into:

```text
build/reports/runtime-license-scan/evidence/
```

The reports link to those local evidence files with relative paths. They do not include remote repository download URLs or local absolute machine paths. This can reduce manual work, but it still does not replace a real legal/compliance review.

## Machine-Readable JSON

`runtime-license-report.json` is the stable input for follow-up tooling. It includes `schemaVersion`, summary counts, `needsManualReview`, scanned configurations, resolution notes, and structured findings with coordinates, license metadata, relative JAR evidence paths, dependency paths, and `dependencyInsightCommands`.

## Triage Policy

- `BLOCK`: treat as a release blocker until reviewed and approved by the project owner or legal/compliance owner.
- `REVIEW`: inspect the dependency, its usage, and your distribution model before accepting it.
- `UNKNOWN`: do not treat as OK; manually verify upstream license information.
- `OK`: no obvious copyleft risk was found in Maven POM metadata, but this is not legal advice.

## Inspect Findings

For each `BLOCK`, `REVIEW`, or `UNKNOWN` row:

1. Check the reported license metadata and dependency paths.
2. Confirm the license from the upstream project or source artifact.
3. Use the reported `dependencyInsight` command if you need Gradle's detailed explanation for why the dependency/version is present.
4. Decide whether the dependency is allowed under your policy.
5. Treat `UNKNOWN` as unresolved until manually checked.

To ask Gradle why a dependency is present, run:

```sh
./gradlew dependencyInsight --dependency group:name --configuration runtimeClasspath
```

For a subproject:

```sh
./gradlew :app:dependencyInsight --dependency group:name --configuration runtimeClasspath
```

Replace `group:name` with the dependency you want to inspect, for example `com.h2database:h2`.

## Limitations

- Maven POM license metadata can be missing, incomplete, or wrong.
- This tool is not legal advice.
- Docker images, npm dependencies, Gradle plugins, OS packages, copied source code, and SaaS terms are not covered.
- `UNKNOWN` dependencies must be checked manually.
- The classifier is intentionally conservative and based on text found in POM license names, URLs, and comments.
- Optional JAR evidence only checks common license files packaged inside resolved runtime JARs; it does not fully scan source code.
