---
layout: post
title:  "Custom and dynamic authorization in ASP.NET"
date:   2020-05-29 23:02:06 +0200
categories: ASP.NET
---
Authorization in general refers to the process that determines a level of access that a user allowed to do against a resource. By definition, authorization is orthogonal and independent from authentication. In case of many commonly used authentication schemes, such as JWT tokens, that is however usually not the case, [as we've seen in the previous post](/asp.net/2020/05/20/manually-validating-azure-ad-microsoft-identity-platform-JWT-access-tokens-in-ASP.NET). Thus, the process of authentication usually generates one or more identities for the current user, which have assigned one or more claims issued by a trusted party and which are subsequently stored in the `HttpContext.User` claims principal, or some other set of options to be reused for the authorization context such as our `SecurityOptions` object.

# ASP.NET Core authorization mechanisms

ASP.NET Core authorization mechanisms come in several flavors. Essentially, it can either be used in a simpler, role-based declarative model, and a rich policy-based model. Authorization is expressed in terms of requirements, and authorization handlers evaluate a user's claims against these requirements. Policy-based model is more general, and requirements can be expressed using the policy syntax, whereby a developer registers a policy by calling `AuthorizationOptions.AddPolicy` as part of the authorization service configuration `IServiceCollection.AddAuthorization` inside the `Startup.ConfigureServices()`. These policies are then applied using the `Policy` property of the `[Authorize]` attribute on a particular action (or the entire controller).

Claims-based authorization is by far the most popular mechanism, and the simplest: it checks the value of a claim assigned to a user protecting a resource, and allows or disallows access to a resource based upon that value. It was de facto the standard mechanism used in the pre-.NET Core era. Similar to the policy-based authorization, it's declarative in nature, and the developer used to annotate controller actions by specifying claims which the current user must possess, and optionally the value the claim must hold to access the requested resource. In ASP.NET Core this authorization mechanisms is now also expressed using policies which need to be registered during application startup, and controller methods are now annotated with named policies instead of claim names and their values.

Custom authorization handlers can also be defined. These custom handlers can either validate a single (by inheriting `<TRequirement>`) or multiple (by implementing the `IAuthorizationHandler` interface) requirements. The requirements are validated against the provided `AuthorizationHandlerContext` to determine if access is allowed. Conversely, multiple authorization handlers can be defined for a single requirement, either being invoked as a part of different policies, or to implement evaluation on a logical OR basis. These authorization handlers do provide a degree of flexibility though - it is possible to inject services such as a repository of policy rules into the constructor, which we can then use inside the `AuthorizationHandler<TRequirement,TResource>.HandleRequirementAsync` for evaluation.

Further customization is enabled by usage of custom authorization policy providers, by implementing the `IAuthorizationPolicyProvider` interface, in combination with custom parametrized `[Authorize]` attributes. These claim to solve the following issues:
* Using an external service to provide policy evaluation
* Using a large range of policies that are painful to register each
* Creating policies at runtime based on information in an external data source or determining authorization requirements dynamically through another mechanism

The `IAuthorizationPolicyProvider` interface implementations can generate named policies dynamically with the standard `AddRequirements`, `RequireClaim`, `RequireRole` or other validators, and custom authorization attributes enable application of such dynamic policies by constructing the corresponding `AuthorizeAttribute.Policy` name dynamically, on the bases of provided attribute name and its arguments. This policy name is usually just a combination of a hard-coded prefix and the supplied arguments. If necessary, policy providers have full access to injectable services through their constructors.

Imperative authorization is an alternative to policy-based approaches, and is centered around the `IAuthorizationService` service that needs to be registered in the service container and injected into actions as necessary. It has two `AuthorizeAsync` method overloads: one accepting the resource and the policy name and the other accepting the resource and a list of requirements to evaluate. This method is supposed to be invoked imperatively within actions, and the result is dependent upon the particular resource that is being authorized, such as some property of an object fetched from the data store. It's a more explicit version of doing the same thing with a `IAuthorizationHandler` implementation, without the need to duplicate access to the data store or do manual model binding. In a generalized authorization strategy with dynamically assigned security policies, this would be built around a custom validation engine with access to the data store with policy rules, and we would validate the resource properties against the contextual and object attributes.

To sum it up, ASP.NET Core team wants developers to write requirements and policies, and the authorization mechanism should validate these against the user identity and resource properties using one of the mentioned strategies. In particular, they don't want developers to use write custom "authorize" attributes where arbitrary requirements could be expressed in terms of arbitrary number or types of parameters in the attribute constructor, which could then be dynamically evaluated against policies stored in the data store or the current user's claims principal. Attribute evaluation however occurs before data binding and before execution of the page handler or action that can access the secured resource, so it has the same set of drawbacks as the policy-based mechanisms other than imperative authorization, in that it can't validate against resource properties that are only fetched within actions themselves.

Unfortunately, the policy/requirement-based approach forced upon us fails to take into consideration the typical usage scenarios in any complex enterprise application. In such scenarios, things such as user roles and security policies governing access to resources or APIs (either individually or as a group) are not fixed, and can change according to organization structure or dynamically created and applied policies. We don't want to neither write manually nor generate programmatically every imaginable policy that can be applied, when they can be created ad hoc. In the most general case, the resource itself shouldn't even know what authorization policy is being attached - it could be generated on its own according to a combination of its static (e.g. name, URI/referrer) or dynamic attributes (execution context including user identity, access type and resource's properties). It is not unusual to end up with a functional requirement to enable authorized personnel creation of policies such as "enable access to payroll processing confirmation only to those accountants who have been employed more than 2 years".

We thus ultimately want to build a form of dynamic authorization mechanism, also known as [attribute-based access control](https://en.wikipedia.org/wiki/Attribute-based_access_control) or ABAC, with the following requirements
* we don't want to preconfigure or register named policies and typed requirements in any manner
* controllers and actions can have either authorization requirements attached, or these can be fetched from external data store and evaluated, either attributively or imperatively

With respect to the second point, it's best to have a centralized external authorization solution which enforces resource access at run-time, with resources not annotated with any kind of metadata other than the one needed to evaluate such policies. This type of run-time access control model also lends itself naturally to the distributed cloud-based architectures, where each microservice could delegate the authorization process entirely, most easily at the API gateway layer (acting as an interceptor or enforcement point in ABAC parlance), or by invoking the authorization engine on its own as a separate event with the necessary attributes supplied even before hitting the invoked action.

# Filter-based authorization
Custom filters in ASP.NET Core allow code to be run before or after specific stages in the request processing pipeline. They elegantly handle cross-cutting concerns such as authorization. Custom authorization filters require a custom authorization framework, and their usage is in generally discouraged in favor of policies/requirements. You can't use constructor injection with attributes using the built-in ASP.NET service container, but you can do property injection with third-party IoC containers such as Autofac. The reason why is straightforward: attribute parameters are evaluated at compile-time, so they have to be compile-time constants. There are also other workarounds such as abusing the service locator (anti)pattern `RequestServices.GetService`.

Several types of built-in filters that support constructor dependencies are provided from built-in service container: `ServiceFilterAttribute`, `TypeFilterAttribute` and `IFilterFactory`, each with its own set of problems. With `ServiceFilter` it is not possible to specify any configuration for the filter, other than any configuation registered with the container in terms of our own custom `IActionFilter`-derived type. `TypeFilter` is very similar to `ServiceFilter` but has two notable differences: the type being resolved does not need to be registered with the service container (but any dependencies do); and arguments can be supplied via the `Arguments` property which are used when constructing the filter. The main issue is the lack of type safety with this property, which is declared as an `new object[]`. Finally, `IFilterFactory` is the most general interface on top of which both `ServiceFilter` and `TypeFilter` are built, and which provides benefits of both worlds: IoC-based dependencies _and_ custom configuration of the attribute.

# TypeFilterAttribute

The `TypeFilter` provides the most convenient solution for the most common need of simply asserting that a given controller or action requires a given claim type. We simply want to slap our custom attribute onto actions or controllers without registering an authorization scheme in the service container. Instead of using claims, we will reuse the `SecurityOptions` options introduced  [in the previous post](/asp.net/2020/05/20/manually-validating-azure-ad-microsoft-identity-platform-JWT-access-tokens-in-ASP.NET) to authorize the access against the specified arguments. First, let's create a custom type parametrized by an authorization type (e.g. claim name) and value (e.g. claim value), which will in practice be constants/enumerations.

{% highlight csharp %}
public class MyAuthorizeAttribute : TypeFilterAttribute
{
    public MyAuthorizeAttribute(string authType = "", string authValue = "")
        : base(typeof(MyRequirementFilter))
    {
        Arguments = new object[] { new Tuple<string, string>(authType, authValue) };
    }
}
{% endhighlight %}

The custom authorization filter can have injectable dependencies in its constructor, other than the elements of the `Arguments` array, which should be stored in the filter instance together with the user's security options:

{% highlight csharp %}
public class ClaimRequirementFilter : IAuthorizationFilter
{
    private readonly string _authType;
    private readonly string _authValue;
    private readonly IOptionsSnapshot<SecurityOptions> _options;

    public ClaimRequirementFilter(
        Tuple<string, string> authRequest, 
        // Add additional dependencies necessary for validation, such as data store or externalized
        // authorization engine
        IOptionsSnapshot<SecurityOptions> options)
    {
        _authType = authRequest.Item1;
        _authValue = authRequest.Item2;
        _options = options;
    }
}
{% endhighlight %}

The filter logic should be implemented in the `OnAuthorization` method which operates on the current `AuthorizationFilterContext`. First, we need to check whether the method has the `[AllowAnonymous]` attribute applied. This is a bit tricky to do manually, since with new endpoint routing added in .NET Core 2.2, MVC does not add `AllowAnonymousFilters` for `AllowAnonymousAttributes` that were discovered on controllers and actions. Fortunately, [the presence of IAllowAnonymous can be checked in endpoint metadata](https://github.com/dotnet/aspnetcore/blob/24784d0681fb3a2167762adf7abf164d22c76bce/src/Mvc/Mvc.Core/src/Authorization/AuthorizeFilter.cs#L221), so let's define a helper method that does so: 

{% highlight csharp %}
private static bool hasAllowAnonymous(AuthorizationFilterContext context)
{
    var filters = context.Filters;
    for (var i = 0; i < filters.Count; i++)
    {
        if (filters[i] is IAllowAnonymousFilter)
        {
            return true;
        }
    }

    // When doing endpoint routing, MVC does not add AllowAnonymousFilters for 
    // AllowAnonymousAttributes that were discovered on controllers and actions. To maintain 
    // compatibility with 2.x, we'll check for the presence of IAllowAnonymous in endpoint metadata.
    var endpoint = context.HttpContext.GetEndpoint();
    if (endpoint?.Metadata?.GetMetadata<IAllowAnonymous>() != null)
    {
        return true;
    }

    return false;
}
{% endhighlight %}

Now we can start implementing the `OnAuthorization` handler. In case of anonymous methods, we want we just pass the filter pipeline onto the next handler:

{% highlight csharp %}
public void OnAuthorization(AuthorizationFilterContext context)
{
    if (context == null)
    {
        throw new ArgumentNullException(nameof(context));
    }

    // Allow Anonymous skips all authorization
    if (hasAllowAnonymous(context))
    {
        return;
    }
}
{% endhighlight %}

Next, we want to check if the user is authenticated. We can either check the `context.HttpContext.User.Identity.IsAuthenticated` or the supplied security options, and short-circuit the pipeline if the user is not authenticated properly. As it turns out, the authorization handler can only send back a status code and not a message, and to return an error message together with 401 response we need to create a custom type derived from `JsonResult`:

{% highlight csharp %}
public class CustomError
{
    public string Error { get; }

    public CustomError(string message)
    {
        Error = message;
    }
}

public class CustomUnauthorizedResult : JsonResult
{
    public CustomUnauthorizedResult(int statusCode, string message)
        : base(new CustomError(message))
    {
        StatusCode = statusCode;
    }
}
{% endhighlight %}

Now we can both return 401 and a status message in case of unauthenticated users:

{% highlight csharp %}
    // Alternatively, validate the context.HttpContext.User.Identity.IsAuthenticated property
    if (!_options.Value.IsAuthenticated)
    {
        context.Result = new CustomUnauthorizedResult(StatusCodes.Status401Unauthorized,
            "Authorization has been denied for this request.");
    }
{% endhighlight %}

`authValue` argument could be a string encoding a Boolean composed of a number of operands denoting individual authorization values that need to be validated independently. For example, it could be used as:

{% highlight csharp %}
[MyAuthorize(ClaimTypes.Role, RoleClaimValues.GlobalAdmin + "|" + RoleClaimValues.CompanyAdmin)]
{% endhighlight %}

In such usages, a parser needs to be implemented to process the encoded conditional. In either case, the authorization handler logic is reduced to method of the folllowing type:

{% highlight csharp %}
private static bool checkAuthorization(SecurityOptions secOptions, string authType, string authValue)
{
    switch (authType)
    {
        case ClaimTypes.Role:
            var desiredRole = (AccountRole)Enum.Parse(typeof(AccountRole), authValue);
            return secOptions.Role == desiredRole;
        // Add custom logic here
        default:
            throw new Exception($"MyAuthorize: Unknown authorization type {authType}");
    }
}
{% endhighlight %}

As it has been said already, it's best to inject any kind of dynamic permissions assigned to the user identity during the authentication, as a part of the claims principal/security options, so that they can injected and reused in other middleware components, actions and filters, rather than accessing the data store directly in the filter. With dynamic policies this is not the case, as these would be created against `ActionDescriptor` or `ControllerDescriptor` properties such as their names or paths, or some "dumb" descriptive attributes such as `[MyDynamicAuthorization("PayrollProcessing")]` whose arguments can be fed into the authorization engine to run against the dynamic policies. In case of an externalized authorization engine conformant to some ABAC standard (e.g. the open-source [AuthzForce](https://github.com/authzforce) or some commercial variant), that would of course be a mere blocking call.