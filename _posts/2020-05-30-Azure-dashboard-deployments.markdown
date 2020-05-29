---
layout: post
title:  "Pinning App Service deployment status to Azure dashboard tiles"
date:   2020-05-30 00:02:06 +0200
categories: Azure
---
As of few years ago the Azure App Service `Deployment` pane got brushed up and renamed to  `Deployment Center`. For a brief period, both panes co-existed in the Azure portal menu until they reached feature parity. However, one particular feature was missed in the new pane: the ability to pin the deployment status to a dashboard. This feature is especially useful for providing a bird's eye view of deployments for multiple projects, when a single push to a repo can trigger a build of a number of projects, in case of a monorepo or a microservice-based architecture with shared dependencies. What's crazier, the old dashboards tiles that have the App Service deployment statuses pinned continue working, and new App Service projects can only be pinned from the `Overview` pane, and such tiles do not display the last (or ongoing) deployment status. Azure CLI provides no interface to the deployment log, and the only other way to [list deployments](https://docs.microsoft.com/en-us/rest/api/appservice/webapps/listdeployments) and get [deployment logs](https://docs.microsoft.com/en-us/rest/api/appservice/webapps/listdeploymentlog) is to poll them using the REST-based interface. (That's also a nice idea for an Azure CLI extension side project, with a log tail support.)

Fortunately, there is a workaround for this, that doesn't seem to be documented anywhere yet:
1. Go to the `Overview` pane of the desired App Service and click the pin icon to pin it to the dashboard
2. Go to your dashboard and click the `Download` button at the top bar to download your dashboard's JSON representation.
3. Open the downloaded file and find an entry for the project from the step 1. Search by the project's name, and it should appear in the `metadata.inputs[0].value` property for a tile.
4. At the project's entry, find the corresponding `type` property, and change it from `"Extension/WebsitesExtension/PartType/SingleWebsitePart"` to `"Extension/WebsitesExtension/PartType/ContinuousDeploymentInfo"`.
5. Save the changes to the file. Go back to the original dashboard and delete it, and then reupload the saved file. Voila, the deployment status for your App Service is being displayed.