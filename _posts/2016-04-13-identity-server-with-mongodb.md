---
layout:     post
title:      Using MongoDB as store for IdentityServer 4
date:       2016-03-21 21:44:00
summary:    Sample implementation on how to use MongoDB to implement stores for users and clients in IdentityServer 4.
categories: aspnet-core
---

This blog posts shows how you can use MongoDB as persistence for your users and clients in IdentityServer 4. 
I've used the MVC Sample from the [IdentityServer4.Sample repository](https://github.com/IdentityServer/IdentityServer4.Samples/tree/master/Mvc) as a starting point 
and replaced the InMemory version of the client store and user store.

The complete solution can also be found on GitHub at <https://github.com/henningst/IdentityServer4-MongoDB-Sample/>

I've decided to implement one common repository class that takes care of all the interaction with MongoDB and then use this repository from
the various services needed by IdentityServer.

The code snippet below shows how I've changed the configuration of services to replace the in-memory implementation
with a custom MongoDB implementation. I've commented out the lines where clients and users are added and instead added
my own implementations.

{% highlight csharp lineanchors %}
public void ConfigureServices(IServiceCollection services)
{
    var cert = new X509Certificate2(Path.Combine(_environment.ApplicationBasePath, "idsrv4test.pfx"), "idsrv3test");

    var builder = services.AddIdentityServer(options =>
    {
        options.SigningCertificate = cert;
    });

    //builder.AddInMemoryClients(Clients.Get());
    //builder.AddInMemoryUsers(Users.Get());
    builder.AddInMemoryScopes(Scopes.Get());

    builder.Services.AddTransient<IRepository, MongoDbRepository>();
    builder.Services.AddTransient<IClientStore, MongoDbClientStore>();
    builder.Services.AddTransient<IProfileService, MongoDbProfileService>();
    builder.Services.AddTransient<IResourceOwnerPasswordValidator, MongoDbResourceOwnerPasswordValidator>();
    builder.Services.AddTransient<IPasswordHasher<MongoDbUser>, PasswordHasher<MongoDbUser>>();
    builder.Services.Configure<MongoDbRepositoryConfiguration>(_configuration.GetSection("MongoDbRepository"));
    builder.AddCustomGrantValidator<CustomGrantValidator>();

    // for the UI
    services
        .AddMvc()
        .AddRazorOptions(razor =>
        {
            razor.ViewLocationExpanders.Add(new IdSvrHost.UI.CustomViewLocationExpander());
        });

    services.AddTransient<IdSvrHost.UI.Login.LoginService>();
}
{% endhighlight %}

Now that I've shown you how to wire up the services, I'll go ahead and implement them.


### Implementing the MongoDB repository

The first thing we'll do is to implement a repository that will take care of all the communication with MongoDB. For simplicity I've decided
to implement everything in the IdSrvHost project, but you could just as well move this into a separate project.


Below you can see the json representation of a user and client the way they will be stored in MongoDB. You probably want to 
extend these models with additional properties, but this is a minimum to get you started. 

User:
{% highlight json lineanchors %}
{
    "_id": "d8b1e4e2-d27a-42fe-a382-d111abc0d026",
    "Username": "henning",
    "HashedPassword": "AQAAAAEAACcQAAAAEMEidxhwmXKdtvoPEX8a1aFdaFAwXuMRv7YLSmKsQqiImbMnhkSTkOAxPhbNVAx64w==",
    "IsActive": "true",
    "FirstName": "Henning",
    "LastName": "Støverud",
    "Email": "henning@stoverud.no",
    "EmailVerified": "true"
}
{% endhighlight %}

For simplicity I've stored ClientSecrets in MongoDB as plain text. You probably want to hash it before persisting it in a real world app.

Client:
{% highlight json lineanchors %}
{
    "_id": {
        "$oid": "57136dc6e4b065a8c4d71e91"
    },
    "ClientId": "mvc_implicit",
    "Flow": 1,
    "RedirectUris": [
        "http://localhost:5000/signin-oidc"
    ],
    "ClientSecrets": [
        "secret"
    ],
    "AllowedScopes": [
        "openid",
        "profile",
        "email",
        "roles",
        "api1",
        "api2"
    ]
}
{% endhighlight %}

To interact with the database, I first define an interface with 4 methods. We need to be able to retrieve a user by username, 
retrieve a user by ID, validate the password for a given user and retrieve a client by id. You probably need a few more methods 
to handle all relevant CRUD operations, but that should be pretty straight forward.

{% highlight csharp lineanchors %}
using IdSvrHost.Models;

namespace IdSvrHost.Services
{
    public interface IRepository
    {
        MongoDbUser GetUserByUsername(string username);
        MongoDbUser GetUserById(string id);
        bool ValidatePassword(string username, string plainTextPassword);
        MongoDbClient GetClient(string clientId);
    }
}
{% endhighlight %}

