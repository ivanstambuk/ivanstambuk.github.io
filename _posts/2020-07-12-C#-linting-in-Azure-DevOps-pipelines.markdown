---
layout: post
title:  "C# linting in Azure DevOps pipelines"
date:   2020-07-12 00:02:06 +0200
categories: Azure
---

The year is 2020 and linting (aka static analysis) has made its way to modern C#. As opposed to weakly-typed languages like JavaScript and Python where it's used to catch stylistic and programming errors that are otherwise detectable only at runtime, or result from the burden of unconstrained freedom that such languages bestow upon unsuspecting engineers, in a more enterprise solutions powered by C# and its strongly-type brethren linting has made a comeback as a policy tool in the CI/CD quality gates, to be run parallel along with security and dependency scan jobs. Since Microsoft has finally embraced `EditorConfig` for all Roslyn-powered projects in VS 2019 16.3+ (and analyzer toolset 3.3+), we don't need to write ugly `.ruleset` files anymore to trigger build errors or to regulate the severity of violations. We can mix and choose many available analyzer packages like the [StyleCop](https://github.com/DotNetAnalyzers/StyleCopAnalyzers), the [Roslynator](https://github.com/JosefPihrt/Roslynator), the port of the well-known [FxCop](https://github.com/dotnet/roslyn-analyzers) for CAXXXX rules, or some obscure and specialized ones like [Meziantou](https://github.com/meziantou/Meziantou.Analyzer) and [VisualStudio.Threading](https://github.com/microsoft/vs-threading/blob/master/doc/analyzers/index.md). There are several possibilities as to how this can integrated into a build pipeline, so let's investigate!

# Treat warnings as errors
We assume `.editorconfig` is configured properly in the solution. If we want to enforce e.g. `StyleCop` on all projects in the solution we simply create a `Directory.Build.props` at the solution level like so:

{% highlight xml %}
<Project>
  <PropertyGroup>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="StyleCop.Analyzers" Version="1.1.118">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
  </ItemGroup>
</Project>
{% endhighlight %}

The `TreatWarningsAsErrors` element set to `true` will force builds to fail if any of the configured rules are violated. Alternatively, we can specify that as a parameter in the (CI) build script:

{% highlight console %}
dotnet build /p:TreatWarningsAsErrors=true
{% endhighlight %}

The `Directory.Build.props` file could also be pulled during pipeline execution from an external source if it is not present in the repository, or the pipeline can have the `TreatWarningsAsErrors` element enforced as a simple [XML transformation](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/file-transform?view=azure-devops) in case it is turned off by default in the development branch.

# dotnet format
[dotnet format](https://github.com/dotnet/format) is a global dotnet tool that can read editorconfig files, check them for violations (`--check`), and even apply fixes (`--fix-style`, `-fix-analyzers`). When running in a checking mode, it will return a non-zero exit code when violations are detected, which will in turn force the Azure DevOps pipeline to fail. We can use it in YAML like so:

{% highlight yaml %}
- script: 'dotnet tool update -g dotnet-format && dotnet format --check --verbosity diagnostic'
    displayName: 'Analyzer scan'
{% endhighlight %}  

# Build Quality Checks
[Build Quality Checks](https://marketplace.visualstudio.com/items?itemName=mspremier.BuildQualityChecks) task can be installed for free on an Azure DevOps organization. It enables various quality gates for the tasks that precede it, such as checking for warnings in the build output and failing if they occur:

{% highlight yaml %}
- task: BuildQualityChecks@7
  displayName: 'Check build quality'
  inputs:
    # ===== Warnings Policy Inputs =====
    checkWarnings: true
    warningThreshold: '0'
{% endhighlight %}

`warningFilters` option could be used to look only for e.g. StyleCop warnings using JavaScript regexes: `/##\[warning\].+SA.+:/i`, or e.g. for FxCop-style warnings `/##\[warning\].+CA.+:/i`, and ignore the junk produced by other tasks, or other types of warnings. In that case, `inclusiveFiltering` should be set to `true` as well.

# Manually checking the build output
We can write an inline PowerShell task that will capture the output of the build command, parse the number of warnings and fail the pipeline manually if any are detected. If necessary, the build command output could be captured in a separate task into a pipeline-scoped variable, so that we don't mix these two processes.

{% highlight yaml %}
- powershell: |
    # Merge all streams into stdout
    $result = dotnet build *>&1
    # reconstruct output string
    $output = $result -join [System.Environment]::NewLine
    # use regex to count the number of warnings
    $warningsCount = ([regex]::Matches($output, ": warning" )).count / 2
    # emit the collected output
    Write-Host $output

    if ($warningsCount -ne 0) {
        # terminate the pipeline
        Write-Host "##vso[task.complete result=Failed;]Quality check warnings, failing build"
    }
  displayName: 'Build and check quality'
{% endhighlight %}