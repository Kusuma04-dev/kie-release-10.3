# Apache KIE 10.3 :: Release procedure

This document describes the pieces that compose the Apache KIE 10.3 release, updated for the consolidated repository architecture and local-first release scripts introduced in 10.3.

- The table at the top contains the 2 active repositories that compose Apache KIE 10.3 (`incubator-kie` and `incubator-kie-tools`) with their respective build commands, version update commands, and notes on produced artifacts.
- The **"Branches strategy"** section depicts the Git timeline for the development stream and short-lived release branches/tags.
- The **"Automations & Script Workflows"** section details the local-first execution model and how Jenkins orchestrates release candidate builds, artifact staging, and final publication.

---

## 1. Repositories Matrix

| # | REPO (`incubator-kie-[repo]`) | GIT REF | OPERATING SYSTEM & REQUIREMENTS | BUILD COMMAND | PRODUCED ARTIFACTS | UPDATE OWN VERSION COMMAND (commits D and R) | UPDATE UPSTREAM VERSIONS (commits D and R) | Additional release command (commit R) |
|---|---|---|---|---|---|---|---|---|
| 1 | **`incubator-kie`** *(Consolidated Drools, OptaPlanner, Kogito Runtimes, and Kogito Apps)* | TAG: `10.3.0` | Ubuntu 22.04+<br>JDK 17.0.12+ (GraalVM JDK 17 for native)<br>Maven 3.9.6+<br>Docker 25+ | `./script/release/build.sh --skip-tests` *(optional: `--jitexecutor-native`)* | JARs, POMs, sources, and javadocs installed to local Maven repository (`~/.m2/repository`) | `./script/release/update-version.sh <version>`<br>*(Handled automatically during RC creation by `rc-commit.sh`)* | n/a (root unified Java reactor) | `./script/release/deploy-to-staging.sh --tag <rc-tag> --deploy` |
| 2 | **`incubator-kie-tools`** | TAG: `10.3.0` | Ubuntu 22.04+<br>Node.js 22<br>pnpm 9.x<br>Go 1.21+<br>Helm 3.x<br>Docker 25+ | `./scripts/release/release-all.sh <version> --rc` | VS Code extensions (`.vsix`), Chrome extension ZIPs, WebApp ZIPs, Sources ZIP, NPM packages ZIP, Container image tarballs, Helm chart tarballs in `release-artifacts/` | `pnpm update-version-to <version>`<br>`pnpm update-stream-name-to <stream-name>` | `pnpm update-kogito-version-to --maven <version>` | `./scripts/release/release-all.sh <version> --publish` |

> **Consolidation Note**: Since the 10.3.x consolidation, `drools`, `optaplanner`, `kogito-runtimes`, and `kogito-apps` are all modules of the same root POM in `incubator-kie`. The release process is a single-repo, single-command workflow.

---

## 2. Branches Strategy

```text
---c-------c-------------------------c----------c-------c---------c--------------------c----------c---c---c-------------c---c------> bmain
   |                                                                                   |                                             v999-SNAPSHOT or v0.0.0
   |                                                                                   '----D------c-------> b10.3.x
   |                                                                                                         v10.3.999-SNAPSHOT or v10.3.999
   '----D-------c----c----c--------------c-------------c-----------c---> b10.3.x	
        |       |                        |                               v10.3.999-SNAPSHOT or v10.3.999
        |       '--R--> t10.3.0-rc2 ✅   '--R--> t10.3.1-rc1 🕑
        |               t10.3.0                  v10.3.1
        |               v10.3.0
        |
        '--R--> t10.3.0-rc1 ❌
                v10.3.0
```

### Legend
- **`c`** = regular commit
- **`D`** = dev version commit → version updated to `major.minor.999-SNAPSHOT` (or `major.minor.999` in `kie-tools`) and upstream versions aligned.
- **`R`** = release commit → version bumped to exact release version (`10.3.0`).
- **`t`** = Git tag (`10.3.0-rcN` or `10.3.0`).
- **`b`** = Git branch (`main`, `10.3.x`).
- **`v`** = Project version.

### Tag & Stream Rules
- **Release Tags**: `major.minor.patch-rcN` (e.g. `10.3.0-rc1`) and `major.minor.patch` (e.g. `10.3.0`).
- Tags point to an **R commit** created on a temporary local release branch that is deleted after tagging.
- Development stream branches (`10.3.x`) remain on `10.3.999-SNAPSHOT` (`10.3.999` in `kie-tools`).

---

## 3. Automations & Local-First Workflows

---

### AUTOMATION A: Create New "Minor" Version Development Stream (e.g., `10.3.x`)

