---
published: true
title: Use job summaries in GitHub Actions to generate a list of releases
tags:
  - GitHub  
---

GitHub has a new nice feature to create [job summaries](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#adding-a-job-summary) for your actions. It allows you to generate markdown from your workflow that is rendered in the GitHub UI as an output of the run.

Internally it works by writing text to a file which is picked up when the workflow completes. The file's name can be found in the `GITHUB_STEP_SUMMARY` environment variable and is unique for each step in a job. 

So to add markdown, you can do something like this:

```bash
echo "### Hello world! :rocket:" >> $GITHUB_STEP_SUMMARY
```

I wanted to generate an overview of the last x releases and output them as part of a workflow. Automating this helps to be compliant and to avoid manual work by collecting this data. So GitHub actions on a schedule and some bash scripting to generate the markdown.

```yaml
name: List releases

on:
  workflow_dispatch:
  schedule:
    - cron:  '0 13 1 */1 *'  # run every month on the first at 13:00

jobs:
  create-list:
    name: Create list of releases
    runs-on: ubuntu-latest   
    steps:      
      - name: Retrieve releases and generate report
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          releases=$(gh release list -L 40 --exclude-drafts)
          
          echo "Overview of last 40 releases" >> $GITHUB_STEP_SUMMARY
              
          while IFS= read -r line; do
              id=$(echo "$line" | cut -d$'\t' -f3)
              release=$(gh release view $id --json body,name,publishedAt,createdAt,url)

              echo "" >> $GITHUB_STEP_SUMMARY # this is a blank line
              name=$(echo $release | jq -r '.name')
              published=$(echo $release | jq -r '.publishedAt')
              created=$(echo $release | jq -r '.createdAt')
              body=$(echo $release | jq -r '.body')
              url=$(echo $release | jq -r '.url')

              echo "# Release: $name"  >> $GITHUB_STEP_SUMMARY
              echo "Published at $published and created at $created" >> $GITHUB_STEP_SUMMARY    
              echo $body  >> $GITHUB_STEP_SUMMARY
              echo "" >> $GITHUB_STEP_SUMMARY # this is a blank line
              echo "Release URL: $url" >> $GITHUB_STEP_SUMMARY

          done <<< "$releases"
    
```

So first of all, the triggers. A schedule to run every month on the first at 13:00 and a workflow dispatch to run it on demand.

We need an agent to run it on; ubuntu latest will suffice. The rest is a bash script using the GH CLI. To authenticate, we need to pass in the GH_TOKEN as an environment variable.

Fetch the last 40 releases with `gh release list -L 40 --exclude-drafts` and then loop over them.

We need to get the identifier of the release and then use `gh release view` to get the details. To split on a tab and select the correct field, we use `cut -d$'\t' -f3`.

Retrieve the contents using `gh release view $id --json body,name,publishedAt,createdAt,url` which will return the specified fields as json. Using the already installed `jq`, we can parse the json and extract the fields we want.

We then output the markdown to the step summary file. When we run this workflow, the step summary file will be available and contain the markdown. 

Adjust to your needs and see the power of outputting markdown in GitHub Actions.