Below you can see the full implementation of the IRepository. I'm not a MongoDB expert, so there might be better ways to implement this, but it works :)


{% highlight csharp lineanchors %}
using System;
using IdSvrHost.Models;
using Microsoft.AspNet.Identity;
using Microsoft.Extensions.OptionsModel;
using MongoDB.Driver;

namespace IdSvrHost.Services
{
    public class MongoDbRepository : IRepository
    {
        private readonly IPasswordHasher<MongoDbUser> _passwordHasher;
        private readonly IMongoDatabase _db;
        private const string UsersCollectionName = "Users";
        private const string ClientsCollectionName = "Clients";
        
        /// <summary>
        /// Get configuration and password hasher via constructor parameters.
        /// </summary>
        /// <param name="config"></param>
        /// <param name="passwordHasher"></param>
        public MongoDbRepository(IOptions<MongoDbRepositoryConfiguration> config, IPasswordHasher<MongoDbUser> passwordHasher)
        {
            _passwordHasher = passwordHasher;
            var client = new MongoClient(config.Value.ConnectionString);
            _db = client.GetDatabase(config.Value.DatabaseName);
        }

        /// <summary>
        /// Retrieve a user by username
        /// </summary>
        /// <param name="username"></param>
        /// <returns></returns>
        public MongoDbUser GetUserByUsername(string username)
        {
            var collection = _db.GetCollection<MongoDbUser>(UsersCollectionName);
            var filter = Builders<MongoDbUser>.Filter.Eq(u => u.Username, username);
            return collection.Find(filter).SingleOrDefaultAsync().Result;
        }

        /// <summary>
        /// Retrieve a user by ID
        /// </summary>
        /// <param name="id"></param>
        /// <returns></returns>
        public MongoDbUser GetUserById(string id)
        {
            var collection = _db.GetCollection<MongoDbUser>(UsersCollectionName);
            var filter = Builders<MongoDbUser>.Filter.Eq(u => u.Id, id);
            return collection.Find(filter).SingleOrDefaultAsync().Result;
        }

        /// <summary>
        /// Validate the given plainTextPassword against the hashed password for the given user.
        /// </summary>
        /// <param name="username"></param>
        /// <param name="plainTextPassword"></param>
        /// <returns></returns>
        public bool ValidatePassword(string username, string plainTextPassword)
        {
            var user = GetUserByUsername(username);
            if (user == null)
            {
                return false;
            }

            var result = _passwordHasher.VerifyHashedPassword(user, user.HashedPassword, plainTextPassword);
            switch (result)
            {
                case PasswordVerificationResult.Success:
                    return true;
                case PasswordVerificationResult.Failed:
                    return false;
                case PasswordVerificationResult.SuccessRehashNeeded:
                    throw new NotImplementedException();
                default:
                    throw new NotImplementedException();
            }
        }

        /// <summary>
        /// Retrieve a client by ID
        /// </summary>
        /// <param name="clientId"></param>
        /// <returns></returns>
        public MongoDbClient GetClient(string clientId)
        {
            var collection = _db.GetCollection<MongoDbClient>(ClientsCollectionName);
            var filter = Builders<MongoDbClient>.Filter.Eq(x => x.ClientId, clientId);
            return collection.Find(filter).SingleOrDefaultAsync().Result;
        }
    }
}
{% endhighlight %}


For this implementation you will need a couple of additional dependencies in project.json. 
I've chosen to use the password hasher provided by Microsoft.AspNet.Identity.

