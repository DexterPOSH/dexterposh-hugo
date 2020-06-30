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

## Solution 🚀

There exists the [ConvertToClass](https://github.com/dfinke/ConvertToClass) module by Doug Finke, which comes to the rescue to automatically convert a JSON object to a PowerShell class. It even has VSCode integration, check it out.

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

### Add validation attributes 🔨

We can add validation attributes to the property `$value` present inside the auto-generated class to add some quick validation rules to the properties present on the class.

> Note the name `$value` is given to the property because this is how ARM parameters file take input.

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
    # validate the prefix length
    [ValidateLength(3,15)]
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
    # validate the prefix length
    [ValidateLength(3,15)]
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
            # you can perform more checks or logging, throwing a warning here
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

Credit goes to Buce Payette's [Windows PowerShell in Action 3rd](https://livebook.manning.com/book/windows-powershell-in-action-third-edition/chapter-19/311) edition which talks a bit about this technique in brief.

In short, you can take a hashtable or PSObjects and cast them as strongly type object instance. This will become more clear when we perform this casting a bit later.

> For cast-initalization technique to work an empty constructor needs to be present in the Class definition. It is present by default if there is no constructor on the class present or you can add one explicitly.

Modify some value in the `azuredeploy.parameters.json` file to not-follow some validation logic we added e.g. putting the value for `numberofStorageAccount` as `20`, remember our `[ValidateRange(1,10)]` attribute on the property should fails. See the parameters file below:

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

Now, see the result when casting this as our `ArmParameters` class.

```powershell
[azureparameters] (Get-Content -Path $PSScriptRoot/azuredeploy.parameters.json | ConvertFrom-Json)
```

![Validation in Action](/static/007/validation.png)

Throws below error:

```error
InvalidArgument: Cannot convert value "@{$schema=https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#; contentVersion=1.0.0.0; parameters=}" to type "AzureParameters". Error: "Cannot convert value "@{storageAccountNamePrefix=; numberofStorageAccounts=; storageAccountType=; location=}" to type "Parameters". Error: "Cannot create object of type "NumberofStorageAccounts". The 20 argument is greater than the maximum allowed range of 10. Supply an argument that is less than or equal to 10 and then try the command again.""
```

Fix the value for the `numberofStorageAccounts` parameter.

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountNamePrefix": {
      "value": "azurearm6754"
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

Run the below again:

```powershell
[azureparameters] (Get-Content -Path $PSScriptRoot/azuredeploy.parameters.json | ConvertFrom-Json)
```

![Validation passes](/static/007/valid.png)

## Conclusion ✅

This is maybe a very basic way of validating input data but I read about a topic and used it to solve a problem in a unique way, which is a win for me 😎.

I think slowly embracing more Object-Oriented programming using PowerShell classes (or C#) can open up some interesting ways to solve problems in my tooling.

## Resource links 📚

* [PowerShell to C# and back](https://leanpub.com/powershell-to-csharp) - Disclaimer: co-author on this one.
* [PowerShell to C# and Back – Introduction to Classes](https://ridicurious.com/2020/06/29/powershell-to-csharp-and-back-classes/)
* [Windows PowerShell in Action, 3rd Edition](https://livebook.manning.com/book/windows-powershell-in-action-third-edition)
* [Doug Finke's ConvertToClass module](https://github.com/dfinke/ConvertToClass)
* [PSClassUtils](https://github.com/Stephanevg/PSClassUtils)
