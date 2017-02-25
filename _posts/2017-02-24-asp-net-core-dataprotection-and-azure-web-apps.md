---
layout:     post
title:      The antiforgery token could not be decrypted - Running ASP.NET Core on Azure App Service using deployment slots
date:       2017-02-24 14:50:00
summary:    If you are deploying ASP.NET Core webapps to Azure App Service and you are using deployment slots, you might run into the same problems I did when using the data protection features.
categories: aspnet-core azure
---

## TL;DR

If you are seeing this exception in you ASP.NET Core web app running in Azure App Service:
```
System.InvalidOperationException: The antiforgery token could not be decrypted. ---> System.Security.Cryptography.CryptographicException: The key {9725081b-7caf-4642-ae55-93cf9c871c36} was not found in the key ring.
```
chances are you are using deployment slots and that your Data Protection Keys are not matching. The default Data Protection configuration does not work when using
Azure Web App deployment slots, so you must either use a different key storage provider, or stop using deployment slots.


There is no out of the box solution for deployment slots, so either implement your own
persistence mechanism for data protection keys or stop using  deployment slots.


## The issue


I recently ran into an issue in an ASP.NET Core web application I am running on Azure App Service. The site was sometimes throwing exceptions
when posting forms. After having a look in the logs, I discovered that a `CryptographicException` was thrown saying `The key {F6CAD132-A41B-49A9-954F-1BA0795072FF} was not found in the key ring.`

Everything worked fine when running the site on my local machine, so I figured this had something to do with how things worked on Azure. I then tried
to create a new Azure Web App and deployed the exact same app, and everything worked fine. I switched back my original Azure Web App, and all of a sudden it worked there aswell. Weird!
I did some small adjustments to the code and deployed to the original site, and the error started occuring again. After doing lots of deployments, I was starting to see a pattern.
The error occured every other time I deployed new code to the site and it seemed like it had something to do with how swapping between the staging and production deployment slot is working.



The Data Protection capabilities in ASP.NET Core is used to protect data, i.e. when you want to round trip some data via an untrusted client. 
You can read more about Data Protection in the [ASP.NET Core documentation](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/introduction).

The documentation states that "The system should offer simplicity of configuration.". This is true as long as you are deploying directly to the live site, 
but if you want to leverage deployment slots to get zero downtime deployments, you might get some nasty surprises.

When not using deployment slots, everything works fine because the data protection keys stored on disk is synchronized across all the machines hosting your web app, but when using
a deployment slot, you will end up with two separate keys. I assumed this would "just work" when running ASP.NET Core in Azure App Service, but that assumption was obviously wrong. 

A couple of sentences about this issue was added to the [documentation](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/configuration/default-settings) a while back, but this is not something you will probably discover
before you are facing the issue.

> If the system is being hosted in Azure Web Sites, keys are persisted to the "%HOME%\ASP.NET\DataProtection-Keys" folder. This folder is backed by network storage and is synchronized across all machines hosting the application. Keys are not protected at rest. This folder supplies the key ring to all instances of an application in a single deployment slot. Separate deployment slots, such as Staging and Production, will not share a key ring. When you swap between deployment slots, for example swapping Staging to Production or using A/B testing, any system using data protection will not be able to decrypt stored data using the key ring inside the previous slot. This will lead to users being logged out of an ASP.NET application that uses the standard ASP.NET cookie middleware, as it uses data protection to protect its cookies. If you desire slot-independent key rings, use an external key ring provider, such as Azure Blob Storage, Azure Key Vault, a SQL store, or Redis cache.
> <br />_Source: [https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/configuration/default-settings](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/configuration/default-settings)_

In addition to problems with anti forgery tokens, this problem also applies to authentication cookies, so users who are logged in when you deploy new versions and swap between staging and deployment, will also experience this issue.


## How to reproduce

- Create an Azure Web App with a separate deployment slot (i.e. a slot called staging). You can see in the screenshot below, that I have a site called DataProtectionSample... with a separate slot for staging.

![Screenshot of slot setup]({{ site.url }}/assets/img/posts/dataprotection/azure-webapp-slot.png)


- Now try to deploy an ASP.NET Core MVC app with a form where you apply the attribute `[ValidateAntiForgeryToken]` to the action you post to. The action can i.e. look like this:

{% highlight csharp lineanchors %}
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> MyAction(MyModel model)
{
    return View("Index", model);
}
{% endhighlight %}

In your view, you can have just a plain form posting to the action above.

{% highlight html lineanchors %}
<form asp-controller="Home" asp-action="MyAction" method="post" class="form-horizontal">
    <label asp-for="MyText">Text</label>
    <input asp-for="MyText" />
    <input type="submit"/>
</form>
{% endhighlight %}


