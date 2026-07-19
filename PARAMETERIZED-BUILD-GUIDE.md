# Jenkins Parameterized Freestyle Build - Complete Guide

This document explains **everything** we did to create the `shway-freestyle-project` pipeline — a parameterized Jenkins Freestyle job with VERSION, SKIP_TESTS, ENVIRONMENT, and BRANCH parameters.

---

## What is a Parameterized Build?

A parameterized build lets you **pass inputs** to your Jenkins job before it runs. Instead of a fixed build, you can choose things like:
- Which branch to build
- Which environment to deploy to (dev/test/prod)
- Whether to skip tests
- What version number to use

When you click "Build with Parameters", Jenkins shows you a form to fill in before the build starts.

---

## Why We Added Parameters

| Parameter | Why It's Useful |
|-----------|----------------|
| `BRANCH` | So you can build from any branch without changing job config |
| `ENVIRONMENT` | Deploy to dev for testing, prod for release — same job, different target |
| `SKIP_TESTS` | Speed up builds when you just want to deploy quickly (skip slow tests) |
| `VERSION` | Tag your build with a version number for tracking releases |

---

## Step-by-Step: How We Created This Pipeline

### Step 1: Create a New Freestyle Project

1. Log into Jenkins → Click **New Item**
2. Name: `shway-freestyle-project`
3. Select: **Freestyle project**
4. Click **OK**

---

### Step 2: Add GitHub Project URL

**Where:** General section → check **GitHub project**

**What we entered:**
```
https://github.com/Shway95/jenkins-aws/
```

**Why:** This links your Jenkins job to the GitHub repo. Jenkins shows a "GitHub" link on the job page so you can quickly jump to the repo.

---

### Step 3: Enable Parameters

**Where:** General section → check **This project is parameterized**

This enables the "Build with Parameters" button instead of just "Build Now".

---

### Step 4: Add the BRANCH Parameter

**Where:** Click **Add Parameter** → **String Parameter**

| Field | Value |
|-------|-------|
| Name | `BRANCH` |
| Default Value | `main` |
| Description | `I want to run this build from branch main` |

**Why:** Lets you specify which git branch to build. Default is `main` so you don't have to type it every time.

**How it works in the shell script:**
```bash
echo "Branch: ${BRANCH}"
```
Jenkins injects the parameter as an environment variable `${BRANCH}`.

---

### Step 5: Add the ENVIRONMENT Parameter

**Where:** Click **Add Parameter** → **Choice Parameter**

| Field | Value |
|-------|-------|
| Name | `ENVIRONMENT` |
| Choices | `dev` (line 1), `test` (line 2), `prod` (line 3) |
| Description | `Target deployment environment` |

**Why:** A choice parameter shows a dropdown menu. The user picks one option — can't type a wrong value. First choice (`dev`) is the default.

**How it works in the shell script:**
```bash
case "${ENVIRONMENT}" in
  dev)
    echo "Deploying to dev cluster..."
    ;;
  test)
    echo "Deploying to staging cluster..."
    ;;
  prod)
    echo "Deploying to production cluster..."
    ;;
esac
```

---

### Step 6: Add the SKIP_TESTS Parameter

**Where:** Click **Add Parameter** → **Boolean Parameter**

| Field | Value |
|-------|-------|
| Name | `SKIP_TESTS` |
| Default | ✅ checked (true) |

**Why:** A boolean parameter shows a checkbox. When checked = `true`, when unchecked = `false`. Default is `true` (skip tests) because most quick builds don't need tests.

**How it works in the shell script:**
```bash
if [ "${SKIP_TESTS}" = "false" ]; then
    echo "=== Running tests ==="
    # mvn test
    echo "Tests passed"
else
    echo "=== Tests SKIPPED (as requested) ==="
fi
```

**Real-world Maven usage:**
```bash
# If SKIP_TESTS is true, add -DskipTests flag to Maven
mvn clean package ${SKIP_TESTS:+-DskipTests}
```

---

### Step 7: Add the VERSION Parameter

**Where:** Click **Add Parameter** → **String Parameter**

| Field | Value |
|-------|-------|
| Name | `VERSION` |
| Default Value | `1.0.${BUILD_NUMBER}` |
| Description | (leave blank or add "Application version") |

