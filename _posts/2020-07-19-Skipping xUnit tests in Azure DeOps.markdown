---
layout: post
title:  "Skipping xUnit tests in Azure DevOps"
date:   2020-07-19 00:02:06 +0200
categories: Azure
---

For some reasons, suppose you don't want your xUnit tests to run in Azure DevOps environment, and you want this to be determined programmatically. This can be achieved by annotating them with a custom attribute inheriting from the xUnit's `[Fact]` attribute, which will detect whether we're running on an Azure DevOps pipeline or not, and activate the `Skip` field accordingly. One of the ways to do that is by checking the existence of a predefined variable `System.DefinitionId`, which is mapped to the environment variable `SYSTEM_DEFINITIONID`.

{% highlight csharp %}
public sealed class IgnoreOnAzureDevopsFactAttribute : FactAttribute
{
    public IgnoreOnAzureDevopsFactAttribute()
    {
        if (!IsRunningOnAzureDevOps())
        {
            return;
        }

        Skip = "Ignored on Azure DevOps";
    }

    /// <summary>Determine if runtime is Azure DevOps.</summary>
    /// <returns>True if being executed in Azure DevOps, false otherwise.</returns>
    public static bool IsRunningOnAzureDevOps()
    {
        return Environment.GetEnvironmentVariable("SYSTEM_DEFINITIONID") != null;
    }
}
{% endhighlight %}

We can then use it in a unit test like so:

{% highlight csharp %}
[IgnoreOnAzureDevopsFact]
public async Task MyTestAsync()
{
    // ...
}
{% endhighlight %}