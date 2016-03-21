---
layout:     post
title:      How to unit test your ASP.NET Core AuthorizationHandler
date:       2015-03-21 21:44:00
summary:    With the new authorization story in ASP.NET Core, you get a lot more flexibility when it comes to handling authorization in your apps. Here I'll walk through how you can implement resource based authorization and how to unit test your AuthorizationHandler.
categories: aspnet-core
---

With the new authorization story in ASP.NET Core, you get a lot more flexibility when it comes to handling authorization in your apps. Here I'll walk through how you can implement resource based authorization and how to unit test your AuthorizationHandler.

To get a good introduction to how you can implement resource based authorization, have a look at the [official ASP.NET documentation](https://docs.asp.net/en/latest/security/authorization/resourcebased.html)[^1].

A common scenario is that you want to return a view from a controller action with a model and you want to make sure that only authorized users can access this data. By using the new IAuthorizationService, you can now check if the current user is authorized for the given resource and operation as shown below.

{% highlight csharp lineanchors %}
public class HomeController : Controller
{
    private readonly IAuthorizationService _authz;

    public HomeController(IAuthorizationService authz)
    {
        _authz = authz;
    }

    public async Task<IActionResult> Index()
    {
        // Create a document owned by bob
        var document = new Document() { Name = "My document name", Owner = "bob"};

        // Check if the current user is authorized by using the CustomAuthorizationHandler
        if (!await _authz.AuthorizeAsync(User, document, new OperationAuthorizationRequirement() {Name = "Read"}))
        {
            return new ChallengeResult();
        }

        return View(document);
    }
}
{% endhighlight %}



To make the call to AuthorizeAsync(..) work, you have to implement an AuthorizationHandler matching your resource (in our case, Document) and operation requirement.

Here is what the AuthorizationHandler looks like:

{% highlight csharp lineanchors %}
using System.Security.Claims;
using AuthorizationDemo.Models;
using Microsoft.AspNet.Authorization;
using Microsoft.AspNet.Authorization.Infrastructure;

namespace AuthorizationDemo.Authz
{
    public class CustomAuthorizationHandler : AuthorizationHandler<OperationAuthorizationRequirement,Document>
    {
        protected override void Handle(AuthorizationContext context, OperationAuthorizationRequirement requirement, Document resource)
        {
            var claim = context.User.FindFirst(ClaimTypes.Name);
            if(claim != null && claim.Value.Equals(resource.Owner))
            {
                context.Succeed(requirement);
            }
        }
    }
}
{% endhighlight %}



We inherit from AuthorizationHandler<TRequiremement, TResource> which in turn implements the IAuthorizationHandler interface.

Our handler is pretty simple and just makes sure the claim containing the name of the currently logged in user is matching the name of the document owner. In in a real world scenario you could have a more sophisticated handler like maybe a permission service injected via DI, or any kind of custom logic.

To make sure your CustomAuthorizationHandler is called, you must register it as a service by adding the following in the ConfigureServices methos in Startup.cs.

{% highlight csharp lineanchors %}
services.AddSingleton<IAuthorizationHandler, CustomAuthorizationHandler>();
{% endhighlight %}


### Adding unit tests

Now we want to add a couple of unit tests to make sure our CustomAuthorizationHandler works as intended. I've added a separate test project and added the necessary references to xUnit.

First I'll add a test to make sure it succeeds if the current user is the document owner.

{% highlight csharp lineanchors %}
[Fact]
public void Handle_WhenCalledWithResourceOwner_ShouldSucceed()
{
    var resource = new Document() {Name = "Homer's document", Owner = "homer.simpson"};
    var user = new ClaimsPrincipal(new ClaimsIdentity(new List<Claim> { new Claim(ClaimTypes.Name, "homer.simpson") }));
    var requirement = new OperationAuthorizationRequirement {Name = "Read"};

    var authzContext = new AuthorizationContext(new List<IAuthorizationRequirement> { requirement }, user, resource);

    var authzHandler = new CustomAuthorizationHandler();
    authzHandler.Handle(authzContext);

    Assert.True(authzContext.HasSucceeded);
}
{% endhighlight %}


I'll also implement a test that checks that the handler does not succeed if the current user is not the document owner. Notice that we check if HasSucceeded is false! We are not checking if HasFailed is true because you should normally not have your authorization handlers Fail.

{% highlight csharp lineanchors %}
[Fact]
public void Handle_WhenCalledWithIllegalUser_ShouldNotSucceed()
{
    var resource = new Document() { Name = "Homer's document", Owner = "homer.simpson" };
    var user = new ClaimsPrincipal(new ClaimsIdentity(new List<Claim> { new Claim(ClaimTypes.Name, "ned.flanders") }));
    var requirement = new OperationAuthorizationRequirement { Name = "Read" };

    var authzContext = new AuthorizationContext(new List<IAuthorizationRequirement> { requirement }, user, resource);

    var authzHandler = new CustomAuthorizationHandler();
    authzHandler.Handle(authzContext);

    Assert.False(authzContext.HasSucceeded);
}
{% endhighlight %}


You can find a complete sample at [https://github.com/henningst/ASPNETCore-AuthorizationDemo](https://github.com/henningst/ASPNETCore-AuthorizationDemo)


### Resources
[ASP.NET Documentation on Resource Based Authorization](https://docs.asp.net/en/latest/security/authorization/resourcebased.html)

[Get started with ASP.NET Core Authorization](https://blogs.msdn.microsoft.com/webdev/2016/03/15/get-started-with-asp-net-core-authorization-part-1-of-2/)

[A run around the new ASP.NET Data Protection & Authorization Stacks - Barry Dorrans](https://vimeo.com/153102690)


---

[^1]: https://docs.asp.net/en/latest/security/authorization/resourcebased.html
