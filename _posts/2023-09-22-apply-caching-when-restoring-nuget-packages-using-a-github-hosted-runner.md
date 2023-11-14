---
published: 2023-09-21T22:33:06.170Z
title: Apply caching when restoring NuGet packages using a GitHub hosted runner
tags:
  - GitHub Actions
  - GitHub
header:
  teaser: "https://mindbyte.nl/images/8-VWpFELw5qujytz6.png"
slug: apply-caching-restoring-nuget-packages-github-hosted-runner
---

When you are using a GitHub hosted runner to build your project, you can apply caching to speed up the build process. This is especially useful when you are using NuGet packages. 
As you get a new runner every time you run a build, you will need to restore all NuGet packages every time. This can take a lot of time as it is not the most efficient network calls. 

To speed up the process, you can cache the NuGet packages. This will make sure that the next time you run a build, the NuGet packages will be restored from the cache instead of the NuGet website. 
To make sure that the cache is invalidated when a new version of a NuGet package is referenced, you can use the hash of the project files as part of the cache key. This will make sure that the cache is invalidated when changes are made to the project files.

For completness sake, I have included a full example of a workflow file that uses caching to speed up the build process. GitHub will automatically add a post step to store the files in the cache.

By setting the `NUGET_PACKAGES` environment variable, you can make sure that the NuGet packages are restored to the same location as the cache. This will make sure that the cache is used when restoring the NuGet packages.

{% raw %}
```yaml
name: Build Services
on:
  push:

jobs:
  build:
    runs-on: windows-latest
    name: Build services
    permissions:
      packages: write
      contents: read
    env:
      NUGET_PACKAGES: ${{ github.workspace }}/.nuget/packages    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: 7.0.x

      - uses: actions/cache@v3
        with:
          path: ${{ github.workspace }}\.nuget\packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }} #hash of project files
          restore-keys: |
            ${{ runner.os }}-nuget-

      - name: Restore dependencies
        run:  dotnet restore solution.sln 

      - name: Build
        run: dotnet build solution.sln  --no-restore
        
      - name: Test
        run: dotnet test solution.sln  --no-build --verbosity normal
      
      - name: Publish
        run: | 
          dotnet publish solution.sln -c Release --no-restore  -o publish 

      - name: Upload a Build Artifact
        uses: actions/upload-artifact@v3
        with:
          name: services
          path: publish/**
          if-no-files-found: error 
```
{% endraw %}

This workflow will restore, build, test and package the solution. The resulting package will be uploaded as an artifact and can be deployed in subsequent steps.

Do use the logs to verify that the cache is used and validate if the amount of time that is used is worthwhile. 