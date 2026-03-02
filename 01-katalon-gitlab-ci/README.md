🚀 **Katalon TestCloud Integration with GitLab CI/CD — Step-by-Step Guide**

This guide explains how to integrate Katalon Studio TestCloud with GitLab CI/CD to automatically execute automation tests when developers push code — even across different repositories.

👉 Designed for real-world enterprise DevOps QA workflows.

📌 Architecture Overview
Developer Repo (Project / App Repo)
        │
        │  Code Push (staging branch)
        ▼
Trigger Job (Cross-Repo Pipeline)
        ▼
Automation Repo (Katalon Project)
        ▼
Katalon TestCloud Execution
        ▼
TestOps Reporting
        ▼
Pass → Continue Deployment
Fail → Block Pipeline

🎯 Use Case
Whenever developers push code to the staging branch:
✅ Katalon automation tests run automatically
✅ If tests pass → deployment continues
❌ If tests fail → pipeline stops
This creates a QA gate before deployment.

⚙️ Prerequisites
Before setup, ensure the following:
GitLab CI Runner configured (Docker executor recommended)
Katalon Studio project with test suites
Katalon TestOps account
API Key from TestOps
Project linked to TestOps
Separate GitLab repositories:
Application repository (Dev team)
Automation repository (Katalon project)
Docker image available:

📁 Repository Structure
🧪 Automation Repository (Repo B)
Contains:
Katalon Studio project
Test Suites / Test Suite Collections
.gitlab-ci.yml for automation execution

automation-katalon/
 ├── project.prj
 ├── Test Suites/
 ├── Test Suite Collections/
 └── .gitlab-ci.yml

 🔑 Step 1 — Get Required Katalon Credentials
API Key
From TestOps:
User Profile → API Key

Organization ID
From TestOps dashboard URL or settings.

🔐 Step 2 — Configure GitLab CI/CD Variables
In Automation Repo → Settings → CI/CD → Variables
Add:
KATALON_API_KEY = ********
KATALON_PROJECT_PATH = /builds/group/automation/project.prj
TEST_SUITE_COLLECTION = Test Suites/Regression Collection
ORG_ID = 123456

🧪 Step 3 — Automation Repo CI Pipeline
Create .gitlab-ci.yml in Automation Repo:

image: katalonstudio/katalon:latest
stages:
  - katalon_test
katalon_test_job:
  stage: katalon_test
  timeout: 2h
  script:
    - echo "Running Katalon TestCloud Tests..."
    - katalonc.sh -noSplash -retry=0 \
        -projectPath="$KATALON_PROJECT_PATH" \
        -testSuiteCollectionPath="$TEST_SUITE_COLLECTION" \
        -apiKey="$KATALON_API_KEY" \
        -orgID="$ORG_ID" \
        -testOpsUploadReport="true" \
        -testRunName="CI Pipeline - ${CI_PIPELINE_ID}"
  artifacts:
    paths:
      - Reports/**
    when: always
  only:
    refs:
      - qa-automation-branch
     
🔄 Step 4 — Cross-Repository Trigger
Goal:
👉 When Dev repo pipeline runs → trigger automation repo pipeline.

**Approach 1 — API Trigger Using curl**

Step 1 — Create Pipeline Trigger Token

In Repo B:
Settings → CI/CD → Pipeline Triggers
Generate a trigger token.
Step 2 — Add Trigger Job in Repo A

trigger_katalon_pipeline:
  stage: trigger
  script:
    - echo "Triggering Katalon pipeline in another repo..."
    - >
      curl -X POST --fail
      -F token="TRIGGER_TOKEN"
      -F ref="feature"
      https://gitlab.example.com/api/v4/projects/123/trigger/pipeline
  only:
    refs:
      - staging
Problem with this approach
⚠️ Runs asynchronously
⚠️ Dev pipeline does NOT wait for test result
⚠️ Deployment may happen even if tests fail

✅ Approach 2 — Native GitLab Cross-Project Trigger (Recommended)
GitLab provides built-in support using:
Use GitLab’s built-in trigger: keyword.

✔ Waits for test completion
✔ Fails pipeline if tests fail
✔ Enables gated deployments

🧠 Implementation
In Repo B (Automation Repositery)

image: katalonstudio/katalon:11.0.0
stages:
  - katalon_test
variables:
  KATALON_PROJECT_PATH: "$KATALON_PROJECT_PATH"  # Adjust if project path is different
  TEST_SUITE_COLLECTION: "$TEST_SUITE_COLLECTION"
test:
  stage: katalon_test
  script:
    - echo "Running Katalon TestCloud Tests.."
    - katalonc.sh -noSplash -retry=0
        -projectPath="$KATALON_PROJECT_PATH"
        -testSuiteCollectionPath="$TEST_SUITE_COLLECTION"
        -apiKey=$API_KEY
        -orgID="1234" 
  artifacts:
    paths:
      - Reports/**
    when: on_failure 
    expire_in: 1 week
  only:
    refs:
      - qa-automation-branch 

      
🔍 What strategy: depend Does
Waits for downstream pipeline completion
Fails if tests fail
Prevents deployment on failure

🔐 Step 5 — Allow Cross-Project Access
In Automation Repo → Settings → CI/CD → Job Token Permissions
Add the Dev repository path to allow triggering.

🚀 Step 6 — Dev Repo Pipeline Configuration (Repo A)
Create .gitlab-ci.yml in Dev Repo:
stages:
  - test
  - deploy
trigger_katalon_tests:
  stage: test
  trigger:
    project: group/automation-katalon   # Automation repo path
    branch: qa-automation-branch        # Branch containing CI config
    strategy: depend                    # WAIT for results
  only:
    refs:
      - staging
deploy_to_staging:
  stage: deploy
  script:
    - echo "Deploying application to staging..."
  needs:
    - trigger_katalon_tests
  only:
    refs:
      - staging
  dependencies:
     -trigger_katalon_pipeline    # ✅ Ensures deploy waits for test result 


🧠 What strategy: depend Does
This is the key to proper QA gating:
✔ Dev pipeline waits for automation results
✔ If automation fails → Dev pipeline fails
✔ Deployment is blocked
✔ Ensures production safety

📊 Execution Flow

Successful Case
Developer Push → Trigger Automation → Tests Pass → Deploy

Failure Case
Developer Push → Trigger Automation → Tests Fail → Pipeline Stops