**Why:** Auto-generates a version like `1.0.1`, `1.0.2`, `1.0.3` based on build number. You can also override it manually (e.g., `2.0.0` for a major release).

**How `${BUILD_NUMBER}` works:** Jenkins automatically increments this counter every build. So the default version auto-increments.

**How it works in the shell script:**
```bash
echo "Build complete: version ${VERSION}"
```

---

### Step 8: Configure Source Code Management (Git)

**Where:** Source Code Management section → select **Git**

| Field | Value |
|-------|-------|
| Repository URL | `https://github.com/Shway95/jenkins-aws` |
| Credentials | Select your GitHub credentials (username + PAT) |
| Branch Specifier | `**` (builds any branch) |

**Why `**` instead of `*/main`:**
- `*/main` = only builds when main branch changes
- `**` = builds when ANY branch changes (more flexible with the BRANCH parameter)

---

### Step 9: Configure Build Trigger (SCM Polling)

**Where:** Build Triggers section → check **Poll SCM**

**Schedule:**
```
* * * * *
```

**What this means:** Jenkins checks GitHub every minute for new commits. If it finds new commits, it triggers a build.

**Cron format explained:**
```
* * * * *
│ │ │ │ │
│ │ │ │ └── Day of week (0-7, Sunday=0 or 7)
│ │ │ └──── Month (1-12)
│ │ └────── Day of month (1-31)
│ └──────── Hour (0-23)
└────────── Minute (0-59)

* * * * *  = Every minute
H/5 * * * * = Every 5 minutes
H * * * * = Every hour
```

**Why SCM Polling vs Webhook:**
- SCM Polling: Simple, no extra setup, Jenkins checks periodically
- Webhook: Instant trigger, but needs GitHub webhook configuration

---

### Step 10: Add Build Step 1 — System Info

**Where:** Build Steps → **Add build step** → **Execute shell**

```bash
#!/bin/bash
set -euo pipefail

echo "========================================="
echo "  Jenkins Freestyle Job - Hello World    "
echo "========================================="
echo ""
echo "Build Information:"
echo "  Build Number:  ${BUILD_NUMBER}"
echo "  Build ID:      ${BUILD_ID}"
echo "  Job Name:      ${JOB_NAME}"
echo "  Workspace:     ${WORKSPACE}"
echo "  Jenkins URL:   ${JENKINS_URL}"
echo "  Node Name:     ${NODE_NAME}"
echo ""
echo "System Information:"
echo "  Hostname:      $(hostname)"
echo "  User:          $(whoami)"
echo "  Date:          $(date)"
echo "  OS:            $(uname -a)"
echo ""
echo "Build SUCCESS!"
```

**What `set -euo pipefail` means:**
- `-e` = Exit immediately if any command fails
- `-u` = Treat unbound (undefined) variables as errors
- `-o pipefail` = If any command in a pipe fails, the whole pipe fails

**Built-in Jenkins variables used:**
| Variable | What It Contains |
|----------|-----------------|
| `${BUILD_NUMBER}` | Auto-incrementing build counter (1, 2, 3...) |
| `${BUILD_ID}` | Same as build number |
| `${JOB_NAME}` | Name of the Jenkins job |
| `${WORKSPACE}` | Directory where Jenkins checks out your code |
| `${JENKINS_URL}` | URL of your Jenkins server |
| `${NODE_NAME}` | Which Jenkins agent ran the build |

---

### Step 11: Add Build Step 2 — Repository Info

**Where:** Build Steps → **Add build step** → **Execute shell** (again)

```bash
#!/bin/bash
set -euo pipefail

echo "=== Repository Information ==="
echo "Branch: $(git branch --show-current)"
echo "Commit: $(git log -1 --oneline)"
echo "Author: $(git log -1 --format='%an <%ae>')"
echo ""
echo "=== Repository Contents ==="
ls -la
echo ""
echo "=== Recent Commits ==="
git log --oneline -5
```

**Why this step:** Shows what code Jenkins actually checked out — which commit, who wrote it, and what files are in the repo. Useful for debugging.

