---
title: "PowerShell to C# & back - JSON parse & enumerate"
date: 2020-09-16T11:12:33+05:30
categories: [".NET"]
tags: [".net", "csharp", "json", "powershell", "pwshtoC#andback"]
highlightjslanguages: ["csharp", "powershell"]
image: "/static/009/cover.png"
---

## Background

Coming from PowerShell background, while learning ASP.NET Core based development
I wanted to wrap my head around how to handle JSON, so ended up taking notes on how to do this in C# and as
an exercise convert those into PowerShell code snippets for my reference.

In my explorations I like using the dotnet-script global tool for interactive user experience with C# and PowerShell core (which is already an interactive shell). So, you will see that I mostly use C# script files (.csx) and PowerShell script files (.ps1).

> Giving the credit where it's due, I followed this [tutorial](http://zetcode.com/csharp/json/) to learn JSON handling in C# and made these notes of my own.

In this post the topics covered are:

* Parsing JSON
* Enumerating JSON

> **System.Text.Json**
>
> This is the new namespace in .NET core which provides the API to work with JSON.

## JSON parsing

JSON parsing is basically reading the JSON data and converting it into a `JsonDocument` typed object in C#.

Long story short:

* Use the static method `Parse` on the `JsonDocument` class to consume a JSON string and get an object back.
* `RootElement` on the instance of `JsonDocument` gives you the entire JSON object
* Root element if it contains array can be indexed
* To get the value of a property on the JSON object use `GetProperty('propertyName')`

Below are the contents of a test.csx file.

```csharp
#!/usr/bin/env dotnet-script
using System;
using System.Text.Json;

string data = " [ {\"name\": \"John Doe\", \"occupation\": \"gardener\"}, " +
                "{\"name\": \"Peter Novak\", \"occupation\": \"driver\"} ]";

// using statement is used to dispose of the JsonDocument object later
// Parse static method parses the JSON string and returns a JsonDocument instance
using (JsonDocument doc = JsonDocument.Parse(data))
{
    var root = doc.RootElement; // RootElement is the real object we are after
    var user1 = root[0]; // since our object contains an array, we can index into it
    var user2 = root[2];

    // Use the method GetProperty() to get properties of the JsonElement object
    Console.WriteLine(user1.GetProperty("name")); // GetProperty()
    Console.WriteLine(user2.GetProperty("name"));
}
```

To do the same in PowerShell, below are the contents of a test.ps1 file.

```powershell
#!/usr/bin/env pwsh
using namespace System.Text.Json

$data = '[{"name": "John Doe", "occupation": "gardener"},' +
                '{"name": "Peter Novak", "occupation": "driver"} ]';

$doc = [JsonDocument]::Parse($data)
$root = $doc.RootElement
$user1 = $root[0]
$user2 = $root[1]

Write-Host $User1.GetProperty("name")
Write-Host $User2.GetProperty("name")

# dispose the JsonDocument once done
# We don't do this in C# coz of the using statement automatically doing this
$doc.Dispose()
```

## JSON Enumerate

In the above example we indexed into the elements of the array directly and then used `GetProperty()` this only works if you know the number of elements in advance along with property names, but there are better ways to iterate and enumerate.

Trick for someone new is to remember:

* Enumerating arrays to fetch different items
* Enumerating objects to fetch properties.

```csharp
#!/usr/bin/env dotnet-script

using System;
using System.Text.Json;

string data = " [ {\"name\": \"John Doe\", \"occupation\": \"gardener\"}, " +
                "{\"name\": \"Peter Novak\", \"occupation\": \"driver\"} ]";

using (var doc = JsonDocument.Parse(data))
{
    JsonElement root = doc.RootElement;

    // call method EnumerateArray() to iterate over items in the array
    foreach (var user in root.EnumerateArray())
    {
        Console.WriteLine("====== User details ====");
        // EnumerateObject() method is used return an ObjectEnumerator
        // which is then used to get the Name & Value pair for each property
        foreach (var prop in user.EnumerateObject())
        {
            Console.WriteLine($"{prop.Name} -> {prop.Value}");
        }
    }
}
```

Doing the same in PowerShell makes it easier to grasp.

```powershell
#!/usr/bin/env pwsh
using namespace System.Text.Json

$data = "[ {`"name`": `"John Doe`", `"occupation`": `"gardener`"}, " +
                "{`"name`": `"Peter Novak`", `"occupation`": `"driver`"} ]";

$doc = [JsonDocument]::Parse($data)
$root = $doc.RootElement


foreach ($user in $root.EnumerateArray()) {
    Write-Host "====== User details ===="
    foreach ($property in $user.EnumerateObject()) {
        Write-Host $("{0} -> {1}" -f $property.Name, $property.Value)
    }
}
```

## Conclusion 🗞

In my continued efforts to learn C# language and take some of the learnings back to PowerShell, I realized that though the JSON cmdlets provide a lot of ease working with JSON, there are other ways to skin the cat.

There will be a follow up post on this soon...

## Reference links 📖

[PowerShell to C# & Back leanpub book](https://leanpub.com/powershell-to-csharp)

[C# JSON Tutorial](http://zetcode.com/csharp/json/)
