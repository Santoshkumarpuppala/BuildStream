# UrbanCode Deploy Integration Guide

IBM UrbanCode Deploy (UCD) is an enterprise release automation platform widely used in financial services, healthcare, and government organizations. This guide shows how to connect BuildStream's Jenkins pipeline to UCD so that signed mobile artifacts (IPA and AAB) produced by the pipeline become UCD component versions that UCD processes can deploy to managed environments.

---

## Integration overview

```
GitHub push
    │
    ▼
Jenkins pipeline (mobile-lib)
    │  Checkout → Build & Test → Sign → Upload to S3
    │
    ▼
UCD Jenkins plugin
    │  Creates component version, uploads artifact, sets properties
    │
    ▼
UCD component (iOS or Android)
    │
    ▼
UCD application process
    │  Deploy to: QA → Staging → Production
    ▼
Target environment
```

The BuildStream pipeline handles everything up through artifact signing and S3 upload. The UCD plugin then publishes that artifact as a versioned UCD component, from which UCD processes manage environment-specific delivery.

---

## 1. Prerequisites

- IBM UrbanCode Deploy server (v7.x or later) accessible from the Jenkins controller
- A UCD user account with permissions to create components, publish versions, and trigger processes
- The Jenkins UCD plugin installed

---

## 2. Install the Jenkins UCD plugin

**Manage Jenkins → Plugin Manager → Available plugins**

Search for: **IBM UrbanCode Deploy**

Install and restart Jenkins.

---

## 3. Configure UCD server connection in Jenkins

**Manage Jenkins → Configure System → IBM UrbanCode Deploy**

| Field | Value |
|---|---|
| UCD server URL | `https://your-ucd-server:8443` |
| Authentication token | UCD authentication token (create in UCD under **Settings → Security → Tokens**) |
| Trust self-signed certificates | Enable if using internal PKI |

Store the token in Jenkins Credentials:

**Manage Jenkins → Credentials → Add → Secret text**

| Field | Value |
|---|---|
| Secret | UCD authentication token value |
| ID | `ucd-auth-token` |
| Description | UCD server authentication token |

---

## 4. Create UCD components

Create one component per platform in UCD.

### iOS component

1. In UCD: **Components → Create Component**

| Setting | Value |
|---|---|
| Name | `myhub-ios` |
| Description | MyHub iOS application — signed IPA |
| Source configuration type | None (artifacts uploaded by Jenkins) |
| Default version type | Full |

2. Add a component property for the S3 path:

**Component → Configuration → Properties → Add Property**

| Property | Description |
|---|---|
| `s3.bucket` | S3 bucket containing the artifact |
| `s3.key` | Full S3 key of the IPA for this version |
| `app.version` | App version string |
| `git.branch` | Source branch that produced this artifact |
| `build.number` | Jenkins build number |

### Android component

Repeat the above with:

| Setting | Value |
|---|---|
| Name | `myhub-android` |
| Description | MyHub Android application — signed AAB |

---

## 5. Create a UCD application and environments

1. **Applications → Create Application** — name it `myhub-mobile`.
2. Add both components to the application.
3. Create environments: `QA`, `Staging`, `Production`.
4. Map components to environments.
5. Add agent resources to each environment matching the deployment targets.

---

## 6. Add UCD publish steps to the BuildStream pipeline

After the S3 upload stage in `mobilePipeline.groovy`, add a UCD publication step. The cleanest way is to call the UCD plugin's `step` inside the existing `script` block or as a new stage.

### In `vars/mobilePipeline.groovy` — add a Publish stage

```groovy
stage('Publish to UCD') {
    when { expression { return isMain } }    // publish only from the main branch
    steps {
        script {
            def ext          = (platform == 'ios') ? 'ipa' : 'aab'
            def componentName = "myhub-${platform}"
            def versionName   = "${branch}-${BUILD_NUM}"
            def s3Key         = "${platform}/${appName}/${branch}/${BUILD_NUM}/${appName}.${ext}"

            // Create the component version and set properties
            step([$class: 'UCDeployPublisher',
                siteName:      'UCD Server',
                component:     componentName,
                versionName:   versionName,
                directoryOffset: '.',
                skip:          false,
                properties:    [
                    "s3.bucket=${S3_BUCKET}",
                    "s3.key=${s3Key}",
                    "git.branch=${branch}",
                    "build.number=${BUILD_NUM}"
                ].join('\n')
            ])

            echo "Published ${componentName} version ${versionName} to UCD"
        }
    }
}
```