- When you have the web app running, load the form (but don't try to submit yet). 

If you have a look at the markup, you can see a hidden input tag containing a `__RequestVerificationToken`. This token is generated by the server
and is validated when you post the form to make sure the form was actually generated by our app.

{% highlight html lineanchors %}
<input name="__RequestVerificationToken" type="hidden" value="CfDJ8INSXJPRzp5Kj_auYHfr4NM9Soli2TnvMdIpfluwUi-EdWzYKC2NtKYw9EdoK7vsB4ThC-njdo4CHVzjkIxgfjXTUb5nHvDAoKTQn84TDXug7othtHS0nmKvbe7Pieqh76NzAPko87cN7JSVkLzxsPE">
{% endhighlight %}

- Now try to deploy your app to the deployment slot and then do a swap between `staging` and `production`. This is a typical approach if you want to have zero downtime deployment.

- When your site has been deployed, try to submit the form. You should now see the following exception:

```
System.InvalidOperationException: The antiforgery token could not be decrypted. ---> System.Security.Cryptography.CryptographicException: The key {9725081b-7caf-4642-ae55-93cf9c871c36} was not found in the key ring.
   at Microsoft.AspNetCore.DataProtection.KeyManagement.KeyRingBasedDataProtector.UnprotectCore(Byte[] protectedData, Boolean allowOperationsOnRevokedKeys, UnprotectStatus& status)
   at Microsoft.AspNetCore.DataProtection.KeyManagement.KeyRingBasedDataProtector.DangerousUnprotect(Byte[] protectedData, Boolean ignoreRevocationErrors, Boolean& requiresMigration, Boolean& wasRevoked)
   at Microsoft.AspNetCore.DataProtection.KeyManagement.KeyRingBasedDataProtector.Unprotect(Byte[] protectedData)
   at Microsoft.AspNetCore.Antiforgery.Internal.DefaultAntiforgeryTokenSerializer.Deserialize(String serializedToken)
```

The app is now unable to decrypt the `__RequestVerificationToken`. 

## Why is this happening

What happened here, is that the `__RequestVerificationToken` you saw in the markup when the page was created, was generated by our app when it was running in the production instance.
This token is generated by the data protection API and it is using an encryption key stored in the directory `%HOME%\ASP.NET\DataProtection-keys` on the Azure Web App.

Then we triggered a new deployment _before_ the form was submitted. The deployment copied our app to the staging slot, which is basically a separate web app running side-by-side with the
production instance. The staging slot also has a file with an encryption key stored in `%HOME%\ASP.NET\DataProtection-keys`, but unfortunately, this is a different file with a different key.

When we swap the staging slot with production, the file with the data protection key is also swapped! When we then post our form and out app attempts to
validate the `__RequestVerificationToken`, it fails because we are unable to find the correct data protection key on disk. 

You can see this in action by using the Kudu site by navigating to https://[your-web-app].scm.azurewebsites.net/DebugConsole and opening the directory `ASP.NET\DataProtection-keys`.

If you do the same on your staging site, you will see that you have a different file with a different key.



## Solution

So how can we fix this problem? Well, one solution is to just not use deployment slots. This is obviously a bad solution because there are a lot of good
reasons to use deployment slots (The ability to deploy with zero warm up time and zero downtime are just a few).

After digging through various github issues it turns out you can configure Data Protection to store the keys to [Azure Blob Storage or Redis](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/implementation/key-storage-providers#azure-and-redis)
by using alternative [key storage providers](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/implementation/key-storage-providers). Thanks to [GuardRex](https://github.com/GuardRex) for pointing
me in the [right direction](https://github.com/aspnet/DataProtection/issues/92#issuecomment-282365822).


Since I already has a Redis instance up and running, I chose to use the Redis provider. There are just a couple of simple steps you need to do in order to
get the Redis Key Storage Provider up and running. 

- First add the following NuGet package to your project: `"Microsoft.AspNetCore.DataProtection.Redis"`
- Then in `Startup.cs` configure Data Protection to use the Redis Key Storage Provider instead of the default configuration like this:

```
var redis = ConnectionMultiplexer.Connect("[your-redis-server-instance-here].redis.cache.windows.net:6380,password=[your-redis-password-here],ssl=True,abortConnect=False");
services.AddDataProtection().PersistKeysToRedis(redis, "DataProtection-Keys");
```
- Deploy the app and everything should now work fine even when swapping deployment slots!





## Some links


When I first ran into this issue, it was difficult to find out what was actually wrong. Here are some of the resources I went through when 
looking for a solution:


Introduction to Data Protection<br/>
[https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/introduction](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/introduction)


Key Storage Providerd documentation<br/>
[https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/implementation/key-storage-providers](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/implementation/key-storage-providers)


Relevant Github issues<br/>
[https://github.com/aspnet/DataProtection/issues/92](https://github.com/aspnet/DataProtection/issues/92)
[https://github.com/aspnet/Docs/issues/2334](https://github.com/aspnet/Docs/issues/2334)
