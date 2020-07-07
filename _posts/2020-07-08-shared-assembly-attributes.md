---
layout: post
title: Shared Assembly Attributes
author: ujr
date: 2020-07-07
---

**Abstract:** in a solution with many projects, it may be useful
to extract constant information from the per-project *AssemblyInfo.cs*
into a solution-wide *SharedAssemblyInfo.cs* file.

## About Assembly Attributes

An assembly (.dll or .exe file) contains four types
of data: (1) an assembly manifest, (2) an application
manifest, (3) compiled code, (4) resources.

The assembly manifest contains assembly name and version,
a directory of contained types and resources, and other
meta information. It is automatically embedded into the
assembly by the compiler.

Name, version, and meta information is provided by
`[assembly: AssemblyFoo(...)]` attributes. They must
appear as the first thing in the source file (except
`using` and `extern` clauses).

Common assembly attributes include:

```C#
[assembly: AssemblyCompany("Company Name")]
[assembly: AssemblyConfiguration("Debug")]
[assembly: AssemblyCopyright("© 2020 Company Name")]
[assembly: AssemblyCulture("")] // only set on satellite assemblies
[assembly: AssemblyDescription("blah blah")]
[assembly: AssemblyFileVersion("1.2.3.444")] // see below
[assembly: AssemblyInformationalVersion("1.2 Update 3")] // display version
[assembly: AssemblyProduct("ProSuite")]
[assembly: AssemblyTitle("Component")]
[assembly: AssemblyVersion("1.2.3.0")] // used by the CLR (see below)
```

## Showing Attributes

To see all assembly attributes use a tool like [ILSpy][ilspy].
Some assembly attributes are shown by the **File Properties**
dialog in Windows Explorer as indicated in this table:

|Property|Attribute|
|--------|---------|
|File description|AssemblyTitle|
|File version|AssemblyFileVersion|
|Product name|AssemblyProduct|
|Product version|AssemblyInformationalVersion|
|Copyright|AssemblyCopyright|
|Legal trademarks|AssemblyTrademark|

Explorer will not show some of the properties
if their attributes are missing or empty.
The value of the `AssemblyDescription` and
`AssemblyConfiguration` attributes appear
to *not* be shown here.

## Assembly Versions

There are three version attributes:

- `AssemblyVersion`: for assembly referencing (used by the CLR,
  defaults to 0.0.0.0)
- `AssemblyFileVersion`: to differentiate files with the
  same AssemblyVersion (defaults to AssemblyVersion)
- `AssemblyInformationalVersion`: for display purposes,
  an arbitrary string (defaults to AssemblyFileVersion)

For example, here are the version attributes from Microsoft's **mscorlib**
(the Multilanguage Standard Common Object Runtime Library, colloquially
also the Microsoft Core Library):

```C#
[assembly: AssemblyVersion("4.0.0.0")]
[assembly: AssemblyFileVersion("4.8.4180.0")]
[assembly: AssemblyInformationalVersion("4.8.4180.0")]
```

The `AssemblyVersion` is part of the assembly's fully qualified name, as
in `System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089`.
Therefore, changing the `AssemblyVersion` changes the assembly's identity
and breaks references (could be fixed with assembly binding redirection).
The two other version attributes are ignored by the CLR and can be changed
at will. Typically, the `AssemblyFileVersion` includes a build number,
and `AssemblyInformationalVersion` is a human friendly string.

**Side note:** `System.Version` provides four integer
components named *Major.Minor.Build.Revision*.
You are free to use the four components in other ways,
typically: `Build` as Patch and `Revision` as Build.

```C#
var v = new System.Version("1.2.3.4");
System.Console.WriteLine("{0}.{1}.{2}.{3}", v.Major, v.Minor, v.Build, v.Revision);
// outputs 1.2.3.4
```

## AssemblyInfo.cs

Visual Studio automatically creates an *AssemblyInfo.cs*
file within projects to contain the AssemblyFoo attributes.
It also provides a dialog window to set them graphically.

> With .NET Core, this will be different, the *AssemblyInfo.cs* files gone.

## Centralising shared attributes

Usually, much of the information in *AssemblyInfo.cs*
is constant across the projects in a Visual Studio solution.
This redundancy can be reduced by factoring the common
attributes out into a solution-wide *SharedAssemblyInfo.cs*
file (or any name you like) and then *linking* from each
project to this shared file.

