---
layout: post
title:  "Azure DevOps pipelines and Atlassian stack"
date:   2020-07-02 00:02:06 +0200
categories: Azure
---

As it turns out, using Atlassian Bitbucket as an imported repository for CI/CD using Azure DevOps pipelines comes [with a set of problems](https://developercommunity.visualstudio.com/content/problem/553972/bug-buildrequestedfor-variables-are-incorrect-for.html) - `Build.RequestedFor` variable, among several others, is not being set by the build agent to the committer's/requestor's identity, which makes it impossible to use Azure DevOps notification system on a per-user basis. The bug is quite old, first reported back in 2017, and the ticket which resurfaced in 2019 is being closed by Microsofties as "Not a bug".

In a similar vein, Atlassian JIRA plugin for Azure DevOps pipelines, reportedly developed in a joint effort with Microsoft, doesn't work more than it does - just a cursory look at [a string of one-star reviews](https://marketplace.atlassian.com/apps/1220515/azure-pipelines-for-jira?hosting=cloud&tab=reviews) is enough to get the full picture of its abysmal state. Installing it alone requires hacking around with browser DOM to make the hidden `iframe` visible in order to complete the installation.

So if your stack is Atlassian JIRA for ALM, Bitbucket for code storage, and Azure DevOps pipelines for CI/CD, you are in a rough spot. If you were thinking of hydrating the variables during the pipeline execution, by using an external service that will transform commit ID into a set of desired credentials - it won't work, because [predefined variables are read only](https://docs.microsoft.com/en-us/azure/devops/pipelines/build/variables?view=azure-devops&tabs=yaml).

Fortunately, there is a elegant solution to the rescue - git remotes. We will create a mirror of the Bitbucket repository on the Azure Repos, and will manually push all changes to both remotes, which will eliminate the `Build.RequestedXxx` bug.

# Multiple git remotes

We will write a PowerShell script that makes sure that all of your push commands are mirrored on the Azure DevOps repository used to build the project. This script should be used only once, the first time you cloned the project from Bitbucket. Before using it, you should of course import the Bitbucket repository to Azure Repos using its Import feature. The script should be run from the local clone of the Bitbucket repository.

First, let's define some variables: the name of your Azure DevOps organization that contains your project, and the URL of your imported Azure Repos repository.

{% highlight powershell %}
$AzureDevOpsOrganization = "FILL ME"
$AzureDevOpsRepo = "FILL ME"
{% endhighlight %}

Then, let's construct the URL of the Azure DevOps remote that we will use, and fetch the URL of the Bitbucket remote

{% highlight powershell %}
$devopsUrl = "https://${AzureDevOpsOrganization}@${AzureDevOpsRepo}"
$bitBucketUrl = (git config remote.origin.url)
{% endhighlight %}

Now we're ready to set the remotes. We will create two new remotes, `devops` for Azure Repos repository and `all` which will be used for both Azure Repos and Bitbucket. For safety reasons, when running the script we first delete both so that they are not added multiple times if script is run again by accident.

{% highlight powershell %}
# delete devops and all remotes, if they exist
git remote rm devops
git remote rm all

#set default push remote to all
git remote add devops ${devopsUrl}
git remote add all ${bitBucketUrl}

git remote set-url --add --push all ${bitBucketUrl}
git remote set-url --add --push all ${devopsUrl}
git push -u all
{% endhighlight %}

Now, the `git remote -v` command in the terminal should list two push URLs for the default remote `all`. The `origin` remote should not be explicitly named in any git commands from now on.