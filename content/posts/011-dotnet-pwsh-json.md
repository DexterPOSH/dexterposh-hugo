---
title: "PowerShell to C# & back - JSON Create, Beautify"
date: 2020-10-05T09:07:45+05:30
categories: [".NET"]
tags: [".net", "csharp", "json", "powershell", "PwshtoCSharpAndBack"]
highlightjslanguages: ["csharp", "powershell"]
image: "/static/009/cover.png"
---

## Background

I have been following up the [C# Tutorial](http://zetcode.com/csharp/json/) and working my way through examples and converting them to PowerShell and notes.

## JSON Create object

Interesting to see that `System.Text.Json` namespace offers `Utf8JsonWriter` type to write JSON (UTF-8 encoded) string from common .NET types e.g. `String`, `Int32` and `DateTime`. What this means is we can write custom JSON strings in-memory (using MemoryStream) and later flush `Utf8JsonWriter` content to get the string back.
I wanted to use this new writer to create a JSON string which will look like below

```json
{
    "ComputerName": "TestVM01"
}
```

Let's do this in C# first. Below is a C# script which I am using.

```csharp
#!/usr/bin/env dotnet-script
using System;
using System.Text;
using System.Text.Json;
using System.Net;
using System.IO;

using (var memStream = new MemoryStream())
{
    using (var jsonWriter = new Utf8JsonWriter(memStream))
    {
        jsonWriter.WriteStartObject(); // this is the start of JSON Object
        jsonWriter.WriteString("ComputerName", "TestVM01");
        jsonWriter.WriteEndObject(); // this is the end of the JSON Object
        //jsonWriter.Flush();
    }

    string json = Encoding.UTF8.GetString(memStream.ToArray());
    Console.WriteLine(json);
}
```

This effectively translates to below in PowerShell, see inline comments.

```powershell
#!/usr/bin/env pwsh
using namespace System.IO
using namespace System.Text
using namespace System.Text.Json

# create a memory stream to hold the data in-memory
# the memory stream is a performant way to hold data without locking the source
$memStream = [MemoryStream]::new()

# use the Utf8 Json Writer object
$jsonWriter = [Utf8JsonWriter]::new($memStream)

# indicate the JSON object is starting
$jsonWriter.WriteStartObject();

# now the JSON string is written
$jsonWriter.WriteString("ComputerName", "TestVM01");

# instruct the JSON object is ending
$jsonWriter.WriteEndObject();

# flush the json writer content to the stream
$jsonWriter.Flush();

# dispose of the json writer
$jsonWriter.Dispose();

# Get the string from the byte[] array from the memory stream
[Encoding]::UTF8.GetString($memStream.ToArray())

# dispose the memory stream once read the json string
$memStream.Dispose()
```

Below is the sample output that gets generated. Note it is a regular JSON string, let's make it prettier in next section.
![JsonStringOutput](/static/011/JsonStringOutput.png)

## Beautify JSON

This is where tapping into the underlying .NET APIs to learn something really shines, there are so many tweaks that can be applied.

One such instance is to make out output JSON string look pretty, we can pass some options to our Utf8JsonWriter instance.

```csharp
#!/usr/bin/env dotnet-script
using System;
using System.Text;
using System.Text.Json;
using System.Net;
using System.IO;

using (var memStream = new MemoryStream())
{
    using (var jsonWriter = new Utf8JsonWriter(memStream, new JsonWriterOptions {Indented = true}))
    {
        jsonWriter.WriteStartObject(); // this is the start of JSON Object
        jsonWriter.WriteString("ComputerName", "TestVM01");
        jsonWriter.WriteEndObject(); // this is the end of the JSON Object
        //jsonWriter.Flush();
    }

    string json = Encoding.UTF8.GetString(memStream.ToArray());
    Console.WriteLine(json);
}
```

This is pretty similar in PowerShell as well, create options for the `Utf8JsonWriter` and pass them.

```powershell
#!/usr/bin/env pwsh
using namespace System.IO
using namespace System.Text
using namespace System.Text.Json

# create a memory stream to hold the data in-memory
# the memory stream is a performant way to hold data without locking the source
$memStream = [MemoryStream]::new()

<#
create Json writer options which is a structure (value-type) using PowerShell cast initialization technique
or you could do below:
$jsonWriterOptions = [JsonWriterOptions]::new(); $jsonWriterOptions.Iden
$jsonWriterOptions.Indented = $true
#>
$jsonWriterOptions = [JsonWriterOptions]@{"Indented" = $true}

# use the Utf8 Json Writer object
$jsonWriter = [Utf8JsonWriter]::new($memStream, $jsonWriterOptions)

# indicate the JSON object is starting
$jsonWriter.WriteStartObject();

# now the JSON string is written
$jsonWriter.WriteString("ComputerName", "TestVM01");

# instruct the JSON object is ending
$jsonWriter.WriteEndObject();

# flush the json writer content to the stream
$jsonWriter.Flush();

# dispose of the json writer
$jsonWriter.Dispose();

# Get the string from the byte[] array from the memory stream
[Encoding]::UTF8.GetString($memStream.ToArray())

# dispose the memory stream once read the json string
$memStream.Dispose()
```

Output is below:
![Beautify JSON string output](/static/011/beautifyJsonString.png)


## Conclusion

As a PowerShell developer, I learned using `Utf8JsonWriter` to create JSON strings from scratch and beautify them.

There will be a follow up post on this soon...

## Reference links 📖

[PowerShell to C# & Back leanpub book](https://leanpub.com/powershell-to-csharp)

[C# JSON Tutorial](http://zetcode.com/csharp/json/)