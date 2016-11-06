# Ocelot

[![Build status](https://ci.appveyor.com/api/projects/status/roahbe4nl526ysya?svg=true)](https://ci.appveyor.com/project/TomPallister/ocelot)

[![Join the chat at https://gitter.im/Ocelotey/Lobby](https://badges.gitter.im/Ocelotey/Lobby.svg)](https://gitter.im/Ocelotey/Lobby?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

Attempt at a .NET Api Gateway

This project is aimed at people using .NET running 
a micro services / service orientated architecture 
that need a unified point of entry into their system.

In particular I want easy integration with 
IdentityServer reference and bearer tokens. 

We have been unable to find this in my current workplace
without having to write our own Javascript middlewares 
to handle the IdentityServer reference tokens. We would
rather use the IdentityServer code that already exists
to do this.

This is not ready for production yet as uses a lot of rc and beta .net core packages.
Hopefully by the start of 2017 it will be in use.

## Contributing

Pull requests, issues and commentary welcome! No special process just create a request and get in 
touch either via gitter or create an issue. 

## How to install

Ocelot is designed to work with ASP.NET core only and is currently 
built to netcoreapp1.4 [this](https://docs.microsoft.com/en-us/dotnet/articles/standard/library) documentation may prove helpful when working out if Ocelot would be suitable for you.

Install Ocelot and it's dependecies using nuget. At the moment 
all we have is the pre version. Once we have something working in 
a half decent way we will drop a version.

`Install-Package Ocelot -Pre`

All versions can be found [here](https://www.nuget.org/packages/Ocelot/)

## Configuration

An example configuration can be found [here](https://github.com/TomPallister/Ocelot/blob/develop/test/Ocelot.ManualTest/configuration.json) 
and an explained configuration can be found [here](https://github.com/TomPallister/Ocelot/blob/develop/configuration-explanation.txt). More detailed instructions to come on how to configure this.

There are two sections to the configuration. An array of ReRoutes and a GlobalConfiguration. 
The ReRoutes are the objects that tell Ocelot how to treat an upstream request. The Global 
configuration is a bit hacky and allows overrides of ReRoute specific settings. It's useful
if you don't want to manage lots of ReRoute specific settings.

		{
			"ReRoutes": [],
			"GlobalConfiguration": {}
		}

More information on how to use these options is below..

## Startup

An example startup using a json file for configuration can be seen below. 
Currently this is the only way to get configuration into Ocelot.

	 public class Startup
    {
        public Startup(IHostingEnvironment env)
        {
            var builder = new ConfigurationBuilder()
                .SetBasePath(env.ContentRootPath)
                .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
                .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true)
                .AddJsonFile("configuration.json")
                .AddEnvironmentVariables();

            Configuration = builder.Build();
        }

        public IConfigurationRoot Configuration { get; }

        public void ConfigureServices(IServiceCollection services)
        {
            Action<ConfigurationBuilderCachePart> settings = (x) =>
            {
                x.WithMicrosoftLogging(log =>
                {
                    log.AddConsole(LogLevel.Debug);
                })
                .WithDictionaryHandle();
            };

            services.AddOcelotOutputCaching(settings);
            services.AddOcelotFileConfiguration(Configuration);
            services.AddOcelot();
        }

        public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
        {
            loggerFactory.AddConsole(Configuration.GetSection("Logging"));

            app.UseOcelot();
        }
    }


This is pretty much all you need to get going.......more to come! 

## Routing

Ocelot's primary functionality is to take incomeing http requests and forward them on
to a downstream service. At the moment in the form of another http request (in the future
this could be any transport mechanism.). 

Ocelot's describes the routing of one request to another as a ReRoute. In order to get 
anything working in Ocelot you need to set up a ReRoute in the configuration.

		{
			"ReRoutes": [
			]
		}

In order to set up a ReRoute you need to add one to the json array called ReRoutes like
the following.

		{
            "DownstreamTemplate": "http://jsonplaceholder.typicode.com/posts/{postId}",
            "UpstreamTemplate": "/posts/{postId}",
            "UpstreamHttpMethod": "Put"
        }

The DownstreamTemplate is the URL that this request will be forwarded to.
The UpstreamTemplate is the URL that Ocelot will use to identity which 
DownstreamTemplate to use for a given request. Finally the UpstreamHttpMethod is used so
Ocelot can distinguish between requests to the same URL and is obviously needed to work :)
In Ocelot you can add placeholders for variables to your Templates in the form of {something}.
The placeholder needs to be in both the DownstreamTemplate and UpstreamTemplate. If it is
Ocelot will attempt to replace the placeholder with the correct variable value from the 
Upstream URL when the request comes in.

At the moment without any configuration Ocelot will default to all ReRoutes being case insensitive.
In order to change this you can specify on a per ReRoute basis the following setting.

		"ReRouteIsCaseSensitive": true

This means that when Ocelot tries to match the incoming upstream url with an upstream template the
evaluation will be case sensitive. This setting defaults to false so only set it if you want 
the ReRoute to be case sensitive is my advice!

## Authentication

Ocelot currently supports the use of bearer tokens with Identity Server (more providers to
come if required). In order to identity a ReRoute as authenticated it needs the following
configuration added.

		"AuthenticationOptions": {
            "Provider": "IdentityServer",
            "ProviderRootUrl": "http://localhost:52888",
            "ScopeName": "api",
            "AdditionalScopes": [
                "openid",
                "offline_access"
            ],
            "ScopeSecret": "secret"
        }

In this example the Provider is specified as IdentityServer. This string is important 
because it is used to identity the authentication provider (as previously mentioned in
the future there might be more providers). Identity server requires that the client
talk to it so we need to provide the root url of the IdentityServer as ProviderRootUrl.
IdentityServer requires at least one scope and you can also provider additional scopes.
Finally if you are using IdentityServer reference tokens you need to provide the scope
secret. 

Ocelot will use this configuration to build an authentication handler and if 
authentication is succefull the next middleware will be called else the response
is 401 unauthorised.

## Authorisation

Ocelot supports claims based authorisation which is run post authentication. This means if
you have a route you want to authorise you can add the following to you ReRoute configuration.

		"RouteClaimsRequirement": {
            "UserType": "registered"
        },

In this example when the authorisation middleware is called Ocelot will check to see
if the user has the claim type UserType and if the value of that claim is registered. 
If it isn't then the user will not be authorised and the response will be 403 forbidden.

## Claims Tranformation

Ocelot allows the user to access claims and transform them into headers, query string 
parameters and other claims. This is only available once a user has been authenticated.

After the user is authenticated we run the claims to claims transformation middleware.
This allows the user to transform claims before the authorisation middleware is called.
After the user is authorised first we call the claims to headers middleware and Finally
the claims to query strig parameters middleware.

The syntax for performing the transforms is the same for each proces. In the ReRoute
configuration a json dictionary is added with a specific name either AddClaimsToRequest,
AddHeadersToRequest, AddQueriesToRequest. 

Note I'm not a hotshot programmer so have no idea if this syntax is good..

Within this dictionary the entries specify how Ocelot should transform things! 
The key to the dictionary is going to become the key of either a claim, header 
or query parameter.

The value of the entry is parsed to logic that will perform the transform. First of
all a dictionary accessor is specified e.g. Claims[CustomerId]. This means we want
to access the claims and get the CustomerId claim type. Next is a greater than (>)
symbol which is just used to split the string. The next entry is either value or value with
and indexer. If value is specifed Ocelot will just take the value and add it to the 
transform. If the value has an indexer Ocelot will look for a delimiter which is provided
after another greater than symbol. Ocelot will then split the value on the delimiter 
and add whatever was at the index requested to the transform.

#### Claims to Claims Tranformation

Below is an example configuration that will transforms claims to claims

		 "AddClaimsToRequest": {
			"UserType": "Claims[sub] > value[0] > |",
			"UserId": "Claims[sub] > value[1] > |"
		},

This shows a transforms where Ocelot looks at the users sub claim and transforms it into
UserType and UserId claims. Assuming the sub looks like this "usertypevalue|useridvalue".

#### Claims to Headers Tranformation

Below is an example configuration that will transforms claims to headers

	   "AddHeadersToRequest": {
			"CustomerId": "Claims[sub] > value[1] > |"
		},

This shows a transform where Ocelot looks at the users sub claim and trasnforms it into a 
CustomerId header. Assuming the sub looks like this "usertypevalue|useridvalue".

#### Claims to Query String Parameters Tranformation

Below is an example configuration that will transforms claims to query string parameters

		"AddQueriesToRequest": {
            "LocationId": "Claims[LocationId] > value",
        },

This shows a transform where Ocelot looks at the users LocationId claim and add its as
a query string parameter to be forwarded onto the downstream service.

## Logging

Ocelot uses the standard logging interfaces ILoggerFactory / ILogger<T> at the moment. 
This is encapsulated in  IOcelotLogger / IOcelotLoggerFactory with an implementation 
for the standard asp.net core logging stuff at the moment. 

There are a bunch of debugging logs in the ocelot middlewares however I think the 
system probably needs more logging in the code it calls into. Other than the debugging
there is a global error handler that should catch any errors thrown and log them as errors.

The reason for not just using bog standard framework logging is that I could not 
work out how to override the request id that get's logged when setting IncludeScopes 
to true for logging settings. Nicely onto the next feature.

## RequestId / CorrelationId

Ocelot supports a client sending a request id in the form of a header. If set Ocelot will
use the requestid for logging as soon as it becomes available in the middleware pipeline. 
Ocelot will also forward the request id with the specified header to the downstream service.
I'm not sure if have this spot on yet in terms of the pipeline order becasue there are a few logs
that don't get the users request id at the moment and ocelot just logs not set for request id
which sucks. You can still get the framework request id in the logs if you set 
IncludeScopes true in your logging config. This can then be used to match up later logs that do
have an OcelotRequestId.

In order to use the requestid feature in your ReRoute configuration add this setting

		"RequestIdKey": "OcRequestId"

In this example OcRequestId is the request header that contains the clients request id.

There is also a setting in the GlobalConfiguration section which will override whatever has been
set at ReRoute level for the request id. The setting is as fllows.

		"RequestIdKey": "OcRequestId",

It behaves in exactly the same way as the ReRoute level RequestIdKey settings.

## Caching 

Ocelot supports some very rudimentary caching at the moment provider by 
the [CacheManager](http://cachemanager.net/) project. This is an amazing project
that is solving a lot of caching problems. I would reccomend using this package to 
cache with Ocelot. If you look at the example [here](https://github.com/TomPallister/Ocelot/blob/develop/test/Ocelot.ManualTest/Startup.cs)
you can see how the cache manager is setup and then passed into the Ocelot 
AddOcelotOutputCaching configuration method. You can use any settings supported by 
the CacheManager package and just pass them in.

Anyway Ocelot currently supports caching on the URL of the downstream service 
and setting a TTL in seconds to expire the cache. More to come!

In orde to use caching on a route in your ReRoute configuration add this setting.

		"FileCacheOptions": { "TtlSeconds": 15 }

In this example ttl seconds is set to 15 which means the cache will expire after 15 seconds.

## Not supported

Ocelot does not support...
	
+ Chunked Encoding - Ocelot will always get the body size and return Content-Length 
header. Sorry if this doesn't work for your use case! 
	
+ Fowarding a host header - The host header that you send to Ocelot will not be 
forwarded to the downstream service. Obviously this would break everything :(

## Things that are currently annoying means

+ The ReRoute configuration object is too large.

+ The base OcelotMiddleware lets you access things that are going to be null
and doesnt check the response is OK. I think the fact you can even call stuff
that isnt available is annoying. Let alone it be null.

## Coming up

You can see what we are working on [here](https://github.com/TomPallister/Ocelot/projects/1)


