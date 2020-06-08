---
title: "ARM templates - iterate & deploy resource"
date: 2020-05-26T15:18:35+05:30
highlight: false
image: "/static/006/transformParams.png"
categories: ["Azure"]
tags: ["azure", "arm", "iac", "cloud"]
---
 
## Background 🧐

I like ARM templates, I use it a lot to deploy Azure cloud resources but as all things it has some pain points associated with it. In this post, let's see how you can iterate over based on certain logic and deploy multiple resources using linked templates.

As it stands out this logic of iterating over and deploying multiple instances of a resource tripped me a lot in the beginning.

## Walkthrough 🏃

Let's work through the whole process of writing an ARM template which deploys multiple resources.

Github Repository - [ArmTemplateLoopExample](https://github.com/DexterPOSH/ArmTemplateLoopExample)

> This is a simple post demonstrating looping logic I often use, feel free to sprinkle your own best practices & modifications on top e.g. storing templates in a private Cloud blob container, adding more parameters, names etc.

### Scenario

Let's take a scenario of deploying many storage accounts based on the user input.

Ideally, if you're in this situation you should write 2 templates and utilize ARM linked templates to deploy them because it becomes too cumbersome to maintain a single ARM template to deploy a resource and loop over user-input and deploy multiple iterations of that resource. Trust me this is coming from experience 😉

So, let's start by creating 2 templates, I am going to use GitHub repository here for storing those but you can use a Cloud blob store account as well.

Below is how my project directory layout looks like.

```output
.
├── azuredeploy.json
└── linkedTemplate
   └── storageaccount.json
```

### Author linked template

First thing to do when you're writing an ARM template is to make sure you understand that component properly, how it works, best practices while using that Azure component etc. Why? you might be wondering because ARM templates is how you deploy your Azure cloud infrastructure and it would be as good as you make your ARM templates, they're called blueprints for your Azure resources for this reason.

But at the same time start small and head over to [azure-quickstart-templates](https://github.com/Azure/azure-quickstart-templates) repository to get some samples.

I found out that the template stored here in this [101-storage-account-create](https://github.com/Azure/azure-quickstart-templates/blob/master/101-storage-account-create/azuredeploy.json) example is good enough for me. So, let me copypasta ✍ this and place the content inside my `linkedtemplate\storageaccount.json` file.

So, we have a starting point which can deploy a single storage account for us, but you would notice on closer inspection that this `storageaccount.json` template doesn't take storageAccountName as a parameter but generates it in the variables section.

Let's quickly modify it. Changes made:

* add parameter `storageAccountName`
* remove variable `storageAccountName`
* change `[variables('storageAccountName')]` references to `[parameters('storageAccountName')]`

```json
{
   "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
   "contentVersion": "1.0.0.0",
   "parameters": {
     "storageAccountName": { // added the storageAccountName parameter
       "type": "string"
     },
     "storageAccountType": {
       "type": "string",
       "defaultValue": "Standard_LRS",
       "allowedValues": [
         "Standard_LRS",
         "Standard_GRS",
         "Standard_ZRS",
         "Premium_LRS"
       ],
       "metadata": {
         "description": "Storage Account type"
       }
     },
     "location": {
       "type": "string",
       "defaultValue": "[resourceGroup().location]",
       "metadata": {
         "description": "Location for all resources."
       }
     }
   },
   "variables": {
     // removed the storageAccountName variable from here
   },
   "resources": [
     {
       "type": "Microsoft.Storage/storageAccounts",
       "apiVersion": "2019-04-01",
       "name": "[parameters('storageAccountName')]",
       "location": "[parameters('location')]",
       "sku": {
         "name": "[parameters('storageAccountType')]"
       },
       "kind": "StorageV2",
       "properties": {}
     }
   ],
   "outputs": {
     "storageAccountName": {
       "type": "string",
       "value": "[parameters('storageAccountName')]"
     }
   }
 }
```

### Author the stitching logic

Moving on to the logic of consolidating user input and then looping over and deploying a storage account multiple times.

The gist is that we have to do below:

* Use variable iteration to create an array of objects based on our `numberofStorageAccounts` parameter value

* Use resource iteration later with a linked template deployment and index into the array created above for parameter values.

> Don't worry if this is a bit daunting. It was for me the first time.

#### Adding barebone template

Let's start by creating a blank ARM template. Open the `azuredeploy.json` in VSCode. Key in `arm` and it would give you a snippet dropdown, select the first one for targeting a Resource group deployment.

![alt](/static/006/arm_snippets.png)

So, we get this.

```json
{
 "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
 "contentVersion": "1.0.0.0",
 "parameters": {},
 "functions": [],
 "variables": {},
 "resources": [],
 "outputs": {}
}
```

##### Add parameters

Time to add in some parameters to our `azuredeploy.json` which is end-user facing. So you need to take input in this one from the user (which could be yourself as well) and then pass those over to the linkedtemplate.

* storageAccountNamePrefix - prefix for the storage accounts to be deployed. Length 5-10.
* numberofStorageAccounts - integer representing how many storage accounts to deploy. [Default - 1, Min 1, Max 10.
* storageAccountType - Type of the storage accounts, predefined allowed values. Default - Standard_GRS.
* location - location for the storage accounts. Default is RG location.

Below is how the `parameters` object looks now.

```json
"parameters": {
  "storageAccountNamePrefix": {
    "type": "string",
    "minLength": "5",
    "maxLength": "10"
  },
  "numberofStorageAccounts": {
    "type": "int",
    "defaultValue": 1,
    "minValue": "1",
    "maxValue": "10"
  },
  "storageAccountType": {
    "type": "string",
    "defaultValue": "Standard_GRS",
    "allowedValues": [
      "Standard_LRS",
      "Standard_GRS",
      "Standard_ZRS",
      "Premium_LRS"
    ]
  },
  "location": {
    "type": "string",
    "defaultValue": "[resourceGroup().location]",
    "metadata": {
      "description": "Location for all resources."
    }
  }
}
```

##### Add the variables (iteration logic)

I typically like to use variables a lot for transforming the input parameters and then using these variables later in the resources because it makes it easier in future to just modify these variables at once place.

Use the concept of [variable iteration](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/copy-variables) in ARM templates.

We use the above concept to do the below

* Create a variable named `_deployMultipleStorageAccounts` (sort of a convention I follow to name these variables used later in deployment to preceed with `_deploy`)
* Use the parameter `numberofStorageAccounts` to loop over that many times
* Use the `input` property in the copy loop object to generate an object containing properties which will be mapped one to one with the linked template storage.

I prefer one to one mapping between the properties inside the `input` in the copy loop to the parameters of the linked template. It makes it easier to index into them and specify them (you'll see later).

Below is a gist of what I added in the variables property.

```json
"variables": {
  "copy": [
    {
      "name": "_deployMultipleStorageAccounts", // variable name used later with resources
      "count": "[parameters('numberofStorageAccounts')]", // loop over numberofStorageAccounts time
      "input": { // input object con
        "storageAccountType": "[parameters('storageAccountType')]",
        "location": "[parameters('location')]",
        "storageAccountName": "[
          concat(
            parameters('storageAccountNamePrefix'),
            uniqueString(resourceGroup().id),
            copyIndex('_deployMultipleStorageAccounts')
          )
        ]"
      }
    }
  ]
}
```

In the outputs section we added a `variables` property which essentially displays the value for the variable `_deployMultipleStorageAccounts`. This can be used later with a trick to see what values go inside this variable.


```json
"outputs": {
  "variables": {
    "type": "object",
    "value": {
      "_deployMultipleStorageAccounts": "[variables('_deployMultipleStorageAccounts')]"
    }
  }
}
```

Also, I added a `variables` property in the output which is used to display what goes inside this variable once it is run by ARM API.

![alt](/static/006/transformParams.png)

From a data-view point above creates a variable named `_deployMultipleStorageAccounts` which is an array of Json objects.

If we assume the `parameters('numberofStorageAccounts')` is 2, `parameters('storageAccountType')` is *Standard_GRS* and `parameters('storageAccountType')` is *SouthEastAsia*, then it creates an array like below:

```json
[
  {
    "storageAccountName": "<generatedValuebyARM>",
    "storageAccountType": "Standard_GRS",
    "loation": "SouthEastAsia",
  },
  {
    "storageAccountName": "<generatedValuebyARM>",
    "storageAccountType": "Standard_GRS",
    "loation": "SouthEastAsia",
  }
]
```

#### Add the deployment resource

Now, you already know we have the linked template to deploy a single storage account. So, we just need to invoke/call that template multiple times and pass in paramters.

This is done by an ARM template technique called as ARM template linked templates. Read about it more [here](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/linked-templates#linked-template)

> Follow a [Tutorial](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/deployment-tutorial-linked-template?tabs=azure-powershell#create-a-linked-template) to deploy a linked template, if this is the firs time you're hearing about this.

In short, within our `azuredeploy.json` template we need to use a resource of type `Microsoft.Resources/deployments` to link to our `storageaccount.json` template and inside this resource we need to use another concept termed as [*Resource iteration*](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/copy-resources)

Let's add it.

This is how my resources array property looks like.

```json
"resources": [
  {
    "type": "Microsoft.Resources/deployments",
    "apiVersion": "2019-10-01",
    "condition": false,
    "name": "[concat('deploy-linkedStorageTemplate', copyIndex())]",
    "copy": {
      "count": "[parameters('numberofStorageAccounts')]",
      "name": "_loopToDeployStorageAccounts",
      "mode": "Parallel"
    },
    "properties": {
      "mode": "Incremental",
      "templateLink": {
        "contentVersion": "1.0.0.0",
        "uri": "https://raw.githubusercontent.com/DexterPOSH/ArmTemplateLoopExample/master/linkedtemplate/storageaccount.json"
      },
      "parameters": {
        "storageAccountName": {
          "value": "[variables('_deployMultipleStorageAccounts')[copyIndex()].storageAccountName]"
        },
        "storageAccountType": {
          "value": "[variables('_deployMultipleStorageAccounts')[copyIndex()].storageAccountType]"
        },
        "location": {
          "value": "[variables('_deployMultipleStorageAccounts')[copyIndex()].location]"
        }
      }
    }
  }
]
```

I know it's a handful but below is a breakdown of major things it does.

* Uses `Microsoft.Resources/deployments` resource type to deploy another template.

> Note we generate a unique name for the deployment by concating the `copyIndex()`
> The `condition` property is set to `false`

```json
{
  "type": "Microsoft.Resources/deployments",
  "apiVersion": "2019-10-01",
  "condition": false,
  "name": "[concat('deploy-linkedStorageTemplate', copyIndex())]"
  //<-- skipped below properties -->
}
```



* Uses the `uri` of the raw template link for the `storageaccount.json` in the GitHub repository.

```json
"templateLink": {
  "contentVersion": "1.0.0.0",
  "uri": "https://raw.githubusercontent.com/DexterPOSH/ArmTemplateLoopExample/master/linkedtemplate/storageaccount.json"
}
```

* Uses resource iteration by using `copy` property and using the `parameters('numberofStorageAccounts')` as the value for count, which means it loops over this resource this many times. Also, gives this copy loop a friendly name `_loopToDeployStorageAccounts`.

```json
"copy": {
  "count": "[parameters('numberofStorageAccounts')]",
  "name": "_loopToDeployStorageAccounts"
}
```

* Passes on the parameters to this linked template by indexing into the variable `_deployMultipleStorageAccounts`

```json
"storageAccountName": {
  "value": "[variables('_deployMultipleStorageAccounts')[copyIndex()].storageAccountName]"
}
```

#### Dry run - Verify Variable iteration logic

Based on my experience with this approach of looping over, we can most of the time validate what is inside the variable created for looping to verify it will work.

Remember the `condition` is set to `false` for our linked template deployment resource, which means when we submit this ARM template for deployment it won't trigger it but however the `azuredeploy.json` will be processed and we will get the output back which contains the `variables` property.

Let's create an ARM template parameters file, with the new release of the ARM tools VSCode extension, it is natively possible to generate these parameters file. Read more in the release notes [here](https://marketplace.visualstudio.com/items?itemName=msazurermtools.azurerm-vscode-tools).

Click on *Select Parameter File...* (at the bottom) > *New* > *All parameters* > Save it. Open it and fill the values.

![Generate ARM parameters file](/static/006/generateParam.png)

This is how it looks after adding values.

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
      "value": "Standard_GRS"
    },
    "location": {
      "value": "southindia"
    }
  }
}
```

Deploy it using Az PowerShell module cmdelt

```powershell
$rg =  "test_arm_rg"
New-AzResourceGroupDeployment -TemplateFile ./azuredeploy.json -TemplateParameterFile ./azuredeploy.parameters.json -ResourceGroupName $rg
```

```output
DeploymentName          : azuredeploy
ResourceGroupName       : test_arm_rg
ProvisioningState       : Succeeded
Timestamp               : 06/07/2020 14:24:21
Mode                    : Incremental
TemplateLink            :
Parameters              :
                          Name                        Type                       Value
                          ==========================  =========================  ==========
                          storageAccountNamePrefix    String                     azurearm6754
                          numberofStorageAccounts     Int                        2
                          storageAccountType          String                     Standard_GRS
                          location                    String                     southindia

Outputs                 :
                          Name             Type                       Value
                          ===============  =========================  ==========
                          variables        Object                     {
                            "_deployMultipleStorageAccounts": [
                              {
                                "storageAccountType": "Standard_GRS",
                                "location": "southindia",
                                "storageAccountName": "azurearm67543ub5zsu77klvq0"
                              },
                              {
                                "storageAccountType": "Standard_GRS",
                                "location": "southindia",
                                "storageAccountName": "azurearm67543ub5zsu77klvq1"
                              }
                            ]
                          }

DeploymentDebugLogLevel :
```

Look at the outputs section, it clearly lists out the variable we generated and upon which our whole logic of depolying multiple resources existed.

```json
"_deployMultipleStorageAccounts": [
  {
    "storageAccountType": "Standard_GRS",
    "location": "southindia",
    "storageAccountName": "azurearm67543ub5zsu77klvq0"
  },
  {
    "storageAccountType": "Standard_GRS",
    "location": "southindia",
    "storageAccountName": "azurearm67543ub5zsu77klvq1"
  }
]
```

### Test the solution

Once the variable iteration logic is verified, it is time to deploy the tempalte to see that it actually creates.

Wait! before you jump into testing it you need to make a minor change. Can you guess what? Set `condition` to `true` inside the linked template deployment resource.

Below is a snippet of where that change goes inside the template.

```json
"resources": [
  {
    "type": "Microsoft.Resources/deployments",
    "apiVersion": "2019-10-01",
    "condition": true // set this to true to actually deploy the linked template,
    //<-- skipped below properties -->
  }
]
```

Deploy again, this time it should deploy multiple storage accounts.

```powershell
$rg =  "test_arm_rg"
New-AzResourceGroupDeployment -TemplateFile ./azuredeploy.json -TemplateParameterFile ./azuredeploy.parameters.json -ResourceGroupName $rg
```

## TLDR; Solution 🗞

Head over to this GitHub repository to see the ARM templates.

[ArmTemplateLoopExample](https://github.com/DexterPOSH/ArmTemplateLoopExample)

## References 📚

[Azure Resource Manager (ARM) Tools VSCode extension](https://marketplace.visualstudio.com/items?itemName=msazurermtools.azurerm-vscode-tools)

[Azure QuickStart Templates](https://github.com/Azure/azure-quickstart-templates)

[Linked Templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/linked-templates#linked-template)

[Deploy a linked template tutorial](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/deployment-tutorial-linked-template?tabs=azure-powershell#create-a-linked-template)

[Resource-Ieration](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/copy-resources)