**Git commands explained:**
| Command | What It Shows |
|---------|--------------|
| `git branch --show-current` | Current branch name |
| `git log -1 --oneline` | Latest commit (short hash + message) |
| `git log -1 --format='%an <%ae>'` | Author name and email |
| `ls -la` | All files in the workspace |
| `git log --oneline -5` | Last 5 commits |

---

### Step 12: Add Build Step 3 — Parameterized Build & Deploy

**Where:** Build Steps → **Add build step** → **Execute shell** (again)

```bash
#!/bin/bash
set -euo pipefail

echo "========================================="
echo "  Parameterized Build"
echo "========================================="
echo "Branch:      ${BRANCH}"
echo "Environment: ${ENVIRONMENT}"
echo "Version:     ${VERSION}"
echo "Skip Tests:  ${SKIP_TESTS}"
echo "Build:       #${BUILD_NUMBER}"
echo ""

# Build phase
echo "=== Building application ==="
# mvn clean package ${SKIP_TESTS:+-DskipTests}
echo "Build complete: version ${VERSION}"

# Conditional test execution
if [ "${SKIP_TESTS}" = "false" ]; then
    echo "=== Running tests ==="
    # mvn test
    echo "Tests passed"
else
    echo "=== Tests SKIPPED (as requested) ==="
fi

# Environment-specific deployment
echo "=== Deploying to ${ENVIRONMENT} ==="
case "${ENVIRONMENT}" in
  dev)
    echo "Deploying to dev cluster..."
    # kubectl apply -f k8s/dev/
    ;;
  test)
    echo "Deploying to staging cluster..."
    # kubectl apply -f k8s/staging/
    ;;
  prod)
    echo "Deploying to production cluster..."
    # kubectl apply -f k8s/production/
    ;;
esac

echo ""
echo "Deployment to ${ENVIRONMENT} complete!"
```

**What this step does:**
1. Prints all parameters so you can verify what was selected
2. Simulates a Maven build (commented out `mvn` commands — replace with real ones for a real project)
3. Conditionally runs or skips tests based on `SKIP_TESTS`
4. Deploys to the selected environment using a `case` statement

**The `case` statement:** Like a switch/if-else — runs different code depending on which environment was selected.

---

### Step 13: Save and Run

1. Click **Save**
2. Click **Build with Parameters**
3. Fill in the form:
   - BRANCH: `main`
   - ENVIRONMENT: `dev`
   - SKIP_TESTS: ✅ (checked)
   - VERSION: `1.0.${BUILD_NUMBER}` (leave default)
4. Click **Build**
5. Check Console Output to verify everything ran

---

## Sample Console Output (Build #2)

```
Started by user herovired
Building on the built-in node in workspace /var/lib/jenkins/workspace/shway-freestyle-project
Fetching changes from the remote Git repository
Checking out Revision 99fa815 (origin/main)
Commit message: "docs: update README with second build step and sample output"

=========================================
  Jenkins Freestyle Job - Hello World    
=========================================

Build Information:
  Build Number:  2
  Build ID:      2
  Job Name:      shway-freestyle-project
  Workspace:     /var/lib/jenkins/workspace/shway-freestyle-project
  Jenkins URL:   http://13.236.67.54:8080/
  Node Name:     built-in

System Information:
  Hostname:      ip-172-31-9-130
  User:          jenkins
  Date:          Sun Jul 19 13:39:52 UTC 2026
  OS:            Linux ip-172-31-9-130 6.17.0-1013-aws

Build SUCCESS!

=== Repository Information ===
Branch: 
Commit: 99fa815 docs: update README with second build step and sample output
Author: shwetang95 <shwetang95@outlook.com>

=== Repository Contents ===
README.md
test.txt

=== Recent Commits ===
99fa815 docs: update README with second build step and sample output
ee13dc4 chore: fresh change to trigger build
06b19d9 chore: update test file - commit #4
bc08e3b docs: add step-by-step Jenkins freestyle project guide
93a595a test: update file to trigger SCM polling

=========================================
  Parameterized Build
=========================================
Branch:      main
Environment: dev
Version:     1.0.2
Skip Tests:  true
Build:       #2

=== Building application ===
Build complete: version 1.0.2
=== Tests SKIPPED (as requested) ===
=== Deploying to dev ===
Deploying to dev cluster...

Deployment to dev complete!
Finished: SUCCESS
```

