---
title: "Azure DevOps Tip - Find private APIs"
date: 2020-04-14T15:06:22+05:30
author: "Deepak Singh Dhami"
tags: ["azdo", "tip", "api"]
categories: ["azuredevops"]
---

## Problem

Often working with Azure DevOps, I hit a wall trying to automate some tasks but
there are no REST APIs made public yet.

It was one of those task of automating creation of Environments in multi-stage
YAML based pipelines in AzDO.

![Azure DevOps environments](/static/002/env.png)

Quick research reveals that this has been requested in [uservoice](https://developercommunity.visualstudio.com/content/problem/820737/rest-apis-for-environments-and-its-resources-multi.html) (please upvote).
Let's see one of the very simple ways to discover some of these APIs.

## Developers Tools to rescue

Using your browser's developers tools you can actually inspect the HTTP requests
being made while performing an action in the web portal.

Let's do this.

![Developer tools](/static/002/devnetwork.png)

Let's click on the "Create Environment" button, fill out some dummy values,
hit create and keep an eye on the network tab in the developer tools.

![Create env watch network](/static/002/envcreatenetwork.png)

We see some activity, it might take you some time to walk through what happened but in this case the top activity named "environments" has the required details.

See below and note the URL & method used:

![Analyze request sent](/static/002/analyzerequest.png)

Also, make note of the Json request in the payload.

![Request payload](/static/002/requestpayload.png)

That's mostly it, fire up postman/PowerShell to make the API call to test this.

## Invoke-RestMethod in Pwsh

[Generate a Personal Access Token](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=preview-page) in AzDO, typically start with a short lived PAT token with full access and then nail down on the specific permissions you need.

Below is the code snippet, I used with AzDO to hit the REST API endpoint:

```powershell
$url = 'https://dev.azure.com/ddhami/BITPro.AzDeploy/_apis/distributedtask/environments'
$cred = Get-Credential -UserName 'vsts' -Message 'Enter AzDO Personal Access Token with privs to create env'
$encodedValue = [Convert]::ToBase64String(
            [Text.Encoding]::ASCII.GetBytes(
                ("{0}:{1}" -f '', $cred.GetNetworkCredential().Password)
            )
        )

$body = @{
  name = 'test-pwsh-env';
  description = 'test environment from APR'
} | ConvertTo-Json
$headers = @{
  Accept = 'application/json;';
  Authorization = "Basic {0}" -f $encodedValue
}

Invoke-RestMethod -Method POST -Uri $url -Body $body -Headers $headers -ContentType 'application/json'
```

But when I execute the above code, it gives me an error.

![Error thrown](/static/002/pwsherror.png)

The error thrown is below.

```json
{
    "$id": "1",
    "innerException": null,
    "message": "No api-version was supplied for the \"POST\" request. The version must be supplied either as part of the Accept header (e.g. \"application/json; api-version=1.0\") or as a query parameter (e.g. \"?api-version=1.0\").",
    "typeName": "Microsoft.VisualStudio.Services.WebApi.VssVersionNotSpecifiedException, Microsoft.VisualStudio.Services.WebApi",
    "typeKey": "VssVersionNotSpecifiedException",
    "errorCode": 0,
    "eventId": 3000
}
```

Read the error message, it explains that the api-version is missing. Also, looking back at the capture and see where the api-version was specified.

![API Version in header](/static/002/apiversion.png)

Let's make the same change in our code snippet to include API version in the
header.

```powershell {hl_lines=[14], linenostart=1}
$url = 'https://dev.azure.com/ddhami/BITPro.AzDeploy/_apis/distributedtask/environments'
$cred = Get-Credential -UserName 'vsts' -Message 'Enter AzDO Personal Access Token with privs to create env'
$encodedValue = [Convert]::ToBase64String(
            [Text.Encoding]::ASCII.GetBytes(
                ("{0}:{1}" -f '', $cred.GetNetworkCredential().Password)
            )
        )

$body = @{
  name = 'test-pwsh-env';
  description = 'test environment from APR'
} | ConvertTo-Json
$headers = @{
  Accept = 'application/json;api-version=5.0-preview.1';
  Authorization = "Basic {0}" -f $encodedValue
}

Invoke-RestMethod -Method POST -Uri $url -Body $body -Headers $headers -ContentType = 'application/json'
```

Here I go, this finally works and using the similar API endpoint I can fetch the environments as well (GET request).

![Env created success](/static/002/envsuccess.png)