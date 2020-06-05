---
title: ".NET notes - create SHA256 hash"
date: 2020-06-05T10:18:40+05:30
tags: [".net", "csharp", "sha256", "cryptography"]
highlightjslanguages: ["csharp", "bash"]
---

## Background 🧐

Today I was looking to generate SHA256 hash for input string data.
Below are my notes on how I used dotnet script (interactive scripting experience) in .NET to experiment with it.

P.S. - Writing these small .NET recipes helps me in absorbing more.

## Walkthrough :zap:

### Using dotnet script

To quickly test features in .NET core, I use the [*dotnet script*](https://www.nuget.org/packages/dotnet-script/) global tool.

Begin with creating a dotnet script file (.csx extension).

```bash
mkdir hashing
cd hashing
dotnet script init sha256hash
```

Above dotnet command creates an executable file with below content.

```csharp
#!/usr/bin/env dotnet-script

Console.WriteLine("Hello world!");
```

Since this is an executable script, we can run it like this as well.

```bash
./sha256hash.csx
```

Output:

```output
Hello world!
```

### Using statements in scripts

Now, let's modify this script to quickly explore creating a SHA256 hash.

First, thing is we need to place some using statements to bring in the Cryptography and Text namespace like below.

So the content now looks like.

```csharp
#!/usr/bin/env dotnet-script

// Add the using statements
using System.Text;
using System.Security.Cryptography;
```

### Reading the docs

Moving on, after quickly browsing the documentation of the [SHA256 class](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.sha256.create?view=netcore-3.1#System_Security_Cryptography_SHA256_Create) I quickly noticed a *Create()* method which creates a default instance.

Also, noticed that the inheritance chain for this class is
Object -> HashAlgorithm -> SHA256

HashAlgorithm is the base class with below signature, notice it in turn inherits from IDisposable interface this quickly reminded me to use the `using` statement syntax to conveniently dispose this object after re-use, rather than calling `Dispose()` method myself inside try/catch/finally statements.

```csharp
public abstract class HashAlgorithm : IDisposable, System.Security.Cryptography.ICryptoTransform
```

Also, with C# 8 onwwards the using statement syntax is simplified so this means I can add the below to my .csx file.

```csharp
#!/usr/bin/env dotnet-script

using System.Text;
using System.Security.Cryptography;

string text = "DexterPOSH"; // this is our text for which we will generate hash

using (SHA256 hashAlgorithm = SHA256.Create()); // C# 8 syntax for using statement
```

The docs also reveal the `ComputeHash()` method which takes `byte[]` array as argument and returns the byte array back as well.
We need some way to convert our string input to byte array.

### string to byte[] conversion

Quick search suggests to use `Encoding.UTF8.GetBytes()` static method for converting string to byte array.
Using that in code now leads us to this point.

```csharp
#!/usr/bin/env dotnet-script

using System.Text;
using System.Security.Cryptography;

string text = "DexterPOSH"; // this is our text for which we will generate hash

using (SHA256 hashAlgorithm = SHA256.Create()); // C# 8 syntax for using statement
var hashedByteArray = hashAlgorithm.ComputeHash(Encoding.UTF8.GetBytes(input));
Console.WriteLine(hashedByteArray);
```

Output:

```output
System.Byte[]
```

Let's see how we can convert the byte[] to string object.

### byte[] to string conversion

Again search and got a hint of using the ``BitConverter.ToString()`` static method.
Let's add that logic in our script.

```csharp
#!/usr/bin/env dotnet-script

using System.Text;
using System.Security.Cryptography;

string text = "DexterPOSH"; // this is our text for which we will generate hash

using (SHA256 hashAlgorithm = SHA256.Create()); // C# 8 syntax for using statement
var hashedByteArray = hashAlgorithm.ComputeHash(Encoding.UTF8.GetBytes(input));
Console.WriteLine(BitConverter.ToString(hashedByteArray));
```

Output:

```output
1C-29-A2-30-0A-8A-99-6F-67-60-70-7E-21-0D-BD-61-B1-C9-3A-B7-4F-86-EE-13-7B-2E-DE-B6-01-6E-87-93
```

### remove '-' from string

As the last step let's replace the char `-` from the output string.
`String` class has replace method which invoke.

```csharp
#!/usr/bin/env dotnet-script

using System.Text;
using System.Security.Cryptography;
using (SHA256 hashAlgorithm = SHA256.Create())
{
    string input = "DexterPOSH";

    byte[] data = hashAlgorithm.ComputeHash(
        Encoding.UTF8.GetBytes(input)
    );
    Console.WriteLine(BitConverter.ToString(data).Replace("-", String.Empty));
}
```

Output:

```output
1C29A2300A8A996F6760707E210DBD61B1C93AB74F86EE137B2EDEB6016E8793
```

## Solution 😎

Content of the sha256hash.csx file.

```csharp
#!/usr/bin/env dotnet-script

using System.Text;
using System.Security.Cryptography;

using (SHA256 hashAlgorithm = SHA256.Create())
{
    string input = "Deepak Singh Dhami";

    byte[] data = hashAlgorithm.ComputeHash(
        Encoding.UTF8.GetBytes(input)
    );
    Console.WriteLine(BitConverter.ToString(data).Replace("-", String.Empty));
}
```

Run the above file

```bash
dotnet script run ./sha256hash.csx # or simply ./sha256hash.csx
```

Output:

```output
1C29A2300A8A996F6760707E210DBD61B1C93AB74F86EE137B2EDEB6016E8793
```

## Reference links 📖

[System.Security.CryptoGraphy Class](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.sha256?view=netcore-3.1#constructors)

[HashAlgorithm Base Class](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.hashalgorithm?view=netcore-3.1)

[using statement reference in C#](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/using-statement)

[C# String To Byte Array](https://www.c-sharpcorner.com/article/c-sharp-string-to-byte-array/)

[Convert Byte Array To String In C#](https://www.c-sharpcorner.com/article/how-to-convert-a-byte-array-to-a-string/)