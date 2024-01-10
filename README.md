# How to run Semgrep Scheduled scans on all repos in a given GitHub Org: Option-2 using Repository_Dispatch (Final)

# Why?

Customers want to onboard Semgrep quickly without having to go to each team one by one and ask them to add Semgrep to their repo’s workflow files.

They can do this for PR’s using [Repository rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets). 

We can now also accomplish scheduled full scans using the solution below

The solution below can be used in conjunction with the Repository Rulesets to get additional coverage.

# How?

We create a new repo - `appsec-repo` (for example) that is owned by the App Sec team. In this repo we create a workflow file that is triggered on `Workflow_Dispatch` and `Scheduled` trigger

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

```yaml
name: DAILY- Semgrep Scheduled Scan on Repos

on:
  # Scan on-demand through GitHub Actions interface:
  workflow_dispatch: {}
  # Schedule the CI job (this method uses cron syntax):
  schedule:
    - cron: '20 17 * * *' # Sets Semgrep to scan every day at 17:20 UTC.

jobs:
  get-list-of-repos-and-perform-semgrep-scan:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout
      uses: actions/checkout@v3

    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.9'

    - name: Install dependencies
      run: |
        pip install requests
        pip install pandas
        pip install logging
        
    - name: Get list of repositories
      env:
        # Connect to Semgrep Cloud Platform through your SEMGREP_APP_TOKEN.
        # Generate a token from Semgrep Cloud Platform > Settings
        # and add it to your GitHub secrets.
        SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}

        # Generate PAT with Read-Only access to all repos in your GH ORG
        PAT: ${{ secrets.PAT_READ_ONLY_CUSTOMER_REPO }}

        # Generate PAT with permissions to initiate repository dispatch in AppSec repo in your AppSec GH ORG
        PAT_REPOSITORY_DISPATCH_APPSEC_REPO: ${{ secrets.GITHUB_TOKEN }}  

        # Name of the AppSec Repo where the scans will be performed
        APPSEC_REPO_WHERE_SCANS_DONE: ${{ vars.APPSEC_REPO_WHERE_SCANS_DONE }}

        # Name of the AppSec Org where the scans will be performed. This can be the same as the Org where other repos are
        APPSEC_ORG_WHERE_SCANS_DONE: ${{ vars.APPSEC_ORG_WHERE_SCANS_DONE }}

        # Wait time (in seconds) between triggering Semgrep Scans 
        WAIT_TIME_BETWEEN_SCANS: ${{ vars.WAIT_TIME_BETWEEN_SCANS }}

        # logging level
        LOGGING_LEVEL: ${{ vars.LOGGING_LEVEL }}

      run: |
        import requests
        import os
        import requests
        import logging
        import time
        import pandas as pd
        import logging
      
        # Set up logging
        # Get logging level from environment variable
        log_level = os.getenv('LOGGING_LEVEL', 'INFO').upper()
        
        # Configure logging
        logging.basicConfig(level=log_level,
                            format='%(asctime)s - %(name)s - %(levelname)s - %(message)s')
        logging.info("Logging Level: %s", log_level)

        # GitHub repository in the format "org_name/repo_name"
        full_repo_name = os.environ['GITHUB_REPOSITORY']
        # Extract the organization name
        org_name = full_repo_name.split('/')[0]
        logging.info("Organization Name: %s", org_name)
        pat = os.environ['PAT']
        headers = {'Authorization': f'token {pat}'}

        repos = []
        page = 1
        while True:
            url = f'https://api.github.com/orgs/{org_name}/repos?page={page}&per_page=100'
            response = requests.get(url, headers=headers)
            logging.debug("Requesting list of repos in Org: %s, page: %s" ,org_name, page)
            logging.debug("Response Code: %s", response.status_code)
            page_repos = response.json()
            if not page_repos:
                break
            repos.extend(page_repos)
            page += 1
        
        # response = requests.get(f'https://api.github.com/orgs/{org_name}/repos', headers=headers)
        # repos = response.json()

        # Replace 'your_file.csv' with the path to your CSV file
        csv_file = 'daily.csv'
        df = pd.read_csv(csv_file)

        # Extract the list of repo names from the CSV file
        # Replace 'repo_name' with the actual column name in your CSV
        interested_repos = df['repo_name'].tolist()
        
        # Filter the repos list
        filtered_repos = [repo for repo in repos if repo['name'] in interested_repos]

        filtered_repos_names = [repo['name'] for repo in filtered_repos]
        logging.debug("list of filtered repos: %s", filtered_repos_names)

        logging.debug("total number of repos: %s" ,len(repos))
        logging.debug("number of filtered repos: %s" ,len(filtered_repos))

        # GitHub API URL for repository_dispatch
        scanning_repo = os.environ['APPSEC_REPO_WHERE_SCANS_DONE']
        logging.info("APPSEC_REPO_WHERE_SCANS_DONE: %s", scanning_repo)
        
        scanning_org_name = os.environ['APPSEC_ORG_WHERE_SCANS_DONE']
        logging.info("APPSEC_ORG_WHERE_SCANS_DONE: %s", scanning_org_name)

        repo_dispatch_url = f'https://api.github.com/repos/{scanning_org_name}/{scanning_repo}/dispatches'
        logging.debug("Repo Dispatch URL: %s", repo_dispatch_url)
          
        # Headers for the request
        # token below allows to make Repository Dispatch calls to the GH repo/ GH Org where the scans will be done
        token = os.environ['PAT_REPOSITORY_DISPATCH_APPSEC_REPO']
        headers = {
            'Authorization': f'Bearer {token}', 
            'Accept': 'application/vnd.github.v3+json'
        }
        
        
        for repo in filtered_repos:			
          repo_name = repo['name']
          # Check if the value exists in any cell of the DataFrame
          # if df.isin([repo_name]).any().any():
          logging.info('The value "%s" exists in the daily scan list', repo_name)              
          repo_full_name = repo['full_name']
          repo_branch = repo['default_branch']

          repo_url = repo['clone_url']
          repo_clone_url = repo['clone_url']
          logging.info("Repo Name: %s, Repo URL: %s", repo_name, repo_url)
                        
          # Payload for the repository_dispatch event
          # Change 'event_type' and 'client_payload' as per your requirements			    
          payload = {
              'event_type': 'zcs-event',
              'client_payload': { "repository_name": repo_name, "repository_full_name": repo_full_name, "git_url": repo_clone_url }
          }
      
          # Make the POST request
          response = requests.post(repo_dispatch_url, headers=headers, json=payload)
          logging.debug('Making Repository dispatch call for %s to %s / %s : %s', repo_name, scanning_org_name, scanning_repo, repo_dispatch_url)

          # Log the response from GitHub
          logging.debug("Response Code: %s", response.status_code)

          wait_time = os.environ['WAIT_TIME_BETWEEN_SCANS']
          logging.debug('wait time is: %s', wait_time)
          time.sleep(int(wait_time)) #sleep for XX seconds before starting next scan
          # else:
          #   logging.info('The value "%s" does not exist in the daily scan list', repo_name)
          
      shell: python
```

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

```yaml
name: WEEKLY- Semgrep Scheduled Scan on Repos

on:
  # Scan on-demand through GitHub Actions interface:
  workflow_dispatch: {}
  # Schedule the CI job (this method uses cron syntax):
  schedule:
    - cron: '10 02 * * 6' # Sets Semgrep to scan every Saturday at 02:10 UTC.

jobs:
  get-list-of-repos-and-perform-semgrep-scan:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout
      uses: actions/checkout@v3

    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.9'

    - name: Install dependencies
      run: |
        pip install requests
        pip install pandas
        pip install logging
        
    - name: Get list of repositories
      env:
        # Connect to Semgrep Cloud Platform through your SEMGREP_APP_TOKEN.
        # Generate a token from Semgrep Cloud Platform > Settings
        # and add it to your GitHub secrets.
        SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}

        # Generate PAT with Read-Only access to all repos in your GH ORG
        PAT: ${{ secrets.PAT_READ_ONLY_CUSTOMER_REPO }}

        # Generate PAT with permissions to initiate repository dispatch in AppSec repo in your AppSec GH ORG
        PAT_REPOSITORY_DISPATCH_APPSEC_REPO: ${{ secrets.GITHUB_TOKEN }}  #Bearer ${{ secrets.GITHUB_TOKEN }}

        # Name of the AppSec Repo where the scans will be performed
        APPSEC_REPO_WHERE_SCANS_DONE: ${{ vars.APPSEC_REPO_WHERE_SCANS_DONE }}

        # Name of the AppSec Org where the scans will be performed. This can be the same as the Org where other repos are
        APPSEC_ORG_WHERE_SCANS_DONE: ${{ vars.APPSEC_ORG_WHERE_SCANS_DONE }}

        # Wait time (in seconds) between triggering Semgrep Scans 
        WAIT_TIME_BETWEEN_SCANS: ${{ vars.WAIT_TIME_BETWEEN_SCANS }}

        # logging level
        LOGGING_LEVEL: ${{ vars.LOGGING_LEVEL }}

      run: |
        import requests
        import os
        import requests
        import logging
        import time
        import pandas as pd
        import logging
      
        # Set up logging
        # Get logging level from environment variable
        log_level = os.getenv('LOGGING_LEVEL', 'INFO').upper()
        
        # Configure logging
        logging.basicConfig(level=log_level,
                            format='%(asctime)s - %(name)s - %(levelname)s - %(message)s')
        logging.info("Logging Level: %s", log_level)

        # GitHub repository in the format "org_name/repo_name"
        full_repo_name = os.environ['GITHUB_REPOSITORY']
        # Extract the organization name
        org_name = full_repo_name.split('/')[0]
        logging.info("Organization Name: %s", org_name)
        pat = os.environ['PAT']
        headers = {'Authorization': f'token {pat}'}

        repos = []
        page = 1
        while True:
            url = f'https://api.github.com/orgs/{org_name}/repos?page={page}&per_page=100'
            response = requests.get(url, headers=headers)
            logging.debug("Requesting list of repos in Org: %s, page: %s" ,org_name, page)
            logging.debug("Response Code: %s", response.status_code)
            page_repos = response.json()
            if not page_repos:
                break
            repos.extend(page_repos)
            page += 1
        
        # response = requests.get(f'https://api.github.com/orgs/{org_name}/repos', headers=headers)
        # repos = response.json()

        # Replace 'your_file.csv' with the path to your CSV file
        csv_file = 'weekly.csv'
        df = pd.read_csv(csv_file)

        # Extract the list of repo names from the CSV file
        # Replace 'repo_name' with the actual column name in your CSV
        interested_repos = df['repo_name'].tolist()
        
        # Filter the repos list
        filtered_repos = [repo for repo in repos if repo['name'] in interested_repos]

        filtered_repos_names = [repo['name'] for repo in filtered_repos]
        logging.debug("list of filtered repos: %s", filtered_repos_names)

        logging.debug("total number of repos: %s" ,len(repos))
        logging.debug("number of filtered repos: %s" ,len(filtered_repos))

        # GitHub API URL for repository_dispatch
        scanning_repo = os.environ['APPSEC_REPO_WHERE_SCANS_DONE']
        logging.info("APPSEC_REPO_WHERE_SCANS_DONE: %s", scanning_repo)
        
        scanning_org_name = os.environ['APPSEC_ORG_WHERE_SCANS_DONE']
        logging.info("APPSEC_ORG_WHERE_SCANS_DONE: %s", scanning_org_name)

        repo_dispatch_url = f'https://api.github.com/repos/{scanning_org_name}/{scanning_repo}/dispatches'
        logging.debug("Repo Dispatch URL: %s", repo_dispatch_url)
          
        # Headers for the request
        # token below allows to make Repository Dispatch calls to the GH repo/ GH Org where the scans will be done
        token = os.environ['PAT_REPOSITORY_DISPATCH_APPSEC_REPO']
        headers = {
            'Authorization': f'Bearer {token}', 
            'Accept': 'application/vnd.github.v3+json'
        }
        
        
        for repo in filtered_repos:			
          repo_name = repo['name']
          logging.info('The value "%s" exists in the weekly scan list', repo_name)              
          repo_full_name = repo['full_name']
          repo_branch = repo['default_branch']

          repo_url = repo['clone_url']
          repo_clone_url = repo['clone_url']
          logging.info("Repo Name: %s, Repo URL: %s", repo_name, repo_url)
                        
          # Payload for the repository_dispatch event
          # Change 'event_type' and 'client_payload' as per your requirements			    
          payload = {
              'event_type': 'zcs-event',
              'client_payload': { "repository_name": repo_name, "repository_full_name": repo_full_name, "git_url": repo_clone_url }
          }
      
          # Make the POST request
          response = requests.post(repo_dispatch_url, headers=headers, json=payload)
          logging.debug('Making Repository dispatch call for %s to %s / %s : %s', repo_name, scanning_org_name, scanning_repo, repo_dispatch_url)

          # Log the response from GitHub
          logging.debug("Response Code: %s", response.status_code)

          wait_time = os.environ['WAIT_TIME_BETWEEN_SCANS']
          logging.debug('wait time is: %s', wait_time)
          time.sleep(int(wait_time)) #sleep for XX seconds before starting next scan
          
      shell: python
```

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

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/aeaf09c9-e827-4fed-b6a2-cc0fcc31bc3c/1fc21d0f-a904-45d8-b4b7-882c8298a02d/Untitled.png)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/aeaf09c9-e827-4fed-b6a2-cc0fcc31bc3c/e208f82a-ce0f-4ba0-9835-7cce167d0ffe/Untitled.png)

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
        - 
        
        ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/aeaf09c9-e827-4fed-b6a2-cc0fcc31bc3c/8123ab8e-cbb5-4070-9628-5c569dee9cd8/Untitled.png)
        
    - **OPTIONAL (needed only if you hit issues using `GITHUB_TOKEN`):** `PAT_REPOSITORY_DISPATCH_APPSEC_REPO: ${{ secrets.PAT_REPOSITORY_DISPATCH_APPSEC_REPO }}` - Generate PAT with permissions to initiate repository dispatch in AppSec repo in your AppSec GH ORG:  I should clarify that in order to trigger the workflow, the request must be authenticated with a personal access token for a user that is authorized to access the AppSec repo- `semgrep-appsec`. To create an access token, [follow these steps from GitHub](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line) and be sure that you have given the token access to the repo scope.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/aeaf09c9-e827-4fed-b6a2-cc0fcc31bc3c/4fcc6588-49fb-4115-a405-4fc3871af9e1/Untitled.png)

## Semgrep Coverage Report Workflow
