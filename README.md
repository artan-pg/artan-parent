# artan-parent

![CI](https://github.com/artan-pg/artan-parent/actions/workflows/ci.yaml/badge.svg)

The shared parent POM for all projects of the Artan Programming Group.
This project is the central place for managing resources and libraries: dependency versions, build plugin configuration, test and coverage rules, and code style are defined once here and inherited by every child project.

---

## What this parent manages

- **Library versions**: all shared dependencies are pinned in `dependencyManagement` (e.g. `byte-buddy`); child projects use them without declaring versions, and upgrades stay centralized.
- **Plugin configuration**: `pluginManagement` provides consistent behavior for the compiler (`<release>21</release>`), Surefire/Failsafe, JaCoCo, Checkstyle and the Enforcer.
- **Shared properties**: `java.version=21`, `UTF-8` encoding, `threadCount=10`, and other build settings.
- **Build gates**: the Enforcer requires at least **Maven 3.9.0** and **JDK 21** on every build; incompatible environments fail immediately.
- **Project Maven configuration**: `.mvn/maven.config` automatically applies the project `settings.xml` (`.mvn/settings.xml`); the default local repository (`~/.m2/repository`) is used, and `ir.artanpg` artifacts are resolved from the group's Nexus repositories (https://repository.artanpg.ir).

> **Important:** tests are **disabled** by default (`maven.test.skip=true`). They are enabled only through the `execute-tests` profile (see below).

---

## Building

```bash
git clone https://github.com/artan-pg/artan-parent.git
cd artan-parent
./mvnw -B -ntp install
```

Since the project packaging is `pom`, `install` simply installs the parent POM into the local repository (`~/.m2/repository`) so child projects on your machine can resolve it. `package` and `deploy` only move this POM.

---

## Profiles

Five optional profiles are defined in this parent and inherited by every child project:

| Profile         | Purpose                                                               | Example command                             |
|-----------------|-----------------------------------------------------------------------|---------------------------------------------|
| `documentation` | Build the sources jar and the javadoc jar                             | `./mvnw -P documentation package`           |
| `format`        | Check code style with Checkstyle                                      | `./mvnw -P format validate`                 |
| `execute-tests` | Run tests and compute JaCoCo coverage                                 | `./mvnw -P execute-tests verify`            |
| `major`         | Cut the next **major** release with an automatically computed version | `./mvnw -Dmajor initialize release:prepare` |
| `minor`         | Cut the next **minor** release with an automatically computed version | `./mvnw -Dminor initialize release:prepare` |

### 1) The `documentation` profile

Used for publishing documented releases; it produces two extra artifacts in the `package` phase:

- `maven-source-plugin` → sources jar (`jar-no-fork`);
- `maven-javadoc-plugin` → Javadoc jar, compiled with `--enable-preview` and `source=${java.version}`.

Both plugins are declared with `inherited=true`, so they activate in every module of child projects.

### 2) The `format` profile

Unified code-style checking with `maven-checkstyle-plugin` (together with `spring-javaformat-checkstyle`) in the `process-sources` phase.

**Activation requirement**: the child project must define the following three properties; the Enforcer fails the build if any of them is missing:

```xml
<properties>
  <checkstyle.config.location>checkstyle/checkstyle.xml</checkstyle.config.location>
  <checkstyle.properties.location>checkstyle/checkstyle.properties</checkstyle.properties.location>
  <checkstyle.suppressions.location>checkstyle/checkstyle-suppressions.xml</checkstyle.suppressions.location>
</properties>
```

Optional tuning properties: `checkstyle.failOnViolation`, `checkstyle.failsOnError`, `checkstyle.excludes`, `checkstyle.include.test.source` and `checkstyle.include.test.resources`.

### 3) The `execute-tests` profile

Turns on the whole test and coverage cycle (`maven.test.skip=false`):

- **Surefire** (unit tests): only `**/*Tests.java` files; `Abstract*`, `*IT` and `*IntegrationTests` classes are excluded. Tests run in parallel at method level with `threadCount` (default 10).
- **Failsafe** (integration tests): `**/*IntegrationTests.java` and `**/*IT.java` files in the `integration-test` and `verify` phases.
- **Test JVMs**: run with `--enable-preview`, `-Xshare:off` and the `byte-buddy-agent` and `mockito-core` javaagents (versions come from the parent).
- **JaCoCo**: agents for unit and integration tests, plus reports (HTML by default) in the `verify` phase.

**Mandatory minimum coverage** (enforced in the `verify` phase):

| Scope                                   | Counter    | Minimum |
|-----------------------------------------|------------|---------|
| Whole project (excluding `**/utils/**`) | Complexity | 0.95    |
| Whole project (excluding `**/utils/**`) | Branch     | 1.00    |
| Whole project (excluding `**/utils/**`) | Line       | 0.80    |
| `**/utils/**` classes                   | Line       | 0.80    |
| `**/utils/**` classes                   | Method     | 0.80    |

All ratios are tunable through the `jacoco.rule.*` properties. Generated and infrastructure classes (`config`, `dto`, `command`, `entity`, `*Application`, inner classes and `module-info`) are excluded from coverage computation.

### 4) The `major` and `minor` profiles

Release automation — they remove the manual pom editing from the release process.
`build-helper-maven-plugin:parse-version` is bound to the `initialize` phase in this parent's `<plugins>`, so every build (and every child project) exposes the current version as parsed properties; each profile feeds one of those properties into `maven-release-plugin` as the release version:

| Starting version | `-Dmajor` produces | `-Dminor` produces |
|------------------|--------------------|--------------------|
| `1.2.3-SNAPSHOT` | `2.0.0`            | `1.3.0`            |

Both profiles share the same `maven-release-plugin` settings:

- `autoVersionSubmodules=true` — the version rewrite covers every module of a reactor;
- `useReleaseProfile=false` — the release plugin's built-in sources/javadoc profile stays off; producing those jars is the `documentation` profile's job;
- `tagNameFormat=v@{project.version}` — the created tag looks like `v2.0.0`, which matches the `deploy-release` workflow trigger (`v*`) and its tag/version guard.

`release:prepare` rewrites the poms to the release version and commits them, creates the tag, then moves the branch to the next development version (a patch increment — after `2.0.0` development continues as `2.0.1-SNAPSHOT`) and commits again. It does **not** deploy anything; publishing stays with CI (see the Release flow below).

> **Invocation note:** always run `initialize` before `release:prepare` in the same command. `release:prepare` called as a standalone goal runs no lifecycle phase, so the `${parsedVersion.*}` properties are never created and the build fails with `Unable to parse the version string "${parsedVersion...}"`.

```bash
# rehearsal — nothing is committed or tagged
./mvnw -B -Dmajor initialize release:prepare -DpushChanges=false -DdryRun=true

# real run — creates two commits and the tag locally
./mvnw -B -Dmajor initialize release:prepare -DpushChanges=false

# then push the branch and the tag; CI builds and publishes
git push origin main v2.0.0
```

> **Two rules for the real run:**
>
> 1. Run it on a **clean working tree** with no leftover `release.properties` from an aborted rehearsal — `release:prepare` creates commits and aborts on uncommitted changes.
> 2. **Never follow up with `release:perform`.** Publishing is CI's job: the Deploy Release workflow deploys from the pushed tag. A local `release:perform` would upload the same release a second time, and the Nexus `maven-releases` repository rejects redeployment, so the build fails.

---

## Publishing

Publishing goes through the group's Maven repositories at https://repository.artanpg.ir and has two separate flows.

**Prerequisites** (one-time, infrastructure side):

1. `distributionManagement` in the pom must point to `https://repository.artanpg.ir/...`;
2. the `maven-releases` and `maven-snapshots` repositories must sit on the default (file) blob store;
3. two secrets named `NEXUS_MAVEN_USERNAME` and `NEXUS_MAVEN_PASSWORD` must be defined in GitHub.

### SNAPSHOT flow (daily development)

The version on the `main` branch always looks like `X.Y.Z-SNAPSHOT`. Every push to `main` (docs-only changes are skipped) runs the "Deploy Snapshot" workflow, which publishes the artifact to `maven-snapshots`. A guard checks the version first; anything without `-SNAPSHOT` is rejected.

### Release flow (official releases)

The release is prepared locally with the `major`/`minor` profiles (see *Profiles* above) and published by CI:

1. Run the preparation with the profile matching the change scope (add `-DdryRun=true` to rehearse first):

    ```bash
    ./mvnw -B -Dminor initialize release:prepare -DpushChanges=false
    ```

   This rewrites the pom to the release version (e.g. `1.1.0`), commits it, tags `v1.1.0`, and moves `main` back to `1.1.1-SNAPSHOT` — no manual pom editing.

2. Push the commits and the tag — the tag name must equal `v@{project.version}` or the workflow guard rejects it:

    ```bash
    git push origin main v1.1.0
    ```

3. The "Deploy Release" workflow proves the build on all three JDKs (21/25/27), verifies the tag against the pom version, and publishes the artifact from JDK 21 to `maven-releases`.

For a version that does not fit the major/minor scheme (e.g. the initial `1.0.0` release or a hotfix cut from an older branch), pass the versions explicitly instead of activating a profile. The plugin then also moves `main` to the next development version in a second commit — `main` must always carry a `-SNAPSHOT` version, otherwise the Deploy Snapshot guard rejects the next push:

```bash
# example: cut 1.0.0, continue development as 1.1.0-SNAPSHOT
./mvnw -B release:prepare -DpushChanges=false -DreleaseVersion=1.0.0 -DdevelopmentVersion=1.1.0-SNAPSHOT -Dtag=v1.0.0

git push origin main v1.0.0
```

Here `-Dtag=vX.Y.Z` is mandatory, because the `v@{project.version}` tag format is configured inside the profiles; without a profile the default tag name would be `artan-parent-X.Y.Z`, which the `v*` trigger never fires on. The Deploy Release guard then applies unchanged. As always, do not run `release:perform` — CI publishes from the tag.

### Manual publishing (without CI)

```bash
export NEXUS_MAVEN_USERNAME='...'
export NEXUS_MAVEN_PASSWORD='...'
./mvnw -B -ntp deploy
```

Credentials are never stored in `settings.xml`; they are injected through environment variables.

---

## Using it in new group projects

1) Copy the `.mvn` folder (containing `settings.xml` and `maven.config`) from this repository into the new project, so the parent resolves from the group repositories and the local repository stays consistent.

