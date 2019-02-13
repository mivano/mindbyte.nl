---
published: true
mathjax: false
featured: false
comments: false
title: Analyze your project with SonarQube using a docker container
tags: 'dotnet, sonarqube'
imagefeature: /images/sonarqube.png
---
SonarQube is an excellent static code analyzer tool as it has many different analyzers and provides useful suggestions for any potential bugs and issues. As such, it is very beneficial to have an instance of SonarQube running somewhere and process your code when you do a commit to a branch. My colleague Rob Bos has some pointers on how to set this up on his [blog](https://rajbos.github.io/blog/2018/10/20/SonarQube-setup).

But what if you just want to run a quick SonarQube analysis over a project to get its current state? Installing SonarQube can be cumbersome in that case. So I created a small Powershell script that runs a local SonarQube docker container and performs the analysis over it. As Java is a requirement for the SonarQube tools used for scanning, you do need to have those installed once for this to work. With some tinkering, you might even be able to include them inside the script also, but as I already have Java installed and configured, this was not an issue for me.

```powershell
if (-NOT (Test-Path 'env:JAVA_HOME')) { 
   Write-Error "The environment variable called JAVA_HOME does not exist. Make sure the JAVA SDK is installed and the environment variable has been set."
   exit
}

Write-Host "Starting SonarQube"
docker run -d --name sonarqube -p 9000:9000 -p 9092:9092 sonarqube

Write-Host "Make sure the dotnet sonarscanner tooling is installed"
dotnet tool install --global dotnet-sonarscanner --version 4.3.1

Write-Host "Run the tests"
dotnet test ..\..\tst\yourtestprojectfile.csproj /p:CollectCoverage=true /p:CoverletOutputFormat=opencover

Write-Host "Stop the build server"
dotnet build-server shutdown

Write-Host "Start the scanner"
dotnet sonarscanner begin /k:"yourprojectname" /d:sonar.host.url=http://localhost:9000 /d:sonar.cs.opencover.reportsPaths="..\..\tst\coverage.opencover.xml" /d:sonar.coverage.exclusions="**Tests*.cs"

Write-Host "Build the solution"
dotnet build ..\..\yoursolution.sln

Write-Host "Stop the scanner"
dotnet sonarscanner end

Write-Host "Open the page in the browser"
start http://localhost:9000/dashboard/index/yourprojectname
```

First, we check that we indeed have the JAVA_HOME variable defined. If not, we stop already.

Next step is the actual SonarQube container. The `docker run` command will fetch the image and starts it with two configured ports.

Then we need the actual scanner. Using `dotnet tool` we install it globally if it not yet installed. 

Now we are ready; start by running the tests in order to have code coverage included in SonarQube. In this case, we have [coverlet](https://github.com/tonerdo/coverlet) configured to output code coverage files in the opencover format. The scanner will start, and we specify a unique project name, the host with port (which is the just started docker container) and the already generated code coverage files.

Actual building and stopping the scanner will do the rest. The scanner will collect all the needed details and submit those to the docker container running SonarQube.

The last step is to open your default browser to look at the results. It might still be busy processing the results, so you need to refresh in that case.

![sonarqube.png](/images/sonarqube.png)

Keep in mind that data will not be saved between restarts of the container. It is really for brief analysis of a project. However, it is an effortless way to add a small piece of Powershell and start analyzing any project.
