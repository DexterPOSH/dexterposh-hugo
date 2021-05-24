---
title: "Azure Vms Resource Graph Queries"
date: 2021-05-24T10:47:38+05:30
categories: ["DevOps"]
tags: ["Azure", "DevOps", "ResourceGraph", "powershell", "VMs"]
highlightjslanguages: [ "powershell", "bash"]
---

## Why use Resource Graph instead Az CLI/ PowerShell?

If we want to search for resources meeting certain criteria across all our subscriptions, we can't use Az CLI or Az PowerShell to do this type of queries since it would require a lot of overhead to filter and switch between subscription contexts.

Pesudo-Code (not so performant)

```txt
Foreach subscription in subscriptions:
    set Az Context
    Get the VMs, Filter based on criteria
```

Resource Graph queries can help here, as they can be used to fetch metadata about the resource and then after their presence is validated, maybe perform some operation on it. With PowerShell we can even group these resources together based on SubscriptionID and then iterate over each subscription (set the right context) and perform actions.

Pesudo-Code (single query and performant)

```txt
Query Azure Graph for resources based on criteria across all the subscriptions
```

In this post, I share some of the Resource Graph Queries I have found useful while working with Virtual Machines.

## Prerequisite

Prerequisite is the Az CLI (with graph extension) or Az.ResourceGraph PowerShell module which supports this.

```pwsh
# Install the Resource Graph module from PowerShell Gallery
Install-Module -Name Az.ResourceGraph
```

For Az CLI run the below to install the extension:

```bash
# Add the Resource Graph extension to the Azure CLI environment
az extension add --name resource-graph
```

## Virtual Machine Queries

Below are list of queries for Virtual machines.

### Find Virtual Machine with Name

If you want to find out if a virtual machine is present across all the subscriptions you have access to, you can use the below Resource graph query with Az CLI.

```pwsh
# Using Az.ResourceGraph Module
Search-AzGraph -Query "where type =~ 'Microsoft.Compute/VirtualMachines' and name == 'testvm01'"

Now for most of the queries in this post, you can do one to one translation to Az CLI commands by extracting the query and using it, I prefer to use PowerShell and that will be used in the rest of the examples.

```shell
# Using AZ CLI
az graph query -q "where type =~ 'Microsoft.Compute/VirtualMachines' and Name == 'testvm01'"
```

### Find Virtual machines with name regex matching

Note - This regex matching can be applied to any resource or any property defined in the Resource schema as well.

```pwsh
Search-AzGraph -Query "where type =~ 'Microsoft.Compute/VirtualMachines' and name matches regex 'test-[0-9].*'"
```

### Find Virtual machines with a specific tag

Gist is Kusto allows for filtering at all the levels, so we can filter based on a specific tag or can chain multiple tags to filter.

```pwsh
# Filter based on a tag called the department
# Note that tags are case-sensitive
Search-AzGraph -Query "where type =~ 'Microsoft.Compute/VirtualMachines' and tags.department =~ 'ITDepartment'"
 
 
# Filter based on the groupEmail name
Search-AzGraph -Query "where type =~ 'Microsoft.Compute/VirtualMachines' | where tags.group =~ 'SRE'"
 
# Combine both the department & group
 Search-AzGraph -Query "where type =~ 'Microsoft.Compute/VirtualMachines' | where tags.group =~ 'SRE' or tags.department =~ 'ITDepartment'"
 ```

### Find Virtual machines deployed using MarketPlace images

This query utilizes the fact that the publisher field for a market place will not be empty.

Note - Remove the end 'limit' command expression at the end of query if you need the exhaustive list.

```pwsh
Search-AzGraph -Query "where type =~ 'Microsoft.Compute/VirtualMachines' and isnotempty(properties.storageProfile.imageReference.publisher)|  limit 1"
```

### Find Virtual machines deployed using custom Images

This query utilizes the fact that the publisher field does not exist for a VM deployed using a generalized image.

**Note** - Remove the end 'limit' command expression at the end of query if you need the exhaustive list.

```pwsh
Search-AzGraph -Query "where type =~ 'Microsoft.Compute/VirtualMachines' and isempty(properties.storageProfile.imageReference.publisher)| limit 1"
```

### Find Virtual machines which do not have a Tag populated

In this example the query checks for VMs which do not have a specific tag named Department created but not populated.

Note - Remove the end 'limit' command expression at the end of query if you need the exhaustive list.

```pwsh
Search-AzGraph -Query "where type =~ 'Microsoft.Compute/VirtualMachines' and isempty(tags.group)
| project name, location, resourceGroup, subscriptionId, Group=tags.group
| limit 5"
```

### Find Virtual machine with an IPAddress

This can simply be done by doing a reverse lookup of the IPaddress but in some cases where the machines did not have these DNS records created it was a pain.

We start by looking for network interfaces which have this IP address

```pwsh
$nic = Search-AzGraph -Query "where type =~ 'Microsoft.Network/networkInterfaces' |
where properties.ipConfigurations[0].properties.privateIPAddress == '10.10.10.6'"
 
$nic.properties.virtualMachine.id # this is the resource ID of the VM
 
# Now we can fire off another query to get the VM info
Search-AzGraph -Query "where type =~ 'Microsoft.Compute/VirtualMachines' and id == `'$($nic.properties.virtualMachine.id)`'"
```

### Gather extra Virtual Machine information

To gather the relevant information for different VM resources e.g. network, storage, subscription etc, one can use PowerShell to extend the object returned from the resource graph query.

Below I show usage of a crude way of using PowerShell to do this e.g. using a filter.

```pwsh
Filter GetVMInfo {
    $networkQuery = "where type =~ 'Microsoft.Network/networkInterfaces' and id == `'$($PSItem.properties.networkProfile.networkInterfaces[0].id)`'"
    $network = Search-AzGraph -Query $networkQuery
  
    [pscustomObject]@{
        Name = $PSItem.name
        Location = $PSItem.location
        Tags = $PSItem.Tags
        ResourceGroup = $PSItem.resourceGroup
        SubscriptionID = $PSItem.subscriptionId
        SubscriptionName = (Get-AzSubscription -SubscriptionId $PSItem.subscriptionId).Name
        ProvisioningState = $PSItem.properties.provisioningState
        VMSize = $PSItem.properties.hardwareProfile.vmSize
        BaseOSImage = $PSItem.properties.storageProfile.imageReference.id.Split('/')[-1]
        OSType = $PSItem.properties.storageProfile.osDisk.osType
        OSDiskSize = $PSItem.properties.storageProfile.osDisk.diskSizeGB
        DataDisks = foreach ($disk in $PSItem.properties.storageProfile.dataDisks) {
                        "Name = {0}, Size = {1}, IsManaged = $(if ($disk.managedDisk) { $true} else {$false})"    -f $disk.name, $disk.diskSizeGB
                    }
        IPAddress = $network.properties.ipConfigurations[0].properties.privateIPAddress
        SubnetName = $network.properties.ipConfigurations[0].properties.subnet.id.Split('/')[-1]
        DNSservers = @($network.properties.dnsSettings.dnsServers)
    }
}
```

Use the above filter like below:

```pwsh
# Search for the VM first using the Az Resurce Graph
$VM = Search-AzGraph -Query "where type =~ 'Microsoft.Compute/VirtualMachines' and name == 'testvm01'"

# Pipe the object(s) found to the filter we defined
$VM | GetVMInfo
```

## Conclusion

Azure Resource Graph is a powerful CMDB solution for querying Azure resources, they can be used wisely to enhance existing scripts or create new robust and scalable automations.