2) Declare this parent in the new project's `pom.xml`:

```xml
<parent>
    <groupId>ir.artanpg</groupId>
    <artifactId>artan-parent</artifactId>
    <version>1.0.0</version> <!-- or X.Y.Z-SNAPSHOT for the development version -->
</parent>
```

From this point on, dependency version management, plugin configuration, test and coverage rules, and the profiles above are in effect in your project without any extra setup. Activate each capability by invoking the matching profile:

```bash
./mvnw -P execute-tests verify              # tests + coverage
./mvnw -P documentation package             # sources + javadoc
./mvnw -P format validate                   # code-style check
./mvnw -Dminor initialize release:prepare   # cut a release (CI publishes from the tag)
```

---

## CI/CD

Three workflows are defined in `.github/workflows/`:

| Workflow                                                       | Trigger                                               | Job                                                          |
|----------------------------------------------------------------|-------------------------------------------------------|--------------------------------------------------------------|
| [ci.yaml](.github/workflows/ci.yaml)                           | push/PR on `main` (docs-only changes skipped), manual | Build and verify on the JDK 21 / 25 / 27 matrix              |
| [deploy-snapshot.yaml](.github/workflows/deploy-snapshot.yaml) | push to `main` (docs-only changes skipped), manual    | Matrix build, then publish the SNAPSHOT to Nexus from JDK 21 |
| [deploy-release.yaml](.github/workflows/deploy-release.yaml)   | push of a `v*` tag                                    | Matrix build, tag/version guard, then publish the Release    |

Temurin JDK builds are used; the production line always goes through JDK 21 (the stable LTS).

---

## License

artan-parent is Open Source software released under the [Apache 2.0 license](https://www.apache.org/licenses/LICENSE-2.0.html).