---

## Step 14: Add Build Step 4 — Jenkins Predefined Environment Variables

**Where:** Build Steps → **Add build step** → **Execute shell** (add another one)

```bash
#!/bin/bash
echo "=== Jenkins Built-in Environment Variables ==="
echo ""
echo "Build Information:"
echo "  BUILD_NUMBER:     ${BUILD_NUMBER}"
echo "  BUILD_ID:         ${BUILD_ID}"
echo "  BUILD_DISPLAY_NAME: ${BUILD_DISPLAY_NAME}"
echo "  BUILD_URL:        ${BUILD_URL}"
echo "  BUILD_TAG:        ${BUILD_TAG}"

echo ""
echo "Job Information:"
echo "  JOB_NAME:         ${JOB_NAME}"
echo "  JOB_BASE_NAME:    ${JOB_BASE_NAME}"
echo "  JOB_URL:          ${JOB_URL}"

echo ""
echo "Workspace:"
echo "  WORKSPACE:        ${WORKSPACE}"
echo "  WORKSPACE_TMP:    ${WORKSPACE_TMP}"

echo ""
echo "SCM Information:"
echo "  GIT_COMMIT:       ${GIT_COMMIT}"
echo "  GIT_BRANCH:       ${GIT_BRANCH}"
echo "  GIT_URL:          ${GIT_URL}"

echo ""
echo "Node/Agent:"
echo "  NODE_NAME:        ${NODE_NAME}"
echo "  NODE_LABELS:      ${NODE_LABELS}"
echo "  EXECUTOR_NUMBER:  ${EXECUTOR_NUMBER}"

echo ""
echo "Jenkins:"
echo "  JENKINS_URL:      ${JENKINS_URL}"
echo "  JENKINS_HOME:     ${JENKINS_HOME}"
```

**Why this step:** This shows ALL the predefined environment variables that Jenkins automatically provides to every build. You don't define these anywhere — Jenkins creates them for you.

**Why no `set -euo pipefail` here:** Some of these variables (like `WORKSPACE_TMP`) might not exist in all Jenkins versions. Without `-u`, the script won't crash on undefined variables — it'll just print empty.

**What are Predefined Environment Variables?**

These are variables Jenkins automatically injects into every build. You never define them — they're always available:

| Variable | What It Contains | Example Value |
|----------|-----------------|---------------|
| `BUILD_NUMBER` | Auto-incrementing counter | `3` |
| `BUILD_ID` | Same as BUILD_NUMBER | `3` |
| `BUILD_DISPLAY_NAME` | Human-readable build name | `#3` |
| `BUILD_URL` | Full URL to this build | `http://jenkins:8080/job/my-job/3/` |
| `BUILD_TAG` | Unique identifier | `jenkins-my-job-3` |
| `JOB_NAME` | Name of the job | `shway-freestyle-project` |
| `JOB_BASE_NAME` | Job name without folder path | `shway-freestyle-project` |
| `JOB_URL` | URL to the job page | `http://jenkins:8080/job/my-job/` |
| `WORKSPACE` | Where code is checked out | `/var/lib/jenkins/workspace/my-job` |
| `WORKSPACE_TMP` | Temp dir for the build | `/var/lib/jenkins/workspace/my-job@tmp` |
| `GIT_COMMIT` | Full SHA of the commit | `99fa815dc64e5e1f571477434a0a3e4679b52ed7` |
| `GIT_BRANCH` | Branch being built | `origin/main` |
| `GIT_URL` | Git repository URL | `https://github.com/Shway95/jenkins-aws` |
| `NODE_NAME` | Agent that ran the build | `built-in` |
| `NODE_LABELS` | Labels on that agent | `built-in` |
| `EXECUTOR_NUMBER` | Executor slot number | `0` |
| `JENKINS_URL` | Jenkins server URL | `http://13.236.67.54:8080/` |
| `JENKINS_HOME` | Jenkins install directory | `/var/lib/jenkins` |

**Predefined vs User-Defined (Parameters):**

| Type | Source | Example |
|------|--------|---------|
| **Predefined** | Jenkins creates automatically | `BUILD_NUMBER`, `GIT_COMMIT`, `WORKSPACE` |
| **User-Defined (Parameters)** | You define in "This project is parameterized" | `BRANCH`, `ENVIRONMENT`, `SKIP_TESTS`, `VERSION` |

Both types are available as `${VARIABLE_NAME}` in your shell scripts. The difference is predefined ones always exist — you don't need to add them.

---

## How Parameters Flow (Visual)

```
User clicks "Build with Parameters"
         │
         ▼
┌─────────────────────────────┐
│  BRANCH:      [main      ]  │
│  ENVIRONMENT: [dev ▼     ]  │
│  SKIP_TESTS:  [✅]         │
│  VERSION:     [1.0.2     ]  │
│                             │
│        [Build]              │
└─────────────────────────────┘
         │
         ▼
Jenkins injects ALL variables (user params + predefined)
         │
         ▼
┌──────────────────────────────────────────┐
│  Build Step 1: System Info               │ ← Uses ${BUILD_NUMBER}, ${JOB_NAME}
│  Build Step 2: Repo Info                 │ ← Uses git commands
│  Build Step 3: Parameterized Deploy      │ ← Uses ${BRANCH}, ${ENVIRONMENT},
│                                          │   ${VERSION}, ${SKIP_TESTS}
│  Build Step 4: Predefined Env Variables  │ ← Uses ${GIT_COMMIT}, ${BUILD_URL},
│                                          │   ${NODE_NAME}, ${JENKINS_HOME}
└──────────────────────────────────────────┘
         │
         ▼
    Build Result: SUCCESS/FAILURE
```

---

## Parameter Types Explained

| Type | UI Element | Example | Use When |
|------|-----------|---------|----------|
| **String** | Text box | `main`, `1.0.0` | Free-form text input needed |
| **Choice** | Dropdown | `dev/test/prod` | Fixed list of valid options |
| **Boolean** | Checkbox | `true/false` | Yes/No toggle |
| **Password** | Hidden text | `***` | Secrets (API keys, tokens) |
| **File** | File upload | — | Upload a config file to build |

---

## Common Mistakes & How We Fixed Them

### Mistake 1: Unbound Variable Error
**Problem:** Using `set -u` with a parameter that doesn't exist causes:
```
ENVIRONMENT: unbound variable
Build step marked as failure
```

**Fix:** Either:
- Make sure the parameter is defined in "This project is parameterized"
- Use default values: `${ENVIRONMENT:-dev}` (if ENVIRONMENT is empty, use "dev")
- Remove `-u` flag: change `set -euo pipefail` to `set -eo pipefail`

### Mistake 2: Duplicate Branch Specs
**Problem:** Having `*/main` listed twice in branches section.

**Fix:** Remove the duplicate — only need one branch specifier.

### Mistake 3: No Build Steps
**Problem:** Job triggers but does nothing (empty builders section).

**Fix:** Add at least one "Execute shell" build step.

---

## Real-World Usage (For a Spring Boot Maven Project)

Replace the commented-out commands with real ones:

```bash
#!/bin/bash
set -euo pipefail

# Real Maven build
if [ "${SKIP_TESTS}" = "true" ]; then
    mvn clean package -DskipTests -Dversion=${VERSION}
else
    mvn clean package -Dversion=${VERSION}
fi

# Real Docker build
docker build -t myapp:${VERSION} .

# Real Kubernetes deployment
kubectl set image deployment/myapp myapp=myapp:${VERSION} -n ${ENVIRONMENT}
```

---

## Final Configuration Summary

| Setting | Value |
|---------|-------|
| **Job Name** | `shway-freestyle-project` |
| **Type** | Freestyle project |
| **GitHub Repo** | `https://github.com/Shway95/jenkins-aws` |
| **Branch** | `**` (all branches) |
| **Trigger** | SCM Polling every minute (`* * * * *`) |
| **Parameters** | BRANCH, ENVIRONMENT, SKIP_TESTS, VERSION |
| **Build Steps** | 4 shell scripts (System Info → Repo Info → Parameterized Deploy → Predefined Env Vars) |
| **Jenkins URL** | https://jenkinsacademics.herovired.com/job/shway-freestyle-project/ |
