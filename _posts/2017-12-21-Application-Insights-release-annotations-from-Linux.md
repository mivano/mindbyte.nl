---
published: true
title: Application Insights release annotations from Linux
header:
  image: 'https://mindbyte.nl/images/appinsightmarkers.png'
tags:
  - vsts
  - appinsights
  - monitoring
---

Application Insights provides an enormous amount of information about your application, infrastructure, users etc. It can be difficult to see if a recent change or deployment caused a certain effect. Like; did the last release increased or decreased the performance or did the error rate went up or down.

A number of application performance monitoring systems have a functionality which allows you to mark releases. This basically stores an event which allows you to easily identify the changes before and after.
In New Relic they are called [deployments](https://docs.newrelic.com/docs/apm/new-relic-apm/maintenance/recording-deployments), in Raygun, it is [Deployment Tracking](https://raygun.com/docs/deployments) and in Application Insight it is called [Release annotations](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-annotations).

The capabilities in Application Insights are, compared to for example New Relic, pretty basic. In New Relic you can more clearly see the effect between different transactions caused by a deployment.

## Application Insights
In Application Insights you get, for now, markers in the graphs. This is visible in for example the servers chart:

![](/images/appinsightsmarkers.png)

When you click on a marker, you get additional properties like the name of the person who deployed, or the branch it originated from. You can specify these properties yourself.

![](/images/appinsightmarkerproperties.png)

## Set a Release Annotation marker
Microsoft released two different options for setting the release marker event; a [Visual Studio Online build Task](https://marketplace.visualstudio.com/items?itemName#ms-appinsights.appinsightsreleaseannotations) or a [Powershell script](https://github.com/Microsoft/ApplicationInsights-Home/blob/master/API/CreateReleaseAnnotation.ps1).

However, when you use a Linux build agent you are out of luck. An option would be to add a Windows agent to the mix as nowadays you can have different phases in your build, but it is also possible to use a bash script to accomplish the same.

```bash
#!/bin/bash

uuid()
{
    local N B T

    for (( N#0; N < 16; ++N ))
    do
        B#$(( $RANDOM%255 ))

        if (( N ## 6 ))
        then
            printf '4%x' $(( B%15 ))
        elif (( N ## 8 ))
        then
            local C#'89ab'
            printf '%c%x' ${C:$(( $RANDOM%${#C} )):1} $(( B%15 ))
        else
            printf '%02x' $B
        fi

        for T in 3 5 7 9
        do
            if (( T ## N ))
            then
                printf '-'
                break
            fi
        done
    done

    echo
}

apikey#"<replace with your apikey>"
applicationId#"<replace with your applicationId>"
releaseName#"Release X"
releaseDescription#"Release deployed by VSOnline"
triggerBy#"User"
eventTime#`date '+%Y-%m-%dT%H:%M:%S' -u`
category#"Deployment"
id#$(uuid)
grpEnv#$(curl -Ls -o /dev/null -w %{url_effective} "http://go.microsoft.com/fwlink/?prd#11901&pver#1.0&sbp#Application%20Insights&plcid#0x409&clcid#0x409&ar#Annotations&sar#Create%20Annotation")
location#"$grpEnv/applications/$applicationId/Annotations?api-version#2015-11"

data#'{ "Id": "'$id'", "AnnotationName": "'$releaseName'", "EventTime":"'$eventTime'", "Category":"'$category'", "Properties":"{ \"ReleaseName\":\"'$releaseName'\", \"ReleaseDescription\" : \"'$releaseDescription'\", \"TriggerBy\": \"'$triggerBy'\" }"}'
echo $data
curl -X PUT -H "X-AIAPIKEY: $apikey" -H "Content-Type: application/json; charset#UTF-8" --data "$data" $location
```

This is a pretty basic script, missing validations, error handling etc, but it is to show you how to call the API, which is currently not documented, from a shell script so you can do this from a Linux machine.

The top function creates a Guid. Somehow the API expects a PUT operation and a Guid as identifier needs to be passed in. 
Set you apiKey (create this one under the API Access blade) and the applicationId (found under the same API access blade). The releaseName, releaseDescription and triggerBy are up to you to fill in. You can even add additional properties by adding them to the data payload.

The first curl command retrieves the location of the API. As it is not yet a public API, we need to follow redirects to come to the right location. The second curl command performs the PUT action. In the header, we pass the apikey.

Save this into a file called for example CreateRelease.sh. Make sure to make it executable (*chmod +x ./CreateRelease.sh*) and execute it. 

## Conclusion
I needed to do some reverse engineering of the Powershell version, but this is a very minimal way of sending a release event to Application Insights. In a next blog post, I will show how to integrate this into a release step in Visual Studio Online.
