---
layout: post
title:  "Manually validating Azure AD B2C/Microsoft identity platform JWT access tokens in ASP.NET"
date:   2020-05-20 23:32:06 +0200
categories: ASP.NET
---
An ASP.NET Web API that accepts bearer token as a proof of authentication is secured by validating the token they receive from the callers. To validate an `id_token` or an `access_token`, the app should validate: 
* token's signature
* claims
* nonce, as a token replay attack mitigation
* "not before" and "expiration time" claims, to verify that the ID token has not expired
* in case of access tokens, your app should also validate the issuer, the audience, and the signing tokens

These need to be validated against the values in the respective OpenID discovery document. For example, a tenant-independent version of the the OpenID discovery document for Azure AD is located at [https://login.microsoftonline.com/common/.well-known/openid-configuration](https://login.microsoftonline.com/common/.well-known/openid-configuration).

In most cases this validation is done by the built-in capabilities provided by the Azure AD/ASP.NET authentication middleware and Microsoft Identity Model Extension for .NET. When a developer generates a skeleton Web API project using Visual Studio, token validation libraries and code to carry out basic token validation is automatically generated for the project. This is usually sufficient for most scenarios. However, there are some cases when these defaults are insufficient:
* token validation parameters such as public keys are not known in advance, which is necessary when registering the authentication middleware
* restricting the API to just one or more Apps (App IDs) or tenants (issuers)
* implementing additional custom authentication schemes, such as API keys
* implementing dynamic authorization schemes that go beyond claims issued during login and fixed set of policies registered during the startup
* use external configuration during validation or make use of the HTTP execution context

# ISecurityTokenValidator
The simplest way to implement custom validation is to provide a custom [`ISecurityTokenValidator`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.identitymodel.tokens.isecuritytokenvalidator?view=azure-dotnet) implementation in the `Microsoft.AspNetCore.Authentication.JwtBearer` package. The `JwtBearerOptions` type contains a list of `ISecurityTokenValidator` so you can potentially use custom token validators, but the default is to use the built-in `JwtBearerHandler`.

{% highlight csharp %}
public class MyTokenValidator : ISecurityTokenValidator
{
    private int _maxTokenSizeInBytes = TokenValidationParameters.DefaultMaximumTokenSizeInBytes;
    private readonly JwtSecurityTokenHandler _tokenHandler;

    public MyTokenValidator(IServiceProvider serviceProvider)
    {
        _tokenHandler = new JwtSecurityTokenHandler();
    }
    public int MaximumTokenSizeInBytes
    {
        get { return _maxTokenSizeInBytes; }
        set { _maxTokenSizeInBytes = value; }
    }

    public bool CanReadToken(string securityToken)
    {
        return _tokenHandler.CanReadToken(securityToken);            
    }

    public ClaimsPrincipal ValidateToken(
        string securityToken, 
        TokenValidationParameters validationParameters,
        out SecurityToken validatedToken)
    {
        try
        {
            // Set up validation parameters or do extra checks
            return handler.ValidateToken(securityToken, validationParameters, out validatedToken);
        }
        catch (Exception ex)
        {
            validatedToken = null;
            return null;
        }
    }
}
{% endhighlight %}

We can register our custom validator together with the authentication middleware it in the service container:
{% highlight csharp %}
services.AddAuthentication(options =>
{
   // Set up authentication middleware
}).AddJwtBearer(options =>
    {
        // Remove the default validator and add our own
        options.SecurityTokenValidators.Clear();
        options.SecurityTokenValidators.Add(new MyTokenValidator());
    });
{% endhighlight %}

The biggest issue with our validator is that it is instantiated only once, making it effectively a singleton, so we can’t have dependencies on transient and scoped services. To fix this we have to either manully force a new scope from `IServiceProvider` for each token validator call, or register a custom implementation of `IPostConfigureOptions<JwtBearerOptions>` which will wire up the `MyTokenValidator` when instantiated by the service cointainer, and which call also pass any necessary dependencies.

To access the `HttpContext` we would need to register the `IHttpContextAccessor` service in the service container, and use one of the DI workarounds to pass it to the handler where we can obtain the value of the `IHttpContextAccessor.HttpContext` property.

Additionally, bypassing the authenticaion altogether in case of an e.g. unprotected endpoint requires overriding the `OnChallenge` event of the `JwtBearerEvents` class, with a custom logic that also has to be specified when registering the authentication middleware:

{% highlight csharp %}
    options.Events = new JwtBearerEvents()
    {
        OnChallenge = context =>
        {
            // Add custom logic goes here
            context.HandleResponse();
            return Task.FromResult(0);
        }
    };
{% endhighlight %}

# AuthenticationHandler
Creating your own `AuthenticationHandler` by deriving from `AuthenticationHandler<TOptions>` and implementing `HandleAuthenticateAsync` is also one option. However this is a low-level interface that requires a lot of plumbing, and should really only be used by used to implement novel authentication schemes which are meant to be used by others. In general, the outline of the process is as follows:
1. Implement the options class inheriting from `AuthenticationSchemeOptions`
2. Create the handler, inherit from `AuthenticationHandler<TOptions>`
3. Implement the constructor and `HandleAuthenticateAsync` in the handler 
4. Use the static methods of `AuthenticateResult` to create different results (None, Fail or Success)
5. Override other methods to change standard behavior
6. Register the scheme with `AddScheme<TOptions, THandler>(string, Action<TOptions>)` on the `AuthenticationBuilder`, which you get by calling `AddAuthentication` on the service collection

An example implemention for HTTP Basic authentication can be found [here](https://github.com/blowdart/idunno.Authentication/tree/master/src/idunno.Authentication.Basic).

# Custom authentication middleware
The simplest method to implement our custom JWT validation is to bypass the built-in authentication components entirely and instead implement our own authentication middleware. Our middleware should be built on top of `IApplicationBuilder` and invokable like so:
{% highlight csharp %}
app.UseMiddleware<MyAuthenticationMiddleware>();
{% endhighlight %}

The `InvokeAsync` method that we must implement in order to conform to the `IMiddleware` interface has both access to the current `HttpContext` and the transient and scoped services, which we just list as additional parameters similar to constructor injection.

{% highlight csharp %}
public class MyAuthenticationMiddleware
{
    private readonly RequestDelegate _next;

    public MyAuthenticationMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(
        HttpContext context,
        // Add can add any other injectable service like ILogger
        ILogger<MyAuthenticationMiddleware> logger
        )
    { }
}
{% endhighlight %}

Let's see how we should process the request step by step. First, we check if the required HTTP header used for authentication is present.

{% highlight csharp %}
string authHeader = context.Request.Headers["Authorization"];
if (authHeader == null)
{
    // If there is no Authorization header, perhaps we're dealing with an unprotected API, so let it through.
    logger.LogTrace("Passing through unprotected API.");
    await _next(context);
    return;
}
{% endhighlight %}

If the required header is present, the next step should be an inspection of whether the header is in the proper format, i.e having an appropriate prefix such as `Bearer` or `APIKEY`. The appropriate JWT/API key should be extracted and eventually, after a successfull validation, mapped to set of options that identify the user and provide an authoriazion context, to be used later in the request pipeline.

JWT token should be parsed manually and analyzed before invoking the validator. For example, In case of multiple issuers/audiences, it is advisable to do a manual check whether the issued JWT conforms to the required issuer origin policy. For example, if we have separate AD directories for development and production, tokens issued for the development environment should fail validation if used on the production environment, and vice versa. Sometimes however hosts are only available in production environments, and in such cases audiences should be valid in both environment. Such policies can be hard-coded because they are not changed frequently. An example implementation would be as follows:

{% highlight csharp %}
// Extract JWT token payload fields
var handler = new JwtSecurityTokenHandler();
var token = handler.ReadJwtToken(jwtToken);
var aud = token.Audiences.First();
var host = context.Request.Host.Host;
// Custom validation logic for the audience. Same can be done for the issuer
bool issuerOK = checkIssuerOrigin(aud, host);

if (!issuerOK)
{
    context.Response.StatusCode = StatusCodes.Status401Unauthorized;
    await context.Response.WriteAsync("Invalid issuer for the token!");
    return;
}
{% endhighlight %}

Prior to token validation the OpenID metadata endpoint should be constructed. Instead of hard-coding it, the preferred approach is to construct it from the payload claims referring to the issuer and the sign-in policy name `tfp`:

{% highlight csharp %}
var iss = token.Issuer;
var tfp = token.Payload["tfp"].ToString(); // Sign-in policy name
var metadataEndpoint = $"{iss}.well-known/openid-configuration?p={tfp}";
{% endhighlight %}

This metadata endpoint is then fed into the `ConfigurationManager`. A very important thing to keep in mind is to make sure that the `ConfigurationManager` initialization is done only once, either by making the instance a singleton, or by caching the instances into a static `Dictionary<string, ConfigurationManager<OpenIdConnectConfiguration>>` field, with the metadata endpoint used as a key. This is due to the fact that processing the metadata endpoint incurs a non-trivial time cost, and by caching the instance we can reuse it for validation in the subsequent requests, saving dozens of milliseconds.

{% highlight csharp %}
var cm = new ConfigurationManager<OpenIdConnectConfiguration>(metadataEndpoint,
    new OpenIdConnectConfigurationRetriever(),
    new HttpDocumentRetriever());
var discoveryDocument = await cm.GetConfigurationAsync();
var signingKeys = discoveryDocument.SigningKeys;
{% endhighlight %}

In token validation parameters we specify all the possible valid audiences and issuers:
{% highlight csharp %}
var tvp = new TokenValidationParameters()
{
    ValidateAudience = true,
    ValidAudiences = new string [] {"list all audiences"},
    ValidateIssuer = true,
    ValidIssuers = new string [] {"list all issuers"},
    RequireSignedTokens = true,
    ValidateIssuerSigningKey = true,
    ValidateLifetime = true,
    // Allow for some drift in server time. A lower value is better, two minutes or less is recommended.
    ClockSkew = TimeSpan.FromMinutes(2),
    IssuerSigningKeys = signingKeys
};
{% endhighlight %}

Finally, we validate the token in a try-catch block. The [derived types](https://docs.microsoft.com/en-us/dotnet/api/microsoft.identitymodel.tokens.securitytokenvalidationexception?view=azure-dotnet) of `SecurityTokenValidationException` that are of interest should be logged for auditing purposes.

{% highlight csharp %}
try
{
    SecurityToken validatedToken = new JwtSecurityToken();
    ClaimsPrincipal claimsPrincipal = null;
    var tokenHandler = new JwtSecurityTokenHandler();
    claimsPrincipal = tokenHandler.ValidateToken(jwtToken, tvp, out validatedToken);
    context.User = claimsPrincipal;
}
catch (SecurityTokenValidationException e)
{
    context.Response.StatusCode = StatusCodes.Status401Unauthorized;
    logger.LogInformation($"Token validation failed!");
    return;
}
catch (Exception e)
{
    context.Response.StatusCode = StatusCodes.Status500InternalServerError;
    logger.LogInformation($"Unknown exception during validation!");
    return;
}
{% endhighlight %}

If the validation is successful, we can store the acquired `ClaimsPrincipal` object into the `HttpContext.User` property to make use of it in the authorization handler later. Additionally, other checks can be performed on the user identity against external configuration sources or data stores, such as whether the user account has been disabled since the token was issued, or at all if that kind of information wasn't available during the login. Such catch-all deny policies are best kept directly in the authentication layer to make sure that no user context or options initialization occurs at all.

Usually the JWT token contains a lot more than it is required to establish the user identity; the stuff like user ID, assigned roles and whatever else is necessary to authorize access to resources. In fact, the official JWT site explicitly mentions "authorization" (in contrast to "authentication") as a usecase for JWT, assuming that the token claims have been issued against a data store containing all the necessary information to create an authorization context, which is often not the case. This kind of usage makes it impossible to alter those types of policies for a specific user after the token has been issued. The usual fix for rejecting invalidated tokes involves a data store call to verify blacklisted token, which defeats the purpose of using JWT for authorization in the first place. Lastly, it's inconvenient to force a user to relogin/refresh their token each time their access rights have been changed.

That being said, JWT token should be used exclusively for authentication. Once the identity is established the authentication middleware should fill in all the options necessary to provide an authorization context for server resources in subsequent layers. Some of these options could be extracted from the JWT token payload, some from the data store or any other external sources. Since the `InvokeAsync` has a painless access to the service container, let's first define a set of options we're interested in:

{% highlight csharp %}
public class SecurityOptions
{
    public SecurityOptions() { }
    public bool IsAuthenticated { get; set; }
    public string AccountName { get; set; }
    public AccountRole Role { get; set; }
    // Fill in the fields necessary to grant access or provide an execution context for the request
}
{% endhighlight %}

Then, we use it in the `InvokeAsync`:

{% highlight csharp %}
public async Task InvokeAsync(
    HttpContext context,
    IOptionsSnapshot<SecurityOptions> options)
{
    // after a successful validation, fill in the security options
    options.Value.IsAuthenticated = true;
    options.Value.Role = getRolesForUser(accountName);
    // ...
}
{% endhighlight %}

This way we can bypass the primitive "claims principal" collection entirely, and inject a well-typed set of options `IOptionsSnapshot<SecurityOptions>` scoped to the current request and referring to the current user's execution context into any endpoint.