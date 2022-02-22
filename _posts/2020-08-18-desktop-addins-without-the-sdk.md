---
layout: post
title: Build Desktop Add-Ins without the SDK
author: ujr
date: 2020-08-18
---

How to build an Add-In for ArcMap without the SDK.  
How to build an Add-In for multiple versions of ArcMap.

## Why?

After updates of Visual Studio (VS) the SDK regularly
cannot find its dependencies, usually one or the other
of a Visual Studio DLL. Could be fixed with some effort
(keeping old VS around, assembly binding redirection),
but the problem recurs.

Moreover, you may have to build your Add-In for multiple
versions of ArcGIS. This requires some changes to your
build process anyway.

## How?

The approach described here is for ArcGIS Desktop.
For ArcGIS Pro we have not experienced the same problem.
Though a similar approach could be used, we see no need for it.

- Setup project layout (directory structure):
  - anything you like, but we found the example below useful
  - if you build for multiple ArcGIS versions, keep copies of
    the Primary Interop Assemblies (PIAs, *ESRI.ArcGIS.Foo.dll*)
    in your project layout (but do not deploy them).
- Exclude the build output directory from source control
- Edit .csproj file(s)
  - Change *Config.esriaddinx* from AddInContent to Content
    and make sure to CopyToOutputDirectory *Always*
  - Do the same for images and indeed for all former AddInContent
  - Remove the Import of *ESRI.ArcGIS.AddIns.targets*
  - References to *ESRI.ArcGIS.Foo.dll*: if building for multiple
    ArcGIS versions, use a HintPath with an environment variable
    to hint at the proper version of the DLL. See example below.
    Be sure to set Private=False.
  - Cleanup:
    remove the property group with the *esriAddIn* ZipFileExtension,
    and remove the element with the ESRIAddInProperties; you may
    find opportunity for some further cleanup.
- Create an MSBuild file, say *MyProject.build* that does the following:
  - build the project as VS would (invoke MSBuild in your .csproj)
  - copy build artifacts (.dll) and support files (images, Config.esraddinx)
    into the proper folders in the output directory
  - Update Target version in *Config.esriaddinx*:
    use MSBuild's XmlPoke task to update the Add-In's target version
    (only necessary if you want to build your Add-In for multiple
    ArcGIS versions; if building only for one, just hardcode the value).
  - Wrap up the build artifacts and support files into a Zip archive
    and make sure its extension is `.esriaddin`. You may use the
    MSBuild ZipDirectory task.
  - Extras: default values, a target to clean,
    build for multiple ArcGIS versions, etc.

## Example

Consider a project layout like the following:

```text
build/
  MyProject.build                the MSBuild file
  BuildAll.bat                   to kick the build manually
  output/                        here goes the build output
lib/
  Esri/ArcGIS/10.5/              required PIAs for ArcGIS 10.5
  Esri/ArcGIS/10.6/              required PIAs for ArcGIS 10.6
src/
  MyProject/MyProject.csproj     the VS project file
  MyProject/Config.esriaddinx    the Esri Add-In config file
  MySolution.sln                 the VS solution file
  MySolution_10.5.bat            to launch VS if you have ArcGIS 10.5
  MySolution_10.6.bat            to launch VS if you have ArcGIS 10.6
.gitignore                       excludes from source control
```

Be sure to exclude the build output directory from source control:

```text
# .gitignore
build/output/
```

Your project file, say *MyProject.csproj*, might look like this.
Note the use of an environment variable in the `HintPath` of
references to the PIAs.

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="14.0" DefaultTargets="Build"
         xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup>
    <Configuration Condition=" '$(Configuration)' == '' ">Debug</Configuration>
    <Platform Condition=" '$(Platform)' == '' ">AnyCPU</Platform>
    <TargetArcGISVersion Condition=" '$(TargetArcGISVersion)' == '' ">10.5</TargetArcGISVersion>
    <TargetFrameworkVersion Condition=" '$(TargetFrameworkVersion)' == '' ">v4.5</TargetFrameworkVersion>
    <OutputType>Library</OutputType>
    <AppDesignerFolder>Properties</AppDesignerFolder>
    <RootNamespace>MyProject</RootNamespace>
    <AssemblyName>MyProject</AssemblyName>
    ...
  </PropertyGroup>
  <PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Debug|AnyCPU' ">
    ...
  </PropertyGroup>
  <PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Release|AnyCPU' ">
    ...
  </PropertyGroup>
  <ItemGroup>
    <Reference Include="ESRI.ArcGIS.Desktop.AddIns">
      <SpecificVersion>False</SpecificVersion>
      <EmbedInteropTypes>False</EmbedInteropTypes>
      <HintPath>..\..\lib\Esri\ArcGIS\$(TargetArcGISVersion)\ESRI.ArcGIS.Desktop.AddIns.dll</HintPath>
      <Private>False</Private>
    </Reference>
    <!-- and similar for the other Esri PIAs -->
  </ItemGroup>
  <ItemGroup>
    <Reference Include="System" />
    <Reference Include="System.Core" />
  </ItemGroup>
  <ItemGroup>
    <Compile Include="MyCode.cs" />
    <Compile Include="Config.Designer.cs">
      <AutoGen>True</AutoGen>
      <DependentUpon>Config.esriaddinx</DependentUpon>
      <CopyToOutputDirectory>Never</CopyToOutputDirectory>
    </Compile>
    <Compile Include="Properties\AssemblyInfo.cs" />
  </ItemGroup>
  <ItemGroup>
    <!-- Note: Content, not AddInContent -->
    <Content Include="Config.esriaddinx">
      <Generator>ArcGISAddInHostGenerator</Generator>
      <LastGenOutput>Config.Designer.cs</LastGenOutput>
      <CopyToOutputDirectory>Always</CopyToOutputDirectory>
    </Content>
  </ItemGroup>
  <ItemGroup>
    <Content Include="Images\MyIcon.png">
      <CopyToOutputDirectory>Always</CopyToOutputDirectory>
    </Content>
  </ItemGroup>
  <!-- etc. -->
  <Import Project="$(MSBuildToolsPath)\Microsoft.CSharp.targets" />
  <!--
  <Target Name="BeforeBuild"></Target>
  <Target Name="AfterBuild"></Target>
  -->
</Project>
```

Launch Visual Studio with a batch file that sets required
environment variables. For example:

```batch
@cd /d "%~dp0"

set TargetArcGISVersion=10.5
set TargetFrameworkVersion=v4.5

@set VS="C:\Program Files (x86)\Microsoft Visual Studio\2019\Professional\Common7\IDE\devenv.exe"

%VS% MySolution.sln
```

You'll need an extra MSBuild file to create the Add-In,
which is a Zip archive, from the build artifacts:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project DefaultTargets="Dist" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup>
    <Configuration Condition=" '$(Configuration)' == '' ">Debug</Configuration>
    <!-- Defaults in case the Build target is called directly (not via Dist) -->
    <TargetArcGISVersion Condition=" '$(TargetArcGISVersion)' == '' ">10.5</TargetArcGISVersion>
    <TargetFrameworkVersion Condition=" '$(TargetFrameworkVersion)' == '' ">v4.5</TargetFrameworkVersion>
    <!-- For easy reference below -->
    <ProjectPath>..\src\MyProject</ProjectPath>
    <OutputPath>..\src\MyProject\bin\$(Configuration)</OutputPath>
    <ProjectFile>$(ProjectPath)\MyProject.csproj</ProjectFile>
    <!-- How to update Config.esriaddinx (Namespace must be a well-formed XML fragment) -->
    <EsriAddInNamespace>&lt;Namespace Prefix='x' Uri='http://schemas.esri.com/Desktop/AddIns'/&gt;</EsriAddInNamespace>
    <TargetVersionQuery>/*/x:Targets[count(./x:Target[@name='Desktop'])=1]/x:Target[@name='Desktop']/@version</TargetVersionQuery>
  </PropertyGroup>

  <Target Name="Clean">
    <RemoveDir Directories="output\MyProject\10.5" Condition="Exists('output\MyProject\10.5')" />
    <RemoveDir Directories="output\MyProject\10.6" Condition="Exists('output\MyProject\10.6')" />
  </Target>

  <Target Name="Dist" DependsOnTargets="Clean">
    <!-- Recursive MSBuild so that we can change Properties (CallTarget will not work) -->
    <MSBuild Projects="$(MSBuildProjectFile)" Targets="Build"
             Properties="TargetArcGISVersion=10.5;TargetFrameworkVersion=v4.5" />
    <MSBuild Projects="$(MSBuildProjectFile)" Targets="Build"
             Properties="TargetArcGISVersion=10.6;TargetFrameworkVersion=v4.5" />
  </Target>

  <Target Name="Build">

    <PropertyGroup>
      <TargetFolder>output\MyProject\$(TargetArcGISVersion)</TargetFolder>
      <TargetAddInFile>output\MyProject\MyProject.ArcMap.$(TargetArcGISVersion).esriAddIn</TargetAddInFile>
    </PropertyGroup>

    <MSBuild Projects="$(ProjectFile)" Targets="Clean"
             Properties="Configuration=$(Configuration)" />

    <MSBuild Projects="$(ProjectFile)" Targets="Build"
             Properties="Configuration=$(Configuration);TargetArcGISVersion=$(TargetArcGISVersion);TargetFrameworkVersion=$(TargetFrameworkVersion)" />

    <ItemGroup>
      <OutputFiles Include="$(OutputPath)\**\*.*" Exclude="$(OutputPath)\Images\**\*.*" />
      <ImageFiles Include="$(OutputPath)\Images\**\*.*" />
      <ConfigFile Include="$(OutputPath)\Config.esriaddinx" />
    </ItemGroup>

    <Copy SourceFiles="@(OutputFiles)" DestinationFolder="$(TargetFolder)\Install\" />
    <Copy SourceFiles="@(ImageFiles)"  DestinationFolder="$(TargetFolder)\Images" />
    <Copy SourceFiles="@(ConfigFile)"  DestinationFiles="$(TargetFolder)\Config.xml" />

    <XmlPoke XmlInputPath="$(TargetFolder)\Config.xml"
             Query="$(TargetVersionQuery)"
             Value="$(TargetArcGISVersion)"
             Namespaces="$(EsriAddInNamespace)" />

    <ZipDirectory SourceDirectory="$(TargetFolder)"
                  DestinationFile="$(TargetAddInFile)"
                  Overwrite="true" />
  </Target>

</Project>
```

Use a batch file to manually invoke the build. For example:

```batch
@cd /d "%~dp0"

@set MSBUILD="C:\Program Files (x86)\Microsoft Visual Studio\2019\Professional\MSBuild\Current\Bin\msbuild.exe"

%MSBUILD% MyProject.build

@pause
```

- The first line changes to the .bat file's directory.
- The location of MSBuild varies with the VS version.
- The `pause` in the last line allows you to read
  output messages before the command window closes.

## Caveats

The automatic generation of *Config.Designer.cs* from
*Config.esriaddinx* will be gone. You have to maintain
this file manually. In my opinion, this is a small price
to pay. Indeed, I believe the dependency should be the
other way round: auto-generate *Config.esriaddinx* from
code and attributes.

The *Config.Designer.cs* file also contains some “static”
code. You may want to (1) start with the SDK to have it
generate this code, or (2) copy it from another project,
or (3) write it yourself.

You still need ESRI.ArcGIS.Desktop.AddIns.dll from the SDK.

The approach described so far will not build and deploy
the Add-In from within VS. However, this is useful during
development and testing. Use a VS post-build event to emulate
this behaviour of the SDK: create the Zip archive as shown
above, then copy it to the well-known location, which is:

`%USERPROFILE%\Documents\ArcGIS\AddIns\Desktop%ArcGISVersion%\%AddInID%\`

You could use the MSBuild XmlPeek task to extract the `AddInID`
from *Config.esriaddinx*. Here is a sketch:

``` xml
<XmlPeek XmlInputPath="...\Config.esriaddinx"
         Query="/*/x:AddInID[1]/text()"
         Namespaces="<Namespace Prefix='x' Uri='http://schemas.esri.com/Desktop/AddIns'/>">
  <Output TaskParameter="Result" ItemName="AddInIDItem" />
</XmlPeek>

<PropertyGroup>
  <!-- Make property from (hopefully) singleton item: -->
  <AddInID>@(AddInIDItem)</AddInID>
</PropertyGroup>

<Copy SourceFiles="path\to\built\AddInName.esriAddIn"
      DestinationFiles="$(USERPROFILE)\Documents\ArcGIS\AddIns\Desktop10.X\$(AddInID)\AddInName.esriAddIn" />
```

## Signing

You may use the ordinary *ESRISignAddIn.exe* utility
to sign your Add-Ins.

## Summary

We explained how to build an Add-In for ArcGIS Desktop (not Pro),
optionally for multiple versions of ArcGIS, and without using
the Desktop SDK. We did so because it is a common scenario to
provide an Add-In for multiple versions of ArcGIS, and because
the SDK regularly fails to work after upgrades to Visual Studio.