**Description**: Sets up the `10.3.x` release stream branches on `incubator-kie` and `incubator-kie-tools`.

**Execution Steps**:
1. **`incubator-kie`**:
   - Create branch `10.3.x` from `main`.
   - Update versions to stream snapshot: `./script/release/update-version.sh 10.3.999-SNAPSHOT`.
   - Push branch `10.3.x` to `origin`.
2. **`incubator-kie-tools`**:
   - Create branch `10.3.x` from `main`.
   - Update versions:
     ```bash
     pnpm update-version-to 10.3.999
     pnpm update-stream-name-to 10.3.x
     pnpm update-kogito-version-to --maven 10.3.999-SNAPSHOT
     ```
   - Push branch `10.3.x` to `origin`.

---

### AUTOMATION B & C: CI / CD & Nightly SNAPSHOT Pipelines

- Automated daily builds publish snapshot container images (`10.3.x` tags) and Maven SNAPSHOT libraries (`10.3.999-SNAPSHOT`).
- Jenkins pipeline configurations under `.ci/jenkins/` point to stream branches.

---

### AUTOMATION D: Release Candidate Generation (Local-First or Jenkins)

**Inputs**:
- Version: `10.3.0`
- RC Tag: `10.3.0-rc1`

#### Step D.1: Java Reactor Release Candidate (`incubator-kie`)
Run from the `incubator-kie` repository:
```bash
./script/release/release-all.sh \
    --version 10.3.0 \
    --tag 10.3.0-rc1 \
    --skip-tests \
    --deploy \
    --push-tag
```

**Actions Executed**:
1. `rc-commit.sh`: Checks out temporary branch, runs `update-version.sh 10.3.0`, commits the R commit, tags `10.3.0-rc1`, and pushes the tag.
2. `build.sh`: Runs `mvn clean install -DskipTests -Dfull` across all reactor modules.
3. `deploy-to-staging.sh`: Signs artifacts with GPG and deploys to Apache Nexus Staging (`https://repository.apache.org/service/local/staging/deploy/maven2`).
4. **Action**: Log into [repository.apache.org](https://repository.apache.org), inspect and close the staging repository.

---

#### Step D.2: KIE Tools Release Candidate (`incubator-kie-tools`)
Run from the `incubator-kie-tools` repository:
```bash
# 1. Update versions
pnpm update-version-to 10.3.0
pnpm update-kogito-version-to --maven 10.3.0
pnpm update-stream-name-to 10.3.0

# 2. Package all release artifacts
./scripts/release/release-all.sh 10.3.0 --rc
```

**Artifacts Produced in `release-artifacts/`**:

All artifacts adhere strictly to the Apache Incubator naming format: `apache-kie-10.3.0-incubating-<artifact-name>.<ext>`

1. **VS Code Extensions (`.vsix`)**:
   - `apache-kie-10.3.0-incubating-bpmn-vscode-extension.vsix`
   - `apache-kie-10.3.0-incubating-dmn-vscode-extension.vsix`
   - `apache-kie-10.3.0-incubating-drl-vscode-extension.vsix` (*NEW in 10.3.0 via drools-lsp*)
   - `apache-kie-10.3.0-incubating-pmml-vscode-extension.vsix`
   - `apache-kie-10.3.0-incubating-kogito-bundle-vscode-extension.vsix`
   - `apache-kie-10.3.0-incubating-business-automation-bundle-vscode-extension.vsix`
   - `apache-kie-10.3.0-incubating-extended-services-vscode-extension.vsix`

2. **Chrome Extensions (`.zip`)**:
   - `apache-kie-10.3.0-incubating-business-automation-chrome-extension.zip`
   - `apache-kie-10.3.0-incubating-business-automation-chrome-extension-editors.zip`

3. **Web Applications, Accelerators & Sources**:
   - `apache-kie-10.3.0-incubating-sandbox-webapp.zip`
   - `apache-kie-10.3.0-incubating-sandbox-accelerator-quarkus.zip`
   - `apache-kie-10.3.0-incubating-sources.zip`
   - `apache-kie-10.3.0-incubating-tools-npm-packages.zip`

4. **Helm Charts (`helm-charts/`)**:
   - `apache-kie-10.3.0-incubating-sandbox-helm-chart.tar.gz`
   - `apache-kie-10.3.0-incubating-runtime-tools-console-helm-chart.tar.gz`

5. **Container Images (`container-images/`)**:
   - `apache-kie-10.3.0-incubating-cors-proxy-image.tar.gz`
   - `apache-kie-10.3.0-incubating-kogito-management-console-image.tar.gz`
   - `apache-kie-10.3.0-incubating-sandbox-dev-deployment-base-image.tar.gz`
   - `apache-kie-10.3.0-incubating-sandbox-dev-deployment-dmn-form-webapp-image.tar.gz`
   - `apache-kie-10.3.0-incubating-sandbox-dev-deployment-quarkus-blank-app-image.tar.gz`
   - `apache-kie-10.3.0-incubating-sandbox-extended-services-image.tar.gz`
   - `apache-kie-10.3.0-incubating-sandbox-webapp-image.tar.gz`

---

#### Step D.3: Upload to Apache SVN Dev Distribution
```bash
# Checkout SVN dev dist
svn co https://dist.apache.org/repos/dist/dev/incubator/kie/ /tmp/kie-dist-dev
mkdir -p /tmp/kie-dist-dev/10.3.0-rc1

# Sign and checksum all artifacts
cd incubator-kie-tools/release-artifacts
for f in $(find . -type f); do
    gpg --armor --detach-sign "$f"
    sha512sum "$f" > "${f}.sha512"
done

# Copy to SVN and commit
cp -r . /tmp/kie-dist-dev/10.3.0-rc1/
cd /tmp/kie-dist-dev
svn add 10.3.0-rc1
svn commit -m "Apache KIE 10.3.0-rc1 release candidate artifacts"
```

---

### AUTOMATION E: Voting Procedure (72h KIE Podling + 72h IPMC)

Send vote email to `dev@kie.apache.org` containing:
- Nexus Staging Repository URL
- SVN Dev Dist URL (`https://dist.apache.org/repos/dist/dev/incubator/kie/10.3.0-rc1/`)
- Git tags: `10.3.0-rc1` in `incubator-kie` and `incubator-kie-tools`
- PPG KEYS: `https://downloads.apache.org/incubator/kie/KEYS`

---

### AUTOMATION F: Tag Official Release

Once vote passes:
```bash
# In incubator-kie
./script/release/tag-release.sh --rc-tag 10.3.0-rc1 --push

# In incubator-kie-tools
git tag -a 10.3.0 -m "Apache KIE 10.3.0 Release" 10.3.0-rc1
git push origin 10.3.0
```

---

### AUTOMATION G: Publish Release Candidate to Public Registries

1. **Nexus Maven Release**:
   - Log in to `https://repository.apache.org` and release the closed staging repository.

2. **Move SVN dist dev to release**:
   ```bash
   svn move -m "Release Apache KIE 10.3.0" \
       https://dist.apache.org/repos/dist/dev/incubator/kie/10.3.0-rc1 \
       https://dist.apache.org/repos/dist/release/incubator/kie/10.3.0
   ```

3. **Publish KIE Tools Components (`incubator-kie-tools`)**:
   ```bash
   # Executes npm publish, vsce publish, chrome web store upload, docker push, helm push, gh-pages deploy
   ./scripts/release/release-all.sh 10.3.0 --publish
   ```
   - **NPM Packages**: `@kie-tools/*` published to npm registry (requires `NPM_TOKEN`).
   - **VS Code Extensions**: Published to Visual Studio Marketplace (requires `VSCE_PAT`).
   - **Chrome Extensions**: Uploaded and published to Chrome Web Store via Google API (requires `CHROME_*` credentials).
   - **Container Images**: Pushed to `docker.io/apache/incubator-kie-*` (requires `DOCKER_USERNAME` / `DOCKER_PASSWORD`).
   - **Helm Charts**: Pushed to OCI registry (requires `HELM_REGISTRY`).
   - **GitHub Pages**: Sandbox webapp deployed to `incubator-kie-kogito-online` (`gh-pages` branch) and accelerator tagged.

---

## 4. Summary of Changes Between 10.2 and 10.3

| Area | 10.2.0 | 10.3.0 |
|---|---|---|
| **Java Repositories** | 4 separate repos (`drools`, `optaplanner`, `kogito-runtimes`, `kogito-apps`) | 1 unified reactor in `incubator-kie` |
| **Java Release Command** | Multiple distinct Jenkins jobs per repo | Single local-first `./script/release/release-all.sh` command |
| **New VS Code Extensions** | BPMN, DMN, PMML, Bundles | Added `drl-vscode-extension` (via `drools-lsp`) |
| **Removed Packages** | `dashbuilder-*`, `sonataflow-*`, `yard-*`, `serverless-logic-*` | Cleanly omitted from scripts and release bundles |
| **Tooling Execution** | Jenkins-dependent multi-pipeline orchestration | Local-first dry-run and release scripts callable locally or via CI |
