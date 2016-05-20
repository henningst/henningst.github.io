---
layout:     post
title:      Referencing .NET 4.5.1 libraries in ASP.NET Core
date:       2016-05-20 21:30:00
summary:    There are still a lot of .NET libraries that are not updated with support for .NET Core. This article shows how you can still use these libraries in an ASP.NET Core 1 RC2 project.
categories: atlasboard, azure, javascript
---


[ASP.NET Core 1 RC2](https://blogs.msdn.microsoft.com/webdev/2016/05/16/announcing-asp-net-core-rc2/) recently shipped and I'm in the process of upgrading all my RC1 projects
to RC2. The most difficult part so far has been to get project.json set up with the correct
dependencies and frameworks. I'm using a few libraries that are not yet supporting .NET Core
and I therefore have to run my apps on the full .NET framework.


### RC1 example
In RC1 you typically had a project.json file like the one below. In this example I've added MongoDB.Driver 2.2.3 which
does not run on .NET Core. To get this running, you would typically remove the dnxcore50 framework moniker from the
stanard Visual Studio project template.

{% highlight json %}
{
  "version": "1.0.0-*",
  "description": "MyService Class Library",

  "dependencies": {
    "MongoDB.Driver": "2.2.3"
  },

  "frameworks": {
    "dnx451": { }
  }
}
{% endhighlight %}


### Upgrading to RC2

When you create new class library projects in RC2, your project.json will look a bit different because there
are a few changes to the way you specify dependencies and target frameworks. If you use the default project template
after installing the [.NET Core 1 RC2 Tooling Preview 1](https://go.microsoft.com/fwlink/?LinkId=798481) your project
will be configured to target netstarndard1.5. As you can se below, it will also include an import for dnxcore50 in
order to support libraries using the old .NET Core monikers.
 
{% highlight json %}
{
  "version": "1.0.0-*",

  "dependencies": {
    "NETStandard.Library": "1.5.0-rc2-24027",
    "MongoDB.Driver": "2.2.4"
  },

  "frameworks": {
    "netstandard1.5": {
      "imports": "dnxcore50"
    }
  }
}
{% endhighlight %}


When we add the dependency `MongoDB.Driver 2.2.4` and run `dotnet restore` you'll get the error shown below.

```
error: Package MongoDB.Driver 2.2.4 is not compatible with netcoreapp1.0 (.NETCoreApp,Version=v1.0). Package MongoDB.Driver 2.2.4 supports: net45 (.NETFramework,Version=v4.5)
error: Package MongoDB.Bson 2.2.4 is not compatible with netcoreapp1.0 (.NETCoreApp,Version=v1.0). Package MongoDB.Bson 2.2.4 supports: net45 (.NETFramework,Version=v4.5)
error: Package MongoDB.Driver.Core 2.2.4 is not compatible with netcoreapp1.0 (.NETCoreApp,Version=v1.0). Package MongoDB.Driver.Core 2.2.4 supports: net45 (.NETFramework,Version=v4.5)
```

As the error message is saying, the MongoDB packages are not compatible with the framework we have specified. It also says that it _does_ support net45.

To fix this we have to remove `netstandard1.5` from the frameworks section and add `net451` as shown below. 

{% highlight json %}
{
  "version": "1.0.0-*",

  "dependencies": {
    "NETStandard.Library": "1.5.0-rc2-24027",
    "MongoDB.Driver": "2.2.4"
  },

  "frameworks": {
      "net451": {}
  }
}
{% endhighlight %}

You should now be able to restore nuget packages and build the project. Be aware that your library is no longer a cross platform library
and you need to run on the full .net framework.

## Adding a test project

If you want to add a test project for your class library, you can use xUnit by following the [Getting started with xUnit.net](http://xunit.github.io/docs/getting-started-dotnet-core.html)
instructions. However, there are a couple of tweaks you need to do in order to get this working.

Remove the `netcoreapp1.0` section from frameworks and instead add `net451` the same way we did in the class library above. This is necessary
because our class library are only supporting `net451`.

If we try to run the tests now, we still get an error because of an [issue](https://github.com/xunit/xunit/issues/843) with the xunit test runner.

```
$ dotnet test
Project MySuperLibrary (.NETFramework,Version=v4.5.1) was previously compiled. Skipping compilation.
Project MySuperLibraryTests (.NETFramework,Version=v4.5.1) will be compiled because expected outputs are missing
Compiling MySuperLibraryTests for .NETFramework,Version=v4.5.1

Compilation succeeded.
    0 Warning(s)
    0 Error(s)

Time elapsed 00:00:02.0726851


xUnit.net .NET CLI test runner (32-bit win10-x86)
System.DllNotFoundException: Unable to load DLL 'Microsoft.DiaSymReader.Native.x86.dll': The specified module could not be found. (Exception from HRESULT: 0x8007007E)
SUMMARY: Total: 1 targets, Passed: 0, Failed: 1.
```

To fix this, change the `dotnet-test-xunit` dependency to use the rc3 version instead of rc2. 
This nuget package has to be pulled from the xUnit myget feed. If you haven't 
already set up the myget feed, add a NuGet.config file to the root directory of your solution (on the same level as the src and test directory).

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<configuration>
    <packageSources>
      <add key="myget.org xunit" value="https://www.myget.org/F/xunit/api/v3/index.json" />
      <add key="NuGet" value="https://api.nuget.org/v3/index.json" />
    </packageSources>
</configuration>
{% endhighlight %}


After fixing the issues mentioned abov, the project.json for the test project should now look like this and you should be 
able to successfully run `dotnet test`:

{% highlight json %}
{
  "version": "1.0.0-*",

  "buildOptions": {
    "preserveCompilationContext": true
  },

  "dependencies": {
    "dotnet-test-xunit": "1.0.0-rc3-*",
    "xunit": "2.1.0-rc2-*",
    "MySuperLibrary": "1.0.0-*"
  },

  "frameworks": {
    "net451": {}
  },

  "testRunner": "xunit",

  "tooling": {
    "defaultNamespace": "MySuperLibraryTests"
  }
}
{% endhighlight %}

A complete sample solution is available on [Github](https://github.com/henningst/TestingNet451Projects).