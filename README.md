# Github Actions for deploying Tableau workbooks to Tableau Server/Online

This repository is an accompaniment to the blog post <b>Tableau development: Source Control and continuous deployment with git and Github</b>. These scripts enable you to deploy Tableau workbooks directly from Github to Tableau Server/Online using Github Actions. Publishing is enabled using the tabcmd command-line interface for Tableau. 

## Secrets

The following values are stored in Github secrets manager to avoid being exposed in the scripts:
- TABLEAU_HOST (Tableau Server/Online host address)
- TABCMD_URL (URL to the appropriate version of tabcmd for your Tableau Server/Online instance)
- TABLEAU_PASSWORD
- TABLEAU_USERNAME


## Example 1 - Specific workbook

The first script <b>tabcmd-action-example1</b> enables the deployment of a specific Tableau workbook <b>my-tableau-workbook.twbx</b> to a Tableau Server/Online instance. Note that the data source is packaged with the workbook (.twbx). To publish .twb files with tabcmd, the data source must be either a live database connection or a published Tableau Data Source. This script assumes the workbook is at the root level of the repository.

#### Walkthrough
The action is named <b>deploy-tableau-workbook</b> and will appear as such in the Github Actions console and log. The <b>on</b>, <b>push</b> and <b>branches</b> keys determine that the Action should be triggered on a push event to the branch titled "main". When any changes are pushed to the "main" branch, this Action will trigger.

```yaml
name: deploy-tableau-workbook
on:
  push:
    branches:
      - main
```

Next the jobs and dependencies are defined. This script has one job named <b>publish-to-tableau-server</b>. The first dependency is the virtual machine (<b>runs-on</b>), for which we have selected the latest version of the ubuntu machine. Secondly we are dependent on the native Github Action [Checkout](https://github.com/actions/checkout), which allows the Action workflow to access the repository.

```yaml
jobs:
  publish-to-tableau-server:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2.1.0

```

Lastly we perform operations in the workflow with the <b>run</b> key. In this script we download tabcmd, accept the user license agreement, and publish the workbook to Tableau Server.

```yaml
      # Download and unpackage tabcmd
      - run: sudo wget -O tableau-tabcmd.deb "${{ secrets.TABCMD_URL }}"
      - run: sudo dpkg-deb -x ./tableau-tabcmd.deb ./tabcmdpackage/

      # Accept Tableau EULA for tabcmd
      - run: ./tabcmdpackage/opt/tableau/tabcmd/bin/tabcmd --accepteula

      # Log in to Tableau Server with tabcmd
      - run: ./tabcmdpackage/opt/tableau/tabcmd/bin/tabcmd login -s "${{ secrets.TABLEAU_HOST }}" -u "${{ secrets.TABLEAU_USERNAME }}" -p "${{ secrets.TABLEAU_PASSWORD }}"

      # Handle added/modified files
      - run: |
            ./tabcmdpackage/opt/tableau/tabcmd/bin/tabcmd publish "my-tableau-workbook.twbx" -n "My Tableau Workbook" --project "Github Actions Testing" --overwrite         
```