1. Create solution-wide *SharedAssemblyInfo.cs* file
   and add globally shared assembly attributes.
2. Link this shared file into each project's Properties folder:
   Right click Properties / Add / Existing Item / Add As Link
3. Remove duplicate attributes from the per-project
   *AssemblyInfo.cs* files.

In this case, the project's Assembly Information dialog
should not be used as properties would be added to the
per-project *AssemblyInfo.cs* file, thus creating duplicate
assembly attributes (a compile-time error).

In the project (.csproj) file, *Add As Link* creates a `Compile`
element that refers to the real location, with a nested `Link`
element that states the place within the project source tree:

```XML
<Compile Include="..\SharedAssemblyInfo.cs">
  <Link>Properties\SharedAssemblyInfo.cs</Link>
</Compile>
```

**Alternatively,** the globally constant assembly attributes
can be updated through an MSBuild task at build-time. Both
[MSBuild Extension Pack][mepack] and [MSBuild Community Tasks][mctasks]
provide tasks for this job, but both projects appear to be dormant.

## Recommendations

- Do fill in as many of the assembly attributes as possible;
  it will look more prefessional (for example in the Explorer's
  File Properties dialog)

- State AssemblyCopyright as `© YYYY Owner` not as `Copyright (c) YYYY Owner`.

- Consider linking to a globally shared assembly info file
  (to avoid redundancy and simplify automated version bumping).

- State `[assembly:CLSCompliant(true)]` only if the entire assembly
  is *intended* to be CLS compliant (so that compile-time warnings may
  remind you of your intention). The default is false. (The CLS is
  the common subset of all CLR languages, so that a CLS compliant
  assembly can be used by any other CLR language.)

- Increment `AssemblyVersion` when a breaking change is made.
- Increment `AssemblyFileVersion` on each build (automatically).
- Set `AssemblyInformationalVersion` if it deviates from `AssemblyFileVersion`
  (to which it defaults). May be useful for a nicely “human readable”
  version indication along with the product name.

### Example *S*haredAssemblyInfo.cs* (solution-wide)

```C#
using System.Reflection;

[assembly: AssemblyProduct("ProSuite")]
[assembly: AssemblyCompany("")]
[assembly: AssemblyCopyright("© 2020 The ProSuite Authors")]
[assembly: AssemblyTrademark("")]

#if DEBUG
[assembly: AssemblyConfiguration("Debug")]
#else
[assembly: AssemblyConfiguration("Release")]
#endif

[assembly: AssemblyVersion("1.2.0.0")]
[assembly: AssemblyFileVersion("1.2.3.444")]
[assembly: AssemblyInformationalVersion("1.2 patch 3")]
```

### Example *AssemblyInfo.cs* (per-project)

```C#
using System;
using System.Reflection;
using System.Runtime.InteropServices;

[assembly: AssemblyTitle("ProSuite.Essentials")]
[assembly: AssemblyDescription("Coding essentials (assertions, attributes, etc.)")]
[assembly: AssemblyCulture("")] // blank if neutral, only set on satellites!

[assembly: ComVisible(false)] // set to true if access from COM required
[assembly: CLSCompliant(false)] // set to true if CLS compliance is intended

// This GUID is for the ID of the typelib if this project is exposed to COM:
[assembly: Guid("xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx")]
```

## References

- Albahari and Albahari: C# 5.0 in a Nutshell. O'Reilly 2012.
  Chapter 18 on Assemblies. [Amazon](https://www.amazon.com/dp/1449320104)
- Stack Overflow: [What are the best practices for using Assembly Attributes?](https://stackoverflow.com/questions/62353)
- Stack Overflow: [What are differences between AssemblyVersion, AssemblyFileVersion
  and AssemblyInformationalVersion?](https://stackoverflow.com/questions/64602)
- heise Developer: [Assembly-Meta-Daten in .NET Core](https://heise.de/-4509446)
- Microsoft Documentation: [AssemblyCultureAttribute Class](https://docs.microsoft.com/en-us/dotnet/api/system.reflection.assemblycultureattribute)

[ilspy]: https://github.com/icsharpcode/ILSpy
[mepack]: https://www.nuget.org/packages/MSBuild.Extension.Pack/
[mctasks]: https://github.com/loresoft/msbuildtasks/
