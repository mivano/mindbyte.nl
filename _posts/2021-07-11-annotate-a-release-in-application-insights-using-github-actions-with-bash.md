---
published: true
title: Annotate a release in Application Insights using GitHub Actions with Bash
tags:
  - GitHub
  - Azure
  - appinsights
  - monitoring
---

In 2017, I already [blogged](https://mindbyte.nl/2017/12/21/Application-Insights-release-annotations-from-Linux.html) about sending a release annotation to Application Insights. This graphical indication gives an excellent overview of when new releases of your application are deployed so you can see the before and after effects as well.

Microsoft has updated their support for [sending release annotations](https://docs.microsoft.com/en-us/azure/azure-monitor/app/annotations). If you use Azure Pipelines, it will automatically set the release in Application Insights. You can also use the Powershell option. Although there is a GitHub Action in the [marketplace](https://github.com/marketplace/actions/application-insights-annotations), it has the same limitation; you need an API key. Something to create, store secretly, and fetch to use in your pipeline.

I wanted to use something more straightforward and run on a Linux machine, so I altered my previous script a bit to become easier to use.

```bash
echo "Creating release annotation in Application Insights"

APPINSIGHTS_ID=$(az resource show -g $1-rg -n $1-applicationinsights --resource-type "microsoft.insights/components" --query id -o tsv)
UUID=$(cat /proc/sys/kernel/random/uuid)
releaseName=$2
releaseDescription=$3
triggerBy=$4
eventTime=`date '+%Y-%m-%dT%H:%M:%S' -u`
category="Deployment"

data='{ "Id": "'$UUID'", "AnnotationName": "'$releaseName'", "EventTime":"'$eventTime'", "Category":"'$category'", "Properties":"{ \"ReleaseName\":\"'$releaseName'\", \"ReleaseDescription\" : \"'$releaseDescription'\", \"TriggerBy\": \"'$triggerBy'\" }"}'

az rest --method put --uri "$APPINSIGHTS_ID/Annotations?api-version=2015-05-01" --body "$data" -o none
```

This script is saved as `create-release-annotation.sh` and uses the AZ CLI to fetch the Application Insights ID (adjust the resource group and name property to your liking). It uses the `az rest` command to run with enough permissions to create a release annotation.

## GitHub Action

Inside my CICD pipeline, I have a step that deploys my application. When done, I run this script and pass in a couple of parameters. 

```yaml
     - name: "Deploy Azure Functions"
        run: |
          # Do the actual deployment

          branch=${GITHUB_HEAD_REF##*/}
          ${{ "{{" }} env.OUTPUT_PATH }}/scripts/create-release-annotation.sh tst "Release for $branch" "Release for $branch and SHA ${GITHUB_SHA}" "${GITHUB_ACTOR}"

```

The first parameter is the environment; the second one is the name, the third one the description, and the last one indicates the user who triggered the release. As this is part of the CICD of a pull request/feature branch, I extract the name of the branch and use that in the name of the release.

When releasing a deployment, I instead show the version (the tag):

```yaml
    - name: "Deploy Azure Functions"
        run: |
          # Do the actual deployment

          tag=${GITHUB_REF/refs\/tags\//}
          ${{ "{{" }} env.OUTPUT_PATH }}/scripts/create-release-annotation.sh prd "$tag" "Release of $tag (${GITHUB_SHA})" "${GITHUB_ACTOR}"
```

Inside Application Insights, I get deployment information like shown below.

![](/images/2021-07-11-21-47-51.png)