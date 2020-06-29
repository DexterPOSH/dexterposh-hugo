---
title: "PowerShell Classes - Validating input data"
date: 2020-06-29T19:42:59+05:30
image: "/static/007/class1.png"
tags: ["azure","powershell", "tip"]
categories: ["powershell"]
include_toc: true
highlightjslanguages: ["powershell"]
---

## Origin❓

(shameless plug, alert!) 🙃

Recently, I was discussing with my colleague about this new [book](https://leanpub.com/powershell-to-csharp) I am co-authoring (with [Prateek](https://twitter.com/singhprateik)) about why to learn .NET to be a better PowerShell programmer and upon further discussion we pondered some interesting ways to use PowerShell classes.

## Brain-storming 🤔

All was lost, until we had another quick conversation about how to validate ARM templates.
Well, I suggested to write Pester tests to check the input being passed and perform `Test-AzDeployment` for ARM templates.

Another idea that popped up in my mind was what if we write a PowerShell class to model the ARM parameters file and use that to validate the ARM template parameter inputs.

## Solution ✅

There exists the [ConvertToClass](https://github.com/dfinke/ConvertToClass) module by Doug Finke, whic comes to the rescue to automatically convert a JSON object to a PowerShell class. It even has VSCode integration, check it out.

Let's start by taking a sample `azuredeploy.parameters.json` file.

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountNamePrefix": {
      "value": "Azurearm6754"
    },
    "numberofStorageAccounts": {
      "value": 2
    },
    "storageAccountType": {
      "value": "Standard_LRS"
    },
    "location": {
      "value": "westus2"
    }
  }
}
```

### Convert Json to Class ☯

Let's run the `ConverTo-Class` function in the ConvertToClass module against this `azuredeploy.parameters.json` file.

```powershell
ConvertTo-Class -Target (Get-Content ./azuredeploy.parameters.json -Raw)
```

![alt](/static/007/class1.png)

Above command generates below content:

```powershell
class RootObject1 {
    [string]$$schema
    [string]$contentVersion
    [parameters]$parameters
}

class parameters1 {
    [storageAccountNamePrefix]$storageAccountNamePrefix
    [numberofStorageAccounts]$numberofStorageAccounts
    [storageAccountType]$storageAccountType
    [location]$location
}

class storageAccountNamePrefix1 {
    [string]$value
}

class numberofStorageAccounts1 {
    [int]$value
}

class storageAccountType1 {
    [string]$value
}

class location1 {
    [string]$value
}
```

Let's rename the above classes like below:

* `$$schema` property on `RootObject1` to `${$schema}`, done to escape `$` char in property name
* `RootObject1` to `AzureParameters`
* `parameters1` to `Parameters`
* `storageAccountNamePrefix1` to `StorageAccountType`
* `numberofStorageAccounts1` to `StorageAccountType`
* `storageAccountType1` to `StorageAccountType`
* `location1` to `Location`

### Add validation attributes ⌚

We can add validation attributes to the `$value` inside the auto-generated class to add some quick validation rules to the properties present on the class.

```powershell
class AzureParameters {
    [string]${$schema}
    [string]$contentVersion
    [Parameters]$parameters
}

class Parameters {
    [StorageAccountNamePrefix]$storageAccountNamePrefix
    [NumberofStorageAccounts]$numberofStorageAccounts
    [StorageAccountType]$storageAccountType
    [Location]$location
}

class StorageAccountNamePrefix {
    # use the regex to validate lowercase letters and number in the name
    [ValidatePattern("^([a-z]|[0-9])+$")]
    [ValidateLength(3,24)]
    [string]$value
}

class StorageAccountNamePrefix {
    # restricting the min=1 and max=10 storage accounts that one can request
    [ValidateRange(1,10)]
    [int]$value
}

class StorageAccountType {
    # Restricting the StorageAccount SKUs
    [ValidateSet('Standard_LRS', 'Standard_GRS', 'Standard_ZRS')]
    [string]$value
}

class Location {
    # Restricting the locations
    [ValidateSet('westus2', 'northcentralus')]
    [string]$value
}
```

### Add logic inside Empty Constructor ⌛

We can add one more trick to the bag to add an empty constructor explicitly (this is present when no constructor exists) and put some more validation logic if the current validate attributes doesn' suit the needs.

```powershell
class AzureParameters {
    [string]${$schema}
    [string]$contentVersion
    [Parameters]$parameters
}

class Parameters {
    [StorageAccountNamePrefix]$storageAccountNamePrefix
    [NumberofStorageAccounts]$numberofStorageAccounts
    [StorageAccountType]$storageAccountType
    [Location]$location
}

class StorageAccountNamePrefix {
    # use the regex to validate lowercase letters and number in the name
    [ValidateLength(3,24)]
    [string]$value
}

class NumberofStorageAccounts {
    # restricting the min=1 and max=10 storage accounts that one can request
    [ValidateRange(1,10)]
    [int]$value
}

class StorageAccountType {
    # Restricting the StorageAccount SKUs
    [ValidateSet('Standard_LRS', 'Standard_GRS', 'Standard_ZRS')]
    [string]$value

    # Add some more validation logic inside the empty constructor
    storageAccountType() {
        # Perform some validation on the property
        if ($this.value -ne 'Standard_ZRS') {
            Write-Warning -Message "No zone redundancy"
        }
    }
}

class Location {
    # Restricting the locations
    [ValidateSet('westus2', 'northcentralus')]
    [string]$value
}
```

> Why does this has to be an empty constructor?
The answer is that we will be a using a trick with how we create an instance of the class in next section

### Secret-Sauce : Cast-Initialization 🍲

I read Buce Payette's [Windows PowerShell in Action 3rd](https://livebook.manning.com/book/windows-powershell-in-action-third-edition/chapter-19/311) edition which talks a bit about this technique in a brief.

In short, you can take schemaless data in two forms e.g. hashTable or PSObjects and convert them to strongly typed object instances.

> For cast-initalization technique to work an empty constructor needs to be present in the Class definition.

Modify some value in the `azuredeploy.parameters.json` file to not-follow some validation logic we added like below:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountNamePrefix": {
      "value": "azurearm6754"
    },
    "numberofStorageAccounts": {
      "value": 20
    },
    "storageAccountType": {
      "value": "Standard_LRS"
    },
    "location": {
      "value": "westus2"
    }
  }
}
```

```powershell
[azureparameters] (Get-Content -Path $PSScriptRoot/azuredeploy.parameters.json | ConvertFrom-Json)
```

![Validation in Action](/static/007/validation.png)

Now, see the result when casting this as our `ArmParameters` class.

```error
InvalidArgument: Cannot convert value "@{$schema=https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#; contentVersion=1.0.0.0; parameters=}" to type "AzureParameters". Error: "Cannot convert value "@{storageAccountNamePrefix=; numberofStorageAccounts=; storageAccountType=; location=}" to type "Parameters". Error: "Cannot create object of type "NumberofStorageAccounts". The 20 argument is greater than the maximum allowed range of 10. Supply an argument that is less than or equal to 10 and then try the command again.""
```

## Resource links 📚

* [PowerShell to C# and back](https://leanpub.com/powershell-to-csharp) - Disclaimer: co-author on this one.
* [Windows PowerShell in Action, 3rd Edition](https://livebook.manning.com/book/windows-powershell-in-action-third-edition)
* [Doug Finke's ConverToClass module](https://github.com/dfinke/ConvertToClass)
* [PSClassUtils](https://github.com/Stephanevg/PSClassUtils)
