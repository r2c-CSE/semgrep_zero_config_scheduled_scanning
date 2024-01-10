# How to run Semgrep Scheduled scans on all repos in a given GitHub Org: Option-2 using Repository_Dispatch (Final)

# Why?

Customers want to onboard Semgrep quickly without having to go to each team one by one and ask them to add Semgrep to their repo’s workflow files.

They can do this for PR’s using [Repository rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets). 

We can now also accomplish scheduled full scans using the solution below

The solution below can be used in conjunction with the Repository Rulesets to get additional coverage.

# How?


We create a new repo - `appsec-repo` (for example) that is owned by the App Sec team. In this repo we create a workflow file that is triggered on `Workflow_Dispatch` and `Scheduled` trigger. We have 2 different schedules:
* daily
* weekly

<img width="1292" alt="image" src="https://github.com/r2c-CSE/semgrep_zero_config_scheduled_scanning/assets/119853191/a8ad27b9-3343-472a-b2a3-69d488ef9d16">

<img width="1292" alt="image" src="https://github.com/r2c-CSE/semgrep_zero_config_scheduled_scanning/assets/119853191/3f4a4a8e-66ad-47ee-95f6-6adc30516075">

```yaml
on:
  # Scan on-demand through GitHub Actions interface:
  workflow_dispatch: {}
  # Schedule the CI job (this method uses cron syntax):
  schedule:
    - cron: '20 17 * * *' # Sets Semgrep to scan every day at 17:20 UTC.
```

The workflow performs the following steps:

- Retrieve a list of all repos in a given GitHub ORG
- For each of the repos, calls a new GitHub Workflow using HTTP Request (`Repository_Dispatch`)
- The other workflow clones the repo and run the semgrep scan

## Workflow-1: Daily


The `daily.csv` file should be as follows:

```yaml
repo_name
test-repo
DVWA
deepsemgrep
javaspringvulny
WebGoat
```

Note: first line must be `repo_name`

## Workflow-2: Weekly


The `weekly.csv` file should be as follows:

```yaml
repo_name
test-repo
DVWA
deepsemgrep
javaspringvulny
WebGoat
```

Note: first line must be `repo_name`

You will need to create the following 3 Repository Secrets:

- `SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}`
- `PAT_READ_ONLY_CUSTOMER_REPO: ${{ secrets.PAT_READ_ONLY_CUSTOMER_REPO }}`- Generate PAT with Read-Only access to all repos in your GH ORG
    - NOTE: the scans run inside your GitHub Actions runners and no code leaves your environment (or sent to Semgrep) as part of these scheduled scans
- **OPTIONAL (needed only if you hit issues using `GITHUB_TOKEN`):** `PAT_REPOSITORY_DISPATCH_APPSEC_REPO: ${{ secrets.PAT_REPOSITORY_DISPATCH_APPSEC_REPO }}` - Generate PAT with permissions to initiate repository dispatch in AppSec repo in your AppSec GH ORG

You will need to create the following 3 Repository Variables:

- `APPSEC_REPO_WHERE_SCANS_DONE: ${{ vars.APPSEC_REPO_WHERE_SCANS_DONE }}`- Name of the AppSec Repo where the scans will be performed
- `APPSEC_ORG_WHERE_SCANS_DONE: ${{ vars.APPSEC_ORG_WHERE_SCANS_DONE }}`- Name of the AppSec Org where the scans will be performed. This can be the same as the Org where other repos are
- `WAIT_TIME_BETWEEN_SCANS: ${{ vars.WAIT_TIME_BETWEEN_SCANS }}`- Wait time (in seconds) between triggering Semgrep Scans
- `LOGGING_LEVEL: ${{ vars.LOGGING_LEVEL }}` -  Control logging level

![image](https://github.com/r2c-CSE/semgrep_zero_config_scheduled_scanning/assets/119853191/a0fa30da-55ea-447f-8ac8-d033bb73b49d)

![image](https://github.com/r2c-CSE/semgrep_zero_config_scheduled_scanning/assets/119853191/d9a965a8-0bfc-4b68-a04c-7683855b7bda)



## Workflow**-2**: Creating a workflow file in the scanning repo (this could be in the same GH ORG as customer repos or in a different GH ORG)

```yaml
name: Run Semgrep Scan on Dispatch

on:
  workflow_dispatch: {}
  repository_dispatch:
    types: [zcs-event]

jobs:
  clone-repo:
    runs-on: ubuntu-latest
    container:
      image: returntocorp/semgrep
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2
          
      - name: Clone External Repository
        run: |
          # Replace 'https://' with 'https://{PAT}@' in the GIT_URL
          modified_url=$(echo "${GIT_URL}" | sed "s|https://|https://${PAT}@|")
          
          echo "Original URL: ${GIT_URL}"
          echo "Modified URL: ${modified_url}"
          echo "Repo Name: ${REPOSITORY_NAME}"
          echo "Repo Full Name: ${SEMGREP_REPO_NAME}"
          git clone ${modified_url} cloned-repo          
          # show directory contents of root folder
          ls -l cloned-repo
          git config --global --add safe.directory /__w/nn-zcs/nn-zcs
          cd cloned-repo
          git status
          export SEMGREP_REPO_NAME=$SEMGREP_REPO_NAME
          export SEMGREP_REPO_URL=$GIT_URL
          semgrep ci
        env:
          GIT_URL: ${{ github.event.client_payload.git_url }}
          REPOSITORY_NAME: ${{ github.event.client_payload.repository_name }}
          PAT: ${{ secrets.PAT_READ_ONLY_CUSTOMER_REPO }}
          SEMGREP_REPO_URL: ${{ github.event.client_payload.git_url }}
          SEMGREP_REPO_NAME: ${{ github.event.client_payload.repository_full_name }}
          SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}
```

You will need to create the following 2 Secrets:

- `SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}`
- `PAT: ${{ secrets.PAT_READ_ONLY_CUSTOMER_REPO }}` : PAT should have read-only access to the repos that you would like to scan.

# Setup instructions

1. Create a new repo in your GitHub org - `semgrep-appsec`
2. In this new repo, add the 3 workflow files above in the `.github/workflows/` folder
3. We will now to create some secrets and variables needed for the workflows to work
4. Go to `Settings` >> `Secrets and Variables` >> `Actions`
5. You will need to create the following 3 Repository Variables:
    - `APPSEC_REPO_WHERE_SCANS_DONE: ${{ vars.APPSEC_REPO_WHERE_SCANS_DONE }}`- Name of the AppSec Repo where the scans will be performed
    - `APPSEC_ORG_WHERE_SCANS_DONE: ${{ vars.APPSEC_ORG_WHERE_SCANS_DONE }}`- Name of the AppSec Org where the scans will be performed. This can be the same as the Org where other repos are
    - `WAIT_TIME_BETWEEN_SCANS: ${{ vars.WAIT_TIME_BETWEEN_SCANS }}`- Wait time (in seconds) between triggering Semgrep Scans
6. Create 2 files: `daily.csv` and `weekly.csv` in the root directory of your `semgrep-appsec`- daily.csv will have list of repos that would like to scan daily and weekly.csv will have list of repos that would like to scan weekly
7. You will need to create the following 2 Repository Secrets:
    - `SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}`
    - `PAT_READ_ONLY_CUSTOMER_REPO: ${{ secrets.PAT_READ_ONLY_CUSTOMER_REPO }}`- Generate PAT with Read-Only access to all repos in your GH ORG
        - **NOTE**: the scans run inside your GitHub Actions runners and no code leaves your environment (or sent to Semgrep) as part of these scheduled scans
        - https://github.com/settings/tokens?type=beta
        
        ![image](https://github.com/r2c-CSE/semgrep_zero_config_scheduled_scanning/assets/119853191/ee0b76d9-9b1a-40c9-9e2a-078633c73f7d)

        
    - **OPTIONAL (needed only if you hit issues using `GITHUB_TOKEN`):** `PAT_REPOSITORY_DISPATCH_APPSEC_REPO: ${{ secrets.PAT_REPOSITORY_DISPATCH_APPSEC_REPO }}` - Generate PAT with permissions to initiate repository dispatch in AppSec repo in your AppSec GH ORG:  I should clarify that in order to trigger the workflow, the request must be authenticated with a personal access token for a user that is authorized to access the AppSec repo- `semgrep-appsec`. To create an access token, [follow these steps from GitHub](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line) and be sure that you have given the token access to the repo scope.

![image](https://github.com/r2c-CSE/semgrep_zero_config_scheduled_scanning/assets/119853191/7d3bf9fe-c4cd-422d-ac2d-0d1f3b7df380)

## Semgrep Coverage Report Workflow

<img width="1245" alt="image" src="https://github.com/r2c-CSE/semgrep_zero_config_scheduled_scanning/assets/119853191/d2b28211-f597-4ccd-b617-e7abd370a7cc">
