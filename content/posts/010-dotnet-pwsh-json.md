---
title: "PowerShell to C# & back - JSON Seriazlie & Deserialize"
date: 2020-09-25T07:12:33+05:30
categories: [".NET"]
tags: [".net", "csharp", "json", "powershell"]
highlightjslanguages: ["csharp", "powershell"]
image: "/static/009/cover.png"
---

## Background

In the below previous post

* [PowerShell to C# & back - JSON parse & enumerate](https://dexterposh.github.io/posts/009-dotnet-pwsh-json/)

there were notes on how to enumerate & iterate over JSON documents. Let's continue
down the rabbit hole. In this post we will look at how to use `System.Text.Json` namespace
to:

* Serialize to JSON
* Deserialize from JSON

## Serialze to JSON

This means converting any type into a JSON string. We start off with a type named `Computer` and convert it into a JSON string. We call the `JsonSerializer.Serialze()` static method for this operation.

```csharp
#!/usr/bin/env dotnet-script
using System;
using System.Text.Json;

class Computer
{
    public string hostName { get; set; }

    // In C# 6 and later, we can assign a default value to an auto property
    public string domainName { get; set; } = "contoso.com";
    public string IpAddress { get; set; }
}

var machine1 = new Computer()
{
    hostName = "testvm01",
    IpAddress = "10.10.10.101"
};

// serialize the object into JSON string
var json = JsonSerializer.Serialize(machine1);
Console.WriteLine(json);
```

Now doing the same in PowerShell 🤑

```powershell
#!/usr/bin/env pwsh
using namespace System.Text.Json

class Computer {
    [string] $hostName
    [string] $domainName = "contoso.com" # this is the initial value
    [string] $IpAddress
}

$machine1 = [Computer] @{
    hostName = "testvm01"
    IpAddress = "10.10.10.101"
}

# Invoke the static method on the class, it requires 2 arguments
# the Value to convert and any JsonSerializer options which we have passed as $null
$json = [JsonSerializer]::Serialize($machine1, $null)
Write-Host -Object $json
```

## Deserialize from JSON

This means constructing a typed object from JSON string. No surprise there exists a `JsonSerializer.Deserialze()`
static method which can take a JSON string as input and can deserialized it into a typed object instance.

```csharp
#!/usr/bin/env dotnet-script
using System;
using System.Text.Json;
using System.Net;

class Computer
{
    public string hostName { get; set; }
    public string domainName { get; set; } = "contoso.com";
    public string IpAddress { get; set; }
}

string jsonComputer = "{\"domainName\":\"contoso.com\",\"hostName\":\"testvm01\",\"IpAddress\":\"10.10.10.101\"}";

var computerObject = JsonSerializer.Deserialize<Computer>(jsonComputer);
Console.WriteLine(computerObject.domainName);
```

In PowerShell the deserialization works really well with classes. If we have a type
defined in PowerShell class, we can use the `Deserialze()` static method and pass
it the type to deserialize the JSON string into.

```powershell
#!/usr/bin/env pwsh
using namespace System.Text.Json

class Computer {
    [string] $hostName
    [string] $domainName = "contoso.com" # this is the initial value
    [string] $IpAddress
}

$jsonComputer = '{"hostName":"testvm01","IpAddress":"10.10.10.101"}'

# Invoke the Deserialize static method on the class, it requires 3 arguments
# the Value to convert, and the type to convert it into and JsonSerializer options which we have passed as $null
$computer = [JsonSerializer]::Deserialize($jsonComputer, [Computer], $null)
# $computer | Get-Member # verify that the type got back is a Computer object type
Write-Host -Object $computer.domainName
```

## Bonus

Above way to deserialize the JSON string into a typed object in PowerShell is pretty neat
but we do mention another subtle way to achieve the same in our [Book](https://leanpub.com/powershell-to-csharp).

Cast-Initialization technique, which works by invoking an empty constrcutor and initializing values.

```powershell
$jsonComputer = '{"hostName":"testvm01","IpAddress":"10.10.10.101"}'
[Computer]($jsonComputer | ConvertFrom-Json)
```

## Conclusion 🗞

As a PowerShell developer, I learned using `System.Text.Json` namespace in PowerShell core to serialize/deserialize JSON in this post.

There will be a follow up post on this soon...

## Reference links 📖

[PowerShell to C# & Back leanpub book](https://leanpub.com/powershell-to-csharp)

[C# JSON Tutorial](http://zetcode.com/csharp/json/)