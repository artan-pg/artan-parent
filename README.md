# artan-parent

![CI](https://github.com/artan-pg/artan-parent/actions/workflows/ci.yaml/badge.svg)
[![Maven Central](https://img.shields.io/badge/maven%20central-pending-lightgrey)](https://central.sonatype.com/)
[![Java](https://img.shields.io/badge/Java-21%2B-orange)](https://openjdk.org/)
[![Maven](https://img.shields.io/badge/Maven-3.9%2B-blue)](https://maven.apache.org/)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![SemVer](https://img.shields.io/badge/SemVer-2.0.0-blue)](https://semver.org/)
[![Maintenance](https://img.shields.io/badge/Maintenance-Actively%20Maintained-brightgreen)](https://github.com/artan-pg/artan-parent)

The shared parent POM for all projects of the Artan Programming Group.
This project is the central place for managing resources and libraries: dependency versions, build plugin configuration,
test and coverage rules, and code style are defined once here and inherited by every child project.

---

## What this parent manages

- **Library versions**: all shared dependencies are pinned in `dependencyManagement`; child projects use them without
  declaring versions, and upgrades stay centralized.
- **Plugin configuration**: `pluginManagement` provides consistent behavior for the compiler (`<release>21</release>`),
  Surefire/Failsafe, JaCoCo, Checkstyle and the Enforcer.
- **Shared properties**: `java.version=21`, `UTF-8` encoding, `threadCount=10`, and other build settings.
- **Build gates**: the Enforcer runs on every build and requires:
  - **Maven ≥ 3.9.0** and **JDK ≥ 21**
  - no duplicate dependency versions in the POM
  - explicit plugin versions — no `LATEST` / `RELEASE` / plugin snapshots
  - on **release** builds only: no SNAPSHOT dependencies
- **Project Maven configuration**: `.mvn/maven.config` automatically applies the project `settings.xml`
  (`.mvn/settings.xml`); the default local repository (`~/.m2/repository`) is used, and `ir.artanpg` artifacts are
  resolved from the group's Nexus repositories (https://repository.artanpg.ir).

> **Important:** tests are **disabled** by default (`maven.test.skip=true`). They are enabled only through the
`execute-tests` profile (see below).

---

## Building

```bash
git clone https://github.com/artan-pg/artan-parent.git
cd artan-parent
./mvnw -B -ntp install
```

Since the project packaging is `pom`, `install` simply installs the parent POM into the local repository
(`~/.m2/repository`) so child projects on your machine can resolve it. `package` and `deploy` only move this POM.

---

## Profiles

Nine optional profiles are defined in this parent and inherited by every child project:

| Profile         | Purpose                                                                                    | Example command                             |
|-----------------|--------------------------------------------------------------------------------------------|---------------------------------------------|
| `documentation` | Build the sources jar and the javadoc jar                                                  | `./mvnw -P documentation package`           |
| `format`        | Check code style with Checkstyle                                                           | `./mvnw -P format validate`                 |
| `execute-tests` | Run tests and compute JaCoCo coverage                                                      | `./mvnw -P execute-tests verify`            |
| `major`         | Cut the next **major** release with an automatically computed version                      | `./mvnw -Dmajor initialize release:prepare` |
| `minor`         | Cut the next **minor** release with an automatically computed version                      | `./mvnw -Dminor initialize release:prepare` |
| `patch`         | Cut the next **patch** release with an automatically computed version                      | `./mvnw -Dpatch initialize release:prepare` |
| `sign`          | Sign artifacts with GPG and deploy the `.asc` signatures alongside                         | `./mvnw -P sign deploy`                     |
| `snapshot-repo` | Guard and prepare a **SNAPSHOT** deploy to the group Nexus (enforcer + publishing plugins) | `./mvnw -P snapshot-repo deploy`            |
| `release-repo`  | Guard and prepare a **release** deploy to Maven Central (enforcer + publishing plugins)    | `./mvnw -P release-repo deploy`             |

### 1) The `documentation` profile

Used for publishing documented releases; it produces two extra artifacts in the `package` phase:

- `maven-source-plugin` → sources jar (`jar-no-fork`);
- `maven-javadoc-plugin` → Javadoc jar, compiled with `--enable-preview` and `source=${java.version}`.

### 2) The `format` profile

Unified code-style checking with `maven-checkstyle-plugin` (together with `spring-javaformat-checkstyle`) in the
`process-sources` phase.

**Activation requirement**: the child project must define the following three properties; the Enforcer fails the build
if any of them is missing:

```xml
<properties>
  <checkstyle.config.location>checkstyle/checkstyle.xml</checkstyle.config.location>
  <checkstyle.properties.location>checkstyle/checkstyle.properties</checkstyle.properties.location>
  <checkstyle.suppressions.location>checkstyle/checkstyle-suppressions.xml</checkstyle.suppressions.location>
</properties>
```

Optional tuning properties: 

- `checkstyle.failOnViolation`
- `checkstyle.failsOnError`
- `checkstyle.excludes`
- `checkstyle.include.test.source`
- `checkstyle.include.test.resources`

### 3) The `execute-tests` profile

Turns on the whole test and coverage cycle (`maven.test.skip=false`):

- **Surefire** (unit tests): only `**/*Tests.java` files; `Abstract*`, `*IT` and `*IntegrationTests` classes are
  excluded. Tests run in parallel at method level with `threadCount` (default 10).
- **Failsafe** (integration tests): `**/*IntegrationTests.java` and `**/*IT.java` files in the `integration-test` and
  `verify` phases.
- **Test JVMs**: run with `--enable-preview`, `-Xshare:off` and the `byte-buddy-agent` and `mockito-core` javaagents
  (versions come from the parent).
- **JaCoCo**: agents for unit and integration tests, plus reports (HTML by default) in the `verify` phase.

**Mandatory minimum coverage** (enforced in the `verify` phase):

| Scope                                   | Counter    | Minimum |
|-----------------------------------------|------------|---------|
| Whole project (excluding `**/utils/**`) | Complexity | 0.95    |
| Whole project (excluding `**/utils/**`) | Branch     | 1.00    |
| Whole project (excluding `**/utils/**`) | Line       | 0.80    |
| `**/utils/**` classes                   | Line       | 0.80    |
| `**/utils/**` classes                   | Method     | 0.80    |

All ratios are tunable through the `jacoco.rule.*` properties. Generated and infrastructure classes
(`config`, `dto`, `command`, `entity`, `*Application`, inner classes and `module-info`) are excluded from coverage
computation.

The instrumentation scope itself is controlled by `jacoco.includes` (default `**`, every class) and `jacoco.excludes`
(empty by default); each accepts a single Ant-style pattern. Never leave `jacoco.includes` empty: an empty value is
interpolated as a literal `null` pattern that matches no class, which silently disables instrumentation and makes the
coverage check pass vacuously over zero analyzed classes.

### 4) The `major`, `minor` and `patch` profiles

Release automation — they remove the manual pom editing from the release process.
`build-helper-maven-plugin:parse-version` is bound to the `initialize` phase in this parent's `<plugins>`, so every
build (and every child project) exposes the current version as parsed properties; each profile feeds one of those
properties into `maven-release-plugin` as the release version:

| Starting version | `-Dmajor` produces | `-Dminor` produces | `-Dpatch` produces |
|------------------|--------------------|--------------------|--------------------|
| `1.2.3-SNAPSHOT` | `2.0.0`            | `1.3.0`            | `1.2.3`            |

All three profiles share the same `maven-release-plugin` settings — defined once in the parent's `pluginManagement`, so
they also apply to every release made **without** a profile:

- `autoVersionSubmodules=true` — the version rewrite covers every module of a reactor;
- `useReleaseProfile=false` — the release plugin's built-in sources/javadoc profile stays off; producing those jars is
  the `documentation` profile's job;
- `tagNameFormat=v@{project.version}` — the created tag always looks like `v2.0.0`, which matches the `deploy-release`
  workflow trigger (`v*`) and its tag/version guard. Because the setting is defined in `pluginManagement` instead of
  inside the profiles, explicit-version releases and hotfix branches produce `v`-tags too — `-Dtag` is never needed on the
  current parent.

`release:prepare` rewrites the poms to the release version and commits them, creates the tag, then moves the branch to
the next development version and commits again. The next development version is computed by the active profile
(for example, after a `patch` release of `1.2.3` development continues as `1.2.4-SNAPSHOT`). It does **not** deploy
anything; publishing stays with CI (see the Release flow below).

> **Invocation note:** always run `initialize` before `release:prepare` in the same command. `release:prepare` called as
> a standalone goal runs no lifecycle phase, so the `${parsedVersion.*}` properties are never created and the build
> fails with `Unable to parse the version string "${parsedVersion...}"`.

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
> 1. Run it on a **clean working tree** with no leftover `release.properties` from an aborted rehearsal —
     `release:prepare` creates commits and aborts on uncommitted changes.
> 2. **Never follow up with `release:perform`.** Publishing is CI's job: the Deploy Release workflow deploys from the
     pushed tag. A local `release:perform` would upload the same release a second time, and the Nexus `maven-releases`
     repository rejects redeployment, so the build fails.

### 5) The `sign`, `snapshot-repo` and `release-repo` profiles

Optional deploy-time guards for publishing to a repository server. They do **not** replace the CI snapshot/release
flows; they add Enforcer guards and attach the signing / publishing plugins through three separate, combinable
profiles when you deploy manually (or from a custom pipeline).

| Profile         | Enforcer rule                                                | When to use                                             |
|-----------------|--------------------------------------------------------------|---------------------------------------------------------|
| `sign`          | — (no Enforcer rule)                                         | Signing a Nexus upload (SNAPSHOT or release) with GPG   |
| `snapshot-repo` | `requireSnapshotVersion` — version **must** be `*-SNAPSHOT`  | Deploying a development build to a snapshots repository |
| `release-repo`  | `requireReleaseVersion` — version **must not** be a SNAPSHOT | Deploying an official release to a releases repository  |

The `snapshot-repo` and `release-repo` profiles both activate:

- `maven-enforcer-plugin` — the version guard shown in the table above
- `central-publishing-maven-plugin` — publishing support (Sonatype Central)

GPG signing is **not** part of them: it is owned by the separate `sign` profile (the passphrase is read from the
`MAVEN_GPG_PASSPHRASE` environment variable; `--pinentry-mode loopback` is pre-configured). Keeping signing
orthogonal lets it pair with any repository profile — or run alone for a signed upload to the plain
`distributionManagement` destination.

The `sign` profile activates **only** `maven-gpg-plugin`: the artifact (and the POM) are signed in the `verify` phase
and the `.asc` signatures are attached to the project, so the following `install`/`deploy` puts them next to the
artifacts — with the plain `distributionManagement` destination. This is what the CI Nexus uploads (both flows) use:

```bash
# Signed deploy to the configured distributionManagement — no profile-specific routing
./mvnw -B -P sign deploy
```

Signing can be switched off with `-Dgpg.skip=true`.

Examples:

```bash
# SNAPSHOT deploy to Nexus — version guard, unsigned
./mvnw -B -P snapshot-repo deploy

# Release deploy to Maven Central — version guard + publishing, unsigned
./mvnw -B -P release-repo deploy

# The same deploys, signed — `sign` composes with any repo profile
./mvnw -B -P sign,snapshot-repo deploy
./mvnw -B -P sign,release-repo deploy
```

Where each profile publishes is **not** symmetric:

- with a `-SNAPSHOT` version the plugin keeps Maven's standard behavior and uploads to the `<snapshotRepository>` of
  `distributionManagement` — the group Nexus (`maven-snapshots`);
- with a release version the plugin (declared with `<extensions>true</extensions>`) **replaces** the
  `maven-deploy-plugin` bindings at runtime and uploads the bundle to Maven Central — `distributionManagement` is not
  used in that mode.

> **Note:** the group CI already publishes snapshots from `main` / `N.N.x` to the Nexus and releases from `v*` tags
> to both the Nexus and Maven Central (see *Publishing* below). Use these profiles for manual or alternative
> repository publishing where the SNAPSHOT vs release constraint must be enforced explicitly. The plugin requires
> Maven 3.9.x — Maven 4 is not supported yet; this repository pins 3.9.9 through the Maven Wrapper.

---

## Versioning (Semantic Versioning)

This project follows [Semantic Versioning 2.0.0](https://semver.org/spec/v2.0.0.html): every release is
`MAJOR.MINOR.PATCH`, and which segment to bump is decided by the nature of the change. Because this artifact is a parent
POM, its "public API" is everything a child project inherits: the Enforcer rules (minimum Maven/JDK, dependency and
plugin constraints), the plugin configuration in `pluginManagement`, the managed dependency versions, and the profiles
and properties documented here.

| Segment | Increment when…                                      | Examples for this parent                                                                                                                                                |
|---------|------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| MAJOR   | the change can break a child build or child code     | raising the Enforcer's minimum Maven/JDK, changing inherited plugin behavior, removing or renaming profiles/properties, managed-dependency upgrades with a breaking API |
| MINOR   | functionality is added in a backwards-compatible way | new managed dependencies, new optional profiles or properties, non-breaking plugin/dependency upgrades                                                                  |
| PATCH   | backwards-compatible bug fixes                       | fixing a broken configuration, documentation-only changes                                                                                                               |

Development versions on `main` carry Maven's `-SNAPSHOT` marker (`X.Y.Z-SNAPSHOT`). `SNAPSHOT` is Maven's "not yet
released" placeholder rather than a SemVer pre-release: the published, immutable versions are always plain `X.Y.Z`, and
it is those versions the SemVer rules govern. (If an official pre-release is ever needed, `X.Y.Z-rc.N` is valid SemVer
and orders before the release in Maven; the CI format guard below would be relaxed accordingly.) Release tags are named
`vX.Y.Z` — the `v` prefix is a Git naming convention, not part of the SemVer string. Which segment the
*next development* version on `main` uses after a release is a workflow convention SemVer deliberately leaves open; this
project's convention is the one implemented by the `major`/`minor`/`patch` profiles (see *Profiles* above).

All three CI workflows enforce the format before anything else runs: `ci.yaml` and `deploy-snapshot.yaml` require
`MAJOR.MINOR.PATCH-SNAPSHOT`, and `deploy-release.yaml` requires `MAJOR.MINOR.PATCH` (on top of its existing
tag/version guard).

---

## Publishing

Artifacts are published to two destinations: the group's Maven repositories at https://repository.artanpg.ir (Nexus)
and **Maven Central** at https://central.sonatype.com. The two version classes have separate flows and destinations:

| Version class     | Destination(s)                                       | Published by                                        |
|-------------------|------------------------------------------------------|-----------------------------------------------------|
| `X.Y.Z-SNAPSHOT`  | group Nexus — `maven-snapshots` **only**             | *Deploy Snapshot* workflow (push to `main`/`N.N.x`) |
| `X.Y.Z` (release) | group Nexus — `maven-releases` **and** Maven Central | *Deploy Release* workflow (push of a `v*` tag)      |

### SNAPSHOT flow (daily development)

The version on the `main` branch always looks like `X.Y.Z-SNAPSHOT`. Every push to `main` (docs-only changes are
skipped) runs the "Deploy Snapshot" workflow, which publishes the artifact to `maven-snapshots`. A guard checks the
version first; anything without `-SNAPSHOT` is rejected.

Snapshots are group-internal and **never** leave the Nexus: the Central Portal snapshot repository is not used and no
Central Portal credentials are involved in this flow.

The upload is GPG-signed: the workflow deploys with the `sign` profile, so `maven-snapshots` receives the artifact
together with its `.asc` signature.

### Release flow (official releases)

The release is prepared locally with the `major`/`minor`/`patch` profiles (see *Profiles* above) and published by CI:

1. Run the preparation with the profile matching the change scope (add `-DdryRun=true` to rehearse first):

    ```bash
    ./mvnw -B -Dminor initialize release:prepare -DpushChanges=false
    ```

   This rewrites the pom to the release version (e.g. `1.1.0`), commits it, tags `v1.1.0`, and moves `main` to the next
   development version computed by the active profile — after a `minor` release of `1.1.0`, development continues as
   `1.2.0-SNAPSHOT` — no manual pom editing.

2. Push the commits and the tag — the tag name must equal `v@{project.version}` or the workflow guard rejects it:

    ```bash
    git push origin main v1.0.0
    ```

3. The "Deploy Release" workflow proves the build on all three JDKs (21/25/27), verifies the tag against the pom
   version, normalizes the tag message (see *Release tag message* below) and then publishes the artifact from JDK 21
   **twice**:

   - **group Nexus** — a signed `./mvnw -B -ntp deploy -P sign`; the standard `distributionManagement` upload of the
  artifact and its `.asc` GPG signature to `maven-releases`;
   - **Maven Central** — `./mvnw -B -ntp deploy -P sign,release-repo`, which arms the Enforcer release guard and
  the `central-publishing-maven-plugin`, and uploads the signed bundle to the Central Portal.

4. Finalize the Central publication in the Portal UI. With the default `publishingType` (USER_MANAGED) the deployment
   stays in the *Validated* state until you press **Publish** at https://central.sonatype.com/publishing/deployments;
   only then does the version become publicly resolvable from Maven Central. Setting `<autoPublish>true</autoPublish>`
   on the `central-publishing-maven-plugin` in the parent POM removes this manual step, at the cost of losing the final
   review gate.

#### Credentials and secrets (Deploy Release workflow)

| Secret                                                         | Used for                                                                                                                |
|----------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| `NEXUS_MAVEN_USERNAME` / `NEXUS_MAVEN_PASSWORD`                | upload credentials for the group Nexus (`maven-releases`)                                                               |
| `MAVEN_CENTRAL_PORTAL_USERNAME` / `MAVEN_CENTRAL_PORTAL_TOKEN` | Central Portal user token (portal *Generate User Token* action), wired to the `central` server of `.mvn/settings.xml`   |
| `GPG_PRIVATE_KEY`                                              | ASCII-armored private key that signs the artifacts; its public counterpart must be registered with the Portal namespace |
| `GPG_PASSPHRASE`                                               | passphrase of that key, exposed to `maven-gpg-plugin` as `MAVEN_GPG_PASSPHRASE`                                         |

#### Notes on the Central publication

- `central-publishing-maven-plugin` is declared with `<extensions>true</extensions>`, so when it is active it
  **replaces** the `maven-deploy-plugin` bindings at runtime. That is why the workflow runs two separate `deploy`
  executions: the `-P sign` one goes to the Nexus, the `-P sign,release-repo` one goes to Central — a single
  invocation cannot reach both.
- Maven Central requires a GPG signature for every published file; the `sign` profile attaches
  `maven-gpg-plugin`, pre-configured in the parent's `pluginManagement` (see *Profiles* above).
- Both destinations are immutable. If the workflow fails halfway (e.g. the Central upload fails after the Nexus
  deployment succeeded), do not simply re-run: delete the already-uploaded artifact from `maven-releases` (Nexus
  rejects redeployment) and, if the Portal received the bundle, drop that deployment in the Portal UI before retrying.

For a version that does not fit the major/minor/patch scheme (e.g. a custom pre-release), pass the versions explicitly
instead of activating a profile. The plugin then also moves `main` to the next development version in a second commit —
`main` must always carry a `-SNAPSHOT` version, otherwise the Deploy Snapshot guard rejects the next push:

```bash
# example: cut 1.0.0, continue development as 1.1.0-SNAPSHOT
./mvnw -B release:prepare -DpushChanges=false -DreleaseVersion=1.0.0 -DdevelopmentVersion=1.1.0-SNAPSHOT

git push origin main v1.0.0
```

No `-Dtag` is needed any more: the `v@{project.version}` tag format now lives in the parent's `pluginManagement`, so
profile-less runs create the same `vX.Y.Z` tags and the `deploy-release` trigger fires unchanged. (One exception: a
hotfix branch cut from a tag created **before** this change still carries the old pom, so it needs `-Dtag=vX.Y.Z` — or
merge the current parent pom into the branch first.) The Deploy Release guard then applies unchanged. As always, do not
run `release:perform` — CI publishes from the tag.

### Release tag message

`release:prepare` creates the tag itself and hardcodes its annotation as `[maven-release-plugin] copy for tag vX.Y.Z`;
the plugin exposes no parameter to change it (unlike the two release commit messages, which are configurable in the
pom). Instead of re-tagging by hand, the **Deploy Release** workflow normalizes the message automatically: once the
tag/version guard has passed, it re-tags the *same commit* as `Release X.Y.Z` (e.g. `Release 1.0.0`) and force-pushes
the tag before deploying. The step is idempotent — a tag whose message is already normalized is left untouched — and it
cannot loop, because pushes made with `GITHUB_TOKEN` never trigger further workflow runs. No local action is required in
either release flow; the pushed tag simply ends up with a clean, version-based annotation in GitHub's UI.

---

## Using it in new group projects

1) Copy the `.mvn` folder (containing `settings.xml` and `maven.config`) from this repository into the new project, so
   the parent resolves from the group repositories and the local repository stays consistent.

2) Declare this parent in the new project's `pom.xml`:

```xml
<parent>
  <groupId>ir.artanpg</groupId>
  <artifactId>artan-parent</artifactId>
  <version>1.0.0</version> <!-- or X.Y.Z-SNAPSHOT for the development version -->
</parent>
```

From this point on, dependency version management, plugin configuration, test and coverage rules, and the profiles above
are in effect in your project without any extra setup. Activate each capability by invoking the matching profile:

```bash
./mvnw -P execute-tests verify              # tests + coverage
./mvnw -P documentation package             # sources + javadoc
./mvnw -P format validate                   # code-style check
./mvnw -P quality verify                    # PMD + SpotBugs
./mvnw -Dpatch initialize release:prepare   # cut a patch release (CI publishes from the tag)
./mvnw -Dminor initialize release:prepare   # cut a minor release (CI publishes from the tag)
./mvnw -Dmajor initialize release:prepare   # cut a major release (CI publishes from the tag)
./mvnw -P sign deploy                       # signed deploy to distributionManagement (SNAPSHOT or release)
./mvnw -P snapshot-repo deploy              # SNAPSHOT deploy to the group Nexus (version guard + signing)
./mvnw -P release-repo deploy               # release deploy to Maven Central (version guard + signing)
```

---

## CI/CD

Three workflows are defined in `.github/workflows/`:

| Workflow                                                       | Trigger                                               | Job                                                                                         |
|----------------------------------------------------------------|-------------------------------------------------------|---------------------------------------------------------------------------------------------|
| [ci.yaml](.github/workflows/ci.yaml)                           | push/PR on `main` (docs-only changes skipped), manual | Build and verify on the JDK 21 / 25 / 27 matrix                                             |
| [deploy-snapshot.yaml](.github/workflows/deploy-snapshot.yaml) | push to `main` (docs-only changes skipped), manual    | Matrix build, then publish the signed SNAPSHOT to Nexus from JDK 21                         |
| [deploy-release.yaml](.github/workflows/deploy-release.yaml)   | push of a `v*` tag                                    | Matrix build, tag/version guard, then publish the signed Release to Nexus and Maven Central |

Temurin JDK builds are used; the production line always goes through JDK 21 (the stable LTS).

---

## License

artan-parent is Open Source software released under the [Apache 2.0 license](https://www.apache.org/licenses/LICENSE-2.0.html).