No changes to the application `Jenkinsfile` are required — the publish step is centralized in the shared library.

---

## 7. Create UCD deployment processes

### Component process — Deploy iOS IPA

This process runs on the deployment target to pull the signed IPA from S3 and install it via MDM or a distribution service.

Recommended steps in UCD:

| Step | Plugin | Purpose |
|---|---|---|
| Download artifact from S3 | Shell / AWS CLI | `aws s3 cp s3://$(s3.bucket)/$(s3.key) ./$(appName).ipa` |
| Notify MDM server | Shell / HTTP Call | Push IPA to MDM enrollment (e.g., Jamf, Mosyle) |
| Verify installation | Shell | Poll MDM API for install confirmation |
| Cleanup | Shell | Remove local IPA copy |

### Component process — Deploy Android AAB

| Step | Plugin | Purpose |
|---|---|---|
| Download artifact from S3 | Shell / AWS CLI | `aws s3 cp s3://$(s3.bucket)/$(s3.key) ./$(appName).aab` |
| Upload to Google Play | Shell / fastlane | `fastlane supply --aab $(appName).aab --track internal` |
| Verify track status | Shell | Poll Play Developer API for processing status |
| Cleanup | Shell | Remove local AAB copy |

### Application process — Promote to Production

Create a UCD application process that:

1. Runs the component process in QA, waits for approval.
2. Runs the component process in Staging, waits for approval.
3. On approval, runs the component process in Production.

This creates the full promotion gate: BuildStream produces the artifact; UCD manages its controlled rollout across environments.

---

## 8. Triggering UCD deployments from Jenkins

To trigger a UCD deployment process automatically after publishing (for the QA environment):

```groovy
stage('Deploy to QA via UCD') {
    when { expression { return isMain } }
    steps {
        script {
            def versionName = "${branch}-${BUILD_NUM}"

            step([$class: 'UCDeployPublisher',
                siteName:    'UCD Server',
                deploy:      true,
                deployApp:   'myhub-mobile',
                deployEnv:   'QA',
                deployProc:  'Deploy Mobile App',
                versionName: versionName,
                component:   "myhub-${platform}"
            ])
        }
    }
}
```

Staging and Production deployments should remain manual in UCD to preserve release gate control.

---

## 9. Artifact traceability

The component version properties (`s3.bucket`, `s3.key`, `git.branch`, `build.number`) create a full traceability chain:

```
UCD deployment record
    └── component version: myhub-ios/main-42
            ├── s3.bucket:    myhub-mobile-artifacts
            ├── s3.key:       ios/MyHub/main/42/MyHub.ipa
            ├── git.branch:   main
            └── build.number: 42
                    └── Jenkins build #42
                            └── git commit abc1234
```

Any deployment can be traced back to the exact source commit, build log, and S3 artifact that produced it.

---

## 10. Rollback

Because every artifact is retained in S3 with a versioned S3 key, rollback is straightforward:

1. In UCD: select the previous component version (e.g., `main-41`).
2. Run the Deploy process pointing at that version.
3. UCD pulls the previously signed artifact from S3 and deploys it.

No rebuild is required — the signed artifact is immutable in S3.

---

## Summary

| What BuildStream provides | What UCD adds |
|---|---|
| Automated build, test, and signing | Multi-environment deployment gates |
| Versioned, immutable artifacts in S3 | Approval workflows and audit trails |
| Branch-based pipeline control | Promotion automation across QA → Staging → Production |
| CloudWatch build monitoring | UCD deployment history and rollback |

The integration is additive: BuildStream handles the supply side (artifact production); UCD handles the delivery side (controlled deployment). Neither system is required to use the other — UCD can be added to an existing BuildStream installation without changing any pipeline code beyond the two `Publish to UCD` and `Deploy to QA` stages above.
