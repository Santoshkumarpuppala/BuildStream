# Bitbucket Migration Guide

BuildStream is designed with SCM portability in mind. The shared pipeline library (`mobile-lib`) contains zero GitHub-specific logic. Migrating from GitHub to Bitbucket Cloud or Bitbucket Data Center requires changes in three places: the Jenkins SCM configuration, the webhook setup, and the application Jenkinsfile library reference. The pipeline stages themselves are unaffected.

---

## What changes and what does not

| Component | GitHub | Bitbucket | Change required |
|---|---|---|---|
| `mobilePipeline.groovy` | No change | No change | None |
| `IosPipeline.groovy` | No change | No change | None |
| `AndroidPipeline.groovy` | No change | No change | None |
| Application `Jenkinsfile` | `@Library('mobile-lib') _` | `@Library('mobile-lib') _` | None |
| Global Pipeline Library URL | GitHub HTTPS URL | Bitbucket HTTPS or SSH URL | Yes |
| Checkout credential | GitHub PAT | Bitbucket App Password or SSH key | Yes |
| Webhook trigger | GitHub webhook | Bitbucket webhook | Yes |
| Branch source plugin | GitHub Branch Source | Bitbucket Branch Source | Yes |

---

## 1. Migrate source repositories to Bitbucket

### Option A — Mirror from GitHub (read-only)

Use Bitbucket's built-in repository mirroring to keep a synchronized copy:

1. Create a new repository in Bitbucket (e.g., `mobile-lib`).
2. **Repository settings → Mirroring → Add mirror**.
3. Point the mirror at the GitHub remote.
4. Set the sync frequency (15 minutes or triggered).

The mirror is read-only. To accept contributions through Bitbucket, use Option B.

### Option B — Full migration

```bash
# Clone from GitHub with full history
git clone --mirror https://github.com/Santoshkumarpuppala/mobile-lib.git

# Push to Bitbucket
cd mobile-lib.git
git remote set-url origin https://your-workspace@bitbucket.org/your-workspace/mobile-lib.git
git push --mirror
```

Repeat for `myhub-ios`, `myhub-android`, and `buildstream`.

---

## 2. Create a Bitbucket App Password

Bitbucket does not use personal access tokens in the same way as GitHub. Use an **App Password** for Jenkins authentication.

1. Log in to Bitbucket → **Personal settings → App passwords → Create app password**.
2. Grant the following permissions:

| Permission | Required for |
|---|---|
| Repositories: Read | Cloning during pipeline checkout |
| Repositories: Write | Updating commit statuses |
| Webhooks: Read and Write | Jenkins webhook management (optional) |
| Pull requests: Read | Multi-branch pipeline PR detection |

3. Copy the generated password — it is shown only once.

---

## 3. Store Bitbucket credentials in Jenkins

**Manage Jenkins → Credentials → System → Global credentials → Add Credentials**

| Field | Value |
|---|---|
| Kind | Username with password |
| Username | Your Bitbucket username |
| Password | The App Password from step 2 |
| ID | `bitbucket-app-password` |
| Description | Bitbucket App Password for pipeline checkout |

For SSH authentication instead:

| Field | Value |
|---|---|
| Kind | SSH Username with private key |
| Username | `git` |
| Private key | Paste the private key matching the public key added to your Bitbucket account |
| ID | `bitbucket-ssh-key` |

---

## 4. Update the Global Pipeline Library SCM configuration

**Manage Jenkins → Configure System → Global Pipeline Libraries**

Change the `mobile-lib` library entry:

| Setting | GitHub value | Bitbucket value |
|---|---|---|
| Repository URL | `https://github.com/Santoshkumarpuppala/mobile-lib.git` | `https://bitbucket.org/your-workspace/mobile-lib.git` |
| Credentials | `github-username-pat` | `bitbucket-app-password` |
| Branch | `main` | `main` |

The library name (`mobile-lib`) and the application `Jenkinsfile` contents remain unchanged.

---

## 5. Install the Bitbucket Branch Source plugin

The Bitbucket Branch Source plugin enables multi-branch pipeline jobs that auto-discover branches and pull requests.

**Manage Jenkins → Plugin Manager → Available plugins**

Search for and install:
- **Bitbucket Branch Source Plugin**
- **Bitbucket Server Integration Plugin** (if using Bitbucket Data Center)

---

## 6. Configure webhooks in Bitbucket

Bitbucket webhooks notify Jenkins of push events and pull request activity.

### Bitbucket Cloud

1. Navigate to the application repository → **Repository settings → Webhooks → Add webhook**.
2. Configure:

| Field | Value |
|---|---|
| Title | Jenkins BuildStream trigger |
| URL | `http://<jenkins-public-ip>:8080/bitbucket-hook/` |
| Active | Enabled |
| Triggers | Repository push, Pull request created, Pull request updated |

3. Repeat for `myhub-ios` and `myhub-android`.

### Bitbucket Data Center (Server)

Use the Jenkins Bitbucket Server Integration plugin's native webhook endpoint:

```
http://<jenkins-public-ip>:8080/bitbucket-scmsource-hook/notify
```

Configure in Bitbucket Server under **Administration → Hooks → Jenkins**.

---

## 7. Update pipeline checkout in `mobilePipeline.groovy`

The Checkout stage in `vars/mobilePipeline.groovy` specifies the credential ID used to clone the application repository. Update it to reference the Bitbucket credential:

```groovy
// Before (GitHub)
git credentialsId: 'github-username-pat',
    url: "https://github.com/Santoshkumarpuppala/myhub-${platform}.git",
    branch: branch

// After (Bitbucket Cloud)
git credentialsId: 'bitbucket-app-password',
    url: "https://bitbucket.org/your-workspace/myhub-${platform}.git",
    branch: branch
```

No other changes to the pipeline library are required.

---

## 8. GitFlow branch strategy parity

BuildStream's GitFlow strategy works identically on Bitbucket. The `isMain` guard in `mobilePipeline.groovy` evaluates `env.BRANCH_NAME`, which Jenkins populates from whatever SCM is configured. No changes are needed.

| Branch type | Build stages | Distribute stage |
|---|---|---|
| `main` | Checkout → Build & Test → Sign → Upload to S3 | Yes |
| `release/*` | Checkout → Build & Test → Sign → Upload to S3 | No (configurable) |
| `feature/*` | Checkout → Build & Test | No |
| `hotfix/*` | Checkout → Build & Test → Sign → Upload to S3 | No (configurable) |

To promote `release/*` branches to also trigger distribution, extend the `isMain` condition in `mobilePipeline.groovy`:

```groovy
def isMain = (branch == 'main' || branch.startsWith('release/'))
```

---

## 9. Pull request builds

To run pipeline builds on pull requests (pre-merge validation), create a **Multibranch Pipeline** job instead of a standard Pipeline job:

1. **New Item → Multibranch Pipeline**.
2. Under **Branch Sources**, add a Bitbucket source.
3. Set credentials to `bitbucket-app-password`.
4. Set repository URL to the Bitbucket application repository.
5. Under **Behaviors**, enable **Discover pull requests from origin**.

Jenkins will automatically create a sub-pipeline for each open pull request and report build status back to Bitbucket.

---

## Summary

The migration is a configuration change, not a code change. The `mobile-lib` shared library, all application `Jenkinsfile` contents, and all pipeline stages run identically on Bitbucket. Only the credential ID, repository URLs, and Jenkins SCM plugin differ.
