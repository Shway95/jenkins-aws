# Jenkins Freestyle Project with GitHub Integration

A step-by-step guide to create a Jenkins Freestyle project that automatically builds when you push code to GitHub.

## Prerequisites

- Access to a Jenkins server (e.g., `https://jenkinsacademics.herovired.com`)
- A GitHub account with a repository
- Git installed on your local machine
- GitHub credentials configured in Jenkins (username + Personal Access Token)

---

## Step 1: Create a GitHub Repository

1. Go to [GitHub](https://github.com) and sign in
2. Click the **+** icon (top right) → **New repository**
3. Enter repository name (e.g., `jenkins-aws`)
4. Choose **Public** or **Private**
5. Check **Add a README file**
6. Click **Create repository**

---

## Step 2: Clone the Repository Locally

```bash
git clone https://github.com/<your-username>/jenkins-aws.git
cd jenkins-aws
```

---

## Step 3: Create a New Freestyle Project in Jenkins

1. Log in to your Jenkins server
2. Click **New Item** (left sidebar)
3. Enter a name for your project (e.g., `shway-freestyle-proj`)
4. Select **Freestyle project**
5. Click **OK**

---

## Step 4: Configure General Settings

1. In the project configuration page:
   - **Description**: Add a description (e.g., "Freestyle project with GitHub integration")
   - Under **General**, check **GitHub project**
   - **Project url**: Enter your GitHub repo URL  
     ```
     https://github.com/<your-username>/jenkins-aws/
     ```

---

## Step 5: Configure Source Code Management (SCM)

1. Select **Git** under Source Code Management
2. **Repository URL**: Enter your repo URL
   ```
   https://github.com/<your-username>/jenkins-aws
   ```
3. **Credentials**: Select your GitHub credentials from the dropdown
   - If not available, click **Add** → **Jenkins**
   - Kind: **Username with password**
   - Username: Your GitHub username
   - Password: Your GitHub **Personal Access Token** (not your GitHub password)
   - ID: Give it a name (e.g., `github-creds`)
   - Click **Add**, then select it from the dropdown
4. **Branch Specifier**: Enter `*/main` (or `**` for all branches)

---

## Step 6: Configure Build Triggers

### Option A: SCM Polling (Recommended for beginners)

This makes Jenkins check GitHub every minute for new commits.

1. Under **Build Triggers**, check **Poll SCM**
2. In the **Schedule** field, enter:
   ```
   * * * * *
   ```
   This is a cron expression that means "check every minute".

### Option B: GitHub Webhook (More efficient)

This makes GitHub notify Jenkins immediately on push.

1. Under **Build Triggers**, check **GitHub hook trigger for GITScm polling**
2. Then go to your GitHub repo → **Settings** → **Webhooks** → **Add webhook**:
   - **Payload URL**: `https://<your-jenkins-url>/github-webhook/`
   - **Content type**: `application/json`
   - **Secret**: Leave blank
   - **Events**: Select "Just the push event"
   - Click **Add webhook**

> **Note**: You can enable both triggers. SCM polling is simpler (no webhook setup needed), but checks every minute. Webhooks trigger instantly but require additional configuration.

---

## Step 7: Add a Build Step

1. Scroll down to **Build Steps**
2. Click **Add build step** → **Execute shell**
3. Enter your shell script. Example:

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

4. Click **Save**

---

## Step 8: Test the Pipeline

### Manual Test

1. Go to your project page in Jenkins
2. Click **Build Now** (left sidebar)
3. Check **Build History** → click the build number → **Console Output**
4. You should see the shell script output with build info

### Automatic Test (via Git Push)

1. Make a change in your local repo:
   ```bash
   echo "Testing Jenkins auto-build" > test.txt
   git add test.txt
   git commit -m "test: trigger Jenkins build"
   git push origin main
   ```
2. Wait up to 1 minute (if using SCM polling)
3. Check Jenkins — a new build should appear automatically
4. Click the build number → **Console Output** to verify

---

## Step 9: Verify Auto-Trigger is Working

1. In Jenkins, go to your project page
2. Look at **Build History** — you should see a new build
3. The build should say **"Started by an SCM change"** (for polling) or **"Started by GitHub push"** (for webhook)
4. Click the build → **Console Output** to see the commit that triggered it

---

## Project Structure

```
jenkins-aws/
├── README.md        # This documentation
└── test.txt         # Test file for triggering builds
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Build not triggering | Check if SCM polling is enabled (`* * * * *`) |
| Git clone fails | Verify credentials are correct (use Personal Access Token, not password) |
| "Permission denied" in build | Check Jenkins user has execute permissions |
| Webhook not working | Verify webhook URL ends with `/github-webhook/` (trailing slash matters) |
| Wrong branch building | Check Branch Specifier matches your branch name |

---

## Key Concepts

- **Freestyle Project**: A simple Jenkins job type that runs shell commands
- **SCM Polling**: Jenkins periodically checks the repo for new commits (cron-based)
- **GitHub Webhook**: GitHub notifies Jenkins immediately when code is pushed
- **Build Trigger**: The event that starts a Jenkins build
- **Console Output**: Where you see the build logs and results

---

## Cron Expression Reference

| Expression | Meaning |
|-----------|---------|
| `* * * * *` | Every minute |
| `H/5 * * * *` | Every 5 minutes |
| `H/15 * * * *` | Every 15 minutes |
| `H * * * *` | Every hour |
| `H 9 * * 1-5` | 9 AM on weekdays |

> **Tip**: Use `H` instead of `*` for the minute field in production to spread load across Jenkins.

---

## Final Configuration Summary

| Setting | Value |
|---------|-------|
| Project Type | Freestyle |
| SCM | Git |
| Repository | `https://github.com/<your-username>/jenkins-aws` |
| Branch | `*/main` |
| Trigger | Poll SCM: `* * * * *` |
| Build Step | Execute shell (Hello World script) |