And of course you need the [MongoDB.Driver](http://mongodb.github.io/mongo-csharp-driver/) package to do the actual database queries. The MongoDB.Driver does not support dnxcore50, so you need
to remove this from the frameworks section and only target dnx451. 

{% highlight csharp lineanchors %}
    "Microsoft.AspNet.Identity": "3.0.0-rc1-final",
    "MongoDB.Driver": "2.2.3"
{% endhighlight %}

I'm also IOptions to inject a strongly typed configuration class via the constructor. Make sure to set your MongoDB connection string and database name in appsettings.json or another valid
configuration source.


Now that we have the necessary parts of the repository in place, we can continue implementing the interfaces needed by IdentityServer.


### Implementing the User store

There are two interfaces you need to implement in order to have a working user store; IProfileService and IResourceOwnerPasswordValidator.

In each implementation we get the repository injected via the constructor. This is handled by the built in [dependency injection mechanism
in ASP.NET Core](http://docs.asp.net/en/latest/fundamentals/dependency-injection.html).

The MongoDbProfileService is basically just retrieving a user from MongoDB and mapping it to claims which are set on the context.


{% highlight csharp lineanchors %} 
using System.Collections.Generic;
using System.Security.Claims;
using System.Threading.Tasks;
using IdentityModel;
using IdentityServer4.Core.Extensions;
using IdentityServer4.Core.Models;
using IdentityServer4.Core.Services;

namespace IdSvrHost.Services
{
    public class MongoDbProfileService : IProfileService
    {
        private readonly IRepository _repository;

        public MongoDbProfileService(IRepository repository)
        {
            _repository = repository;
        }

        public Task GetProfileDataAsync(ProfileDataRequestContext context)
        {
            var subjectId = context.Subject.GetSubjectId();

            var user = _repository.GetUserById(subjectId);

            var claims = new List<Claim>
            {
                new Claim(JwtClaimTypes.Subject, user.Id),
                new Claim(JwtClaimTypes.Name, $"{user.FirstName} {user.LastName}"),
                new Claim(JwtClaimTypes.GivenName, user.FirstName),
                new Claim(JwtClaimTypes.FamilyName, user.LastName),
                new Claim(JwtClaimTypes.Email, user.Email),
                new Claim(JwtClaimTypes.EmailVerified, user.EmailVerified.ToString().ToLower(), ClaimValueTypes.Boolean)
            };

            context.IssuedClaims = claims;

            return Task.FromResult(0);
        }

        public Task IsActiveAsync(IsActiveContext context)
        {
            var user = _repository.GetUserById(context.Subject.GetSubjectId());

            context.IsActive = (user != null) && user.IsActive;
            return Task.FromResult(0);
        }
    }
}

{% endhighlight %}


The next interface we need to implement is the IResourceOwnerPasswordValidator. Again we are simply injecting the IRepository and calling the appropriate methods we implemented earlier.


{% highlight csharp lineanchors %}
using System.Threading.Tasks;
using IdentityServer4.Core.Validation;

namespace IdSvrHost.Services
{
    public class MongoDbResourceOwnerPasswordValidator : IResourceOwnerPasswordValidator
    {
        private readonly IRepository _repository;

        public MongoDbResourceOwnerPasswordValidator(IRepository repository)
        {
            _repository = repository;
        }

        public Task<CustomGrantValidationResult> ValidateAsync(string userName, string password, ValidatedTokenRequest request)
        {
            if (_repository.ValidatePassword(userName, password))
            {
                return Task.FromResult(new CustomGrantValidationResult(userName, "password"));
            }
       
            return Task.FromResult(new CustomGrantValidationResult("Wrong username or password"));
        }
    }
}
{% endhighlight %}


In order to retrieve the clients from MongoDB, we'll also implement IClientStore. Nothing fancy here either. We're just retrieving the
client via our repository and mapping it to a Client object. As I mentioned above, I'm keeping the client secrets as plain text in mongo db. Here you
can see I'm hashing it using the Sha256() extension method before returning it. You probably want to hash it before storing it to MongoDB, and in that case
you should also remove the redundant hashing before returning it by FindClientByIdAsync. 

{% highlight csharp lineanchors %}
using System.Linq;
using System.Threading.Tasks;
using IdentityServer4.Core.Models;
using IdentityServer4.Core.Services;

namespace IdSvrHost.Services
{
    public class MongoDbClientStore : IClientStore
    {
        private readonly IRepository _repository;

        public MongoDbClientStore(IRepository repository)
        {
            _repository = repository;
        }

        public Task<Client> FindClientByIdAsync(string clientId)
        {
            var client = _repository.GetClient(clientId);
            if (client == null)
            {
                return Task.FromResult<Client>(null);
            }

            return Task.FromResult(new Client()
            {
                ClientId = client.ClientId,
                Flow = client.Flow,
                AllowedScopes = client.AllowedScopes,
                RedirectUris = client.RedirectUris,
                ClientSecrets = client.ClientSecrets.Select(s => new Secret(s.Sha256())).ToList()
            });
        }
    }
}
{% endhighlight %}



Finally we'll change the LoginService to use our repository instead of the InMemory users. 

{% highlight csharp lineanchors %}
using IdSvrHost.Models;
using IdSvrHost.Services;

namespace IdSvrHost.UI.Login
{
    public class LoginService
    {
        private readonly IRepository _repository;

        public LoginService(IRepository repository)
        {
            _repository = repository;
        }

        public bool ValidateCredentials(string username, string password)
        {
            return _repository.ValidatePassword(username, password);
        }

        public MongoDbUser FindByUsername(string username)
        {
            return _repository.GetUserByUsername(username);
        }
    }
}
{% endhighlight %}

You should now have a a working IdentityServer4 where the users and clients are retrieved from MongoDB.

Complete source: <https://github.com/henningst/IdentityServer4-MongoDB-Sample/>