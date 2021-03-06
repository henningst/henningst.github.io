---
layout:     post
title:      Publish xUnit test results in VSTS using Cake
date:       2017-04-26 21:50:00
summary:    This post shows how you can get test results from xUnit shown in VSTS using Cake
categories: aspnet-core azure
---

I've been moving the build process for a few .NET Core projects from Team City to Visual Studio Team Services (VSTS) the last couple of days. I'm using xUnit for testing and have had some issues with getting the test results to show up in VSTS. This blog post shows how to get it up and running.

## Build script

I'm using [Cake](http://cakebuild.net/) to script the build process. This allows me to define all the build steps in a build file I can keep in the git repo together with the rest of the project. It also keeps the number of build tasks I need to define in VSTS at a minimum. It's easy to trigger Cake scripts from VSTS by using the [Cake tool](https://marketplace.visualstudio.com/items?itemName=cake-build.cake) available in the VSTS Marketplace.


VSTS does not automatically pick up results from tests run via Cake, so the first we need to do is to make sure the test runner is writing test results to an output file.

In the Cake script, this is done by passing in a `DotNetCoreTestSettings` object and specifying some additional arguments using `ArgumentCustomization` like this:

```
Task("Test")
    .IsDependentOn("Restore")
    .Does(() =>
    {
        var settings = new DotNetCoreTestSettings
        {
            // Outputing test results as XML so that VSTS can pick it up
            ArgumentCustomization = args => args.Append("--logger \"trx;LogFileName=TestResults.xml\"")
        };

        DotNetCoreTest("test/Project.csproj", settings);
    });
```

When running the `Test` task using Cake, I'm now getting a `TestResults.xml` file that can be used to display a nice test report in VSTS.


## VSTS Build definition

In VSTS my build process consists of mainly 2 tasks. The first task is executing the Cake script and makes sure all the projects compile and all the tests are run. After adding the extra argument as shown above, the main build step is also making sure the test results are written to `TestResults.xml` which we can then pick up in a subsequent step.

![Cake build settings]({{ site.url }}/assets/img/posts/vsts-testresults/01.png)

In order to actually show the test report, we need to add another task to the build definition in VSTS. This task is called `Publish Test Results`. This task is configured to look for all files matching the pattern `**/TestResults*.xml`. 

One important detail here is that you have to choose the `VSTest` test result format even if your tests are actually xUnit tests.

Also remember to choose "Run this task - Even if a previous task has failed, unless the build was canceled" under `Control Options` to 
make sure the test results are published even if there are failing tests or any of the preceeding steps are failing (Thanks
to [@josteinkl](https://twitter.com/josteikl) for pointing this out!).

![Copy test results]({{ site.url }}/assets/img/posts/vsts-testresults/02.png)


When you run a new build, you should now see a nice test report as shown in the image below.

![Test report]({{ site.url }}/assets/img/posts/vsts-testresults/03.png)

