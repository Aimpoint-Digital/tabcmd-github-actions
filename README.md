# Github Actions for deploying Tableau workbooks to Tableau Server/Online

This repository is an accompaniment to this blog post. These scripts enable you to deploy Tableau workbooks directly from Github to Tableau Server/Online using Github Actions. 
Publishing is enabled using the tabcmd command-line interface for Tableau. 

## Secrets

The following values are stored in Github secrets manager to avoid being exposed in the scripts:
- TABLEAU_HOST (Tableau Server/Online host address)
- TABCMD_URL (URL to the appropriate version of tabcmd for your Tableau Server/Online instance)
- TABLEAU_PASSWORD
- TABLEAU_USERNAME


## Example 1 - Specific workbooks

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

Lastly we perform operations in the workflow with the <b>run</b> key. In this script we download tabcmd, accept the user license agreement, login to Tableau Server and publish the workbook.

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



## Example 2 - Added/modified workbooks

This example publishes any Tableau workbooks to the Tableau Server that have been added or modified in the latest push to the main branch. Additionally, the workbooks are not in the root directory but in a folder "Workbooks".

#### Walkthrough
The difference in the first chunk is that we are specifying that the Action should only be triggered by pushes to the main branch if .twbx files in the _Workbooks/_ path are concerned. Changes to .twbx files outside this directory will not trigger the workflow.

```yaml
name: deploy-added-modified-workbooks
on:
  push:
    branches:
      - main
    paths:
      - 'Workbooks/**.twbx'
```

This example has an extra dependency to identify files that have been added or modified - [jitterbit/get-changed-files@v1](https://github.com/jitterbit/get-changed-files). We specify that the full paths to added/modified files will be stored in a csv file, with an alias of "files".

```yaml
jobs:
  publish-to-tableau-server:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2.1.0
      - id: files
        uses: jitterbit/get-changed-files@v1
        with:
          format: 'csv'

```


The following section is the same as Example 1, where we download and login to Tableau Server with tabcmd.

```yaml
      # Download and unpackage tabcmd
      - run: sudo wget -O tableau-tabcmd.deb "${{ secrets.TABCMD_URL }}"
      - run: sudo dpkg-deb -x ./tableau-tabcmd.deb ./tabcmdpackage/

      # Accept Tableau EULA
      - run: ./tabcmdpackage/opt/tableau/tabcmd/bin/tabcmd --accepteula

      # Log in to Tableau Server
      - run: ./tabcmdpackage/opt/tableau/tabcmd/bin/tabcmd login -s "${{ secrets.TABLEAU_HOST }}" -u "${{ secrets.TABLEAU_USERNAME }}" -p "${{ secrets.TABLEAU_PASSWORD }}"

```

Lastly, we loop through the filepaths to the added/modified files that we stored in the csv (referenced by steps.files.output).

```yaml

      # Handle added/modified files
      - run: |
          if [ ! -z "${{ steps.files.outputs.added_modified }}" ]
          then
          mapfile -d ',' -t added_modified_files < <(printf '%s,' '${{ steps.files.outputs.added_modified }}')
          for added_modified_file in "${added_modified_files[@]}"; do
            workbook_name=$(echo "$added_modified_file" | rev | cut -d "/" -f 1 | rev)

            ./tabcmdpackage/opt/tableau/tabcmd/bin/tabcmd publish "${added_modified_file}" -n "$workbook_name" --project "My Project" --overwrite
          done
          fi


```


Drilling into this, the first line is just a catch that only runs the code if the csv is not empty (the jitterbit/get-changed-files@v1 action also collects information about renamed and deleted files, which could trigger the action, rendering the csv for added/modified files empty):

          if [ ! -z "${{ steps.files.outputs.added_modified }}" ]
          then
```

Next we map the contents of the csv to a variable "added_modified_files".
```yaml
          mapfile -d ',' -t added_modified_files < <(printf '%s,' '${{ steps.files.outputs.added_modified }}')
```

Lastly we loop through each value in added_modified_files. We wrangle the workbook name from the file path (assuming the workbook name will be the filename without the path and .twbx extension). Then we publish the workbook to the Tableau Server using the workbook name we just wrangled.

```yaml
          for added_modified_file in "${added_modified_files[@]}"; do
            workbook_name=$(echo "$added_modified_file" | rev | cut -d "/" -f 1 | rev)

            ./tabcmdpackage/opt/tableau/tabcmd/bin/tabcmd publish "${added_modified_file}" -n "$workbook_name" --project "My Project" --overwrite
          done
```
