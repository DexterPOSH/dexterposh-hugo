---
title: "AKS PowerShell Tip - Add Authorized Ip"
date: 2020-05-01T12:21:28+05:30
draft: true
author: "Deepak Singh Dhami"
tags: ["aks", "azure", "api", "powershell", "tip", "ip"]
---

## Background :panda_face:

Recently, I found out that there is no sane way to perform adding a public IP address to
the authorized IP address ranges using either the
[Az CLI](https://docs.microsoft.com/en-us/azure/aks/api-server-authorized-ip-ranges#update-a-clusters-api-server-authorized-ip-ranges)
or [Az.Aks](https://docs.microsoft.com/en-us/powershell/module/az.aks/?view=azps-3.8.0) PowerShell (no cmdlets available yet) module.

From the official docs it says to use  something like below format with Az CLI.

```bash
az aks update \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --api-server-authorized-ip-ranges 73.140.245.0/24
```

But it doesn't tell you how to append the IP to the range, instead you need to
supply a comma separated value of public IP addresses.

## Challenge :cloud:

Well, this is can be done by using Az CLI with PowerShell or Bash and parsing
output then generating a comma separated string and passing it back to Az CLI
 :disappointed:

## Solution :zap:

Often, when I am hit with such limitations with cmdlets or Az CLI making life
hard. I go back to using simply the 2 cmdlets provided by *Az.Resources* module.

Behold mighty!

- *Get-AzResource* - Gets the Az resource
- *Set-AzResource* - Modifies the Az resource

I ended up doing the below and creating a utility function out of it.

First, get the AKS Cluster resource. Make sure to specify the **-ExpandProperties**
switch to get back full fledged resource otherwise it returns a shallow instance.

```powershell
$ResourceGroup = "test-aks-rg"
$Name = "aksCluster001"
$IP = "110.91.234.43"
$AksCluster = Get-AzResource -ResourceGroupName $ResourceGroup -Name $Name -ExpandProperties -ErrorAction Stop
```

Once you have the resource, walk-through the properties and append the IP (+=
operator in PowerShell) to the local copy of the resource.

```powershell
$orgClusterInfo.Properties.apiServerAccessProfile.authorizedIpRanges += $Ip

```

Finally, perform a Set operation by piping the modified local resource copy to
**Set-AzResource** cmdlet.

```powershell
$orgClusterInfo | Set-AzResource -ErrorAction Stop
```

## Takeaway :fire:

Even, when there are certain utility functions not available in the Az PowerShell
module. We can rely on the *`*-Resource* cmdlets to work our way through.
