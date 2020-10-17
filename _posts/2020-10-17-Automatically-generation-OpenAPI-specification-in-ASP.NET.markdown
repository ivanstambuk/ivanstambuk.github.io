---
layout: post
title:  "Automatically generating OpenAPI specification in ASP.NET Core"
date:   2020-10-17 00:02:06 +0200
categories: Azure
---

In ASP.NET Core it is customary to have the OpenAPI specs generated alongside Swagger UI using the popular `Swashbuckle.AspNetCore` package. Beside the nice UI which enable interactive testing of the endpoints, Swashbuckle also generates the OpenAPI specs on the `/swagger/v1/swagger.json` endpoint by default. This endpoint is only available when the application is run, and cannot be retrieved otherwise because it is generated dynamically.

Sometimes, however, it is useful to have the OAS (OpenAPI Specification) specs generated during the build-time for the following reasons:
* you can share it with colleagues who will consume them for UI development by automatically generating appropriate REST clients using tools like [autorest](https://github.com/Azure/autorest) or [swagger-codegen](https://swagger.io/tools/swagger-codegen/). This should be automated post-build on file change with automatic PRs to API consumers.
* OAS specs can be imported into tools like `Postman` for functional testing. This should also be automated in order to keep the two in sync.
* you can run the specs through an API linter (custom or corporate) as a post-build task
* you have a git history of how the code changes have been simultaneously reflected in the API specs. This is often not an obvious correlation, because OAS specs are generated from both static code annotations such as comments or attributes, as well as dynamically by using runtime extensibility features of the Swagger library.

As the first step, install on your dev machine/pipeline the Swashbuckle CLI library:

{% highlight console %}
dotnet tool install -g --version 5.6.2 Swashbuckle.AspNetCore.Cli
{% endhighlight %}

Now we have the `swagger` tool available globally in the CLI.

Then, create a `openapi_build.bat` file in your API project directory (e.g. `src\API`) with the following contents:

{% highlight console %}
swagger tofile --output OAS\openapi.json %1\%2.dll v1
swagger tofile --output OAS\openapi.yaml --yaml %1\%2.dll v1
{% endhighlight %}

Then, create the `OAS` directory in the same folder which will contain OAS specifications in both JSON and YAML, because `swagger` tool can't create it on its own.

Then, in your API project's `.csproj` file, add the following:

{% highlight xml %}
<Target Name="OASTarget" AfterTarget="Build" Condition="$(Configuration)=='Debug'">
	<Exec Command="openapi_build.bat $(OutputPath) $(AssemblyName)"/>
</Target>
{% endhighlight %}

This will cause your project build operation (also when building a solution that contains the project) in `Debug` configuration to create two files in your API project's directory:
* ` OAS\openapi.json` containing OAS specs in JSON format
* ` OAS\openapi.yaml` containing OAS specs in YAML format