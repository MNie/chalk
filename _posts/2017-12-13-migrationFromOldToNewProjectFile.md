---
layout: post
title: Migration from old format of cs/fsproj to the new one
description: "Migration from .net framework csproj/fsproj to the .net core csproj/fsproj isn't an easy task. There are some caveats around it, checkout article to find some of them."
thumb_image: "documentation/sample-image.jpg"
tags: [fsharp, microservices, csharp, netcore, netframework, patterns]
---

Hey,

recently, in my work I encountered problem of adding a nuget packege which targets .net standard to a project which targets .net framework. This caused problems when compiling the project.
In search of a possible solution I found an [issue on github](https://github.com/dotnet/standard/issues/410).

Due to this, I decided to migrate old csproj/fsproj files in libraries and console apps (windows services).
However, in project which contains msbuild tasks inside of a project file, or web projects, I decided to switch from operating on packages.config and project file for nuget packages to a packagereference in project file.
Each of such packagereference contains information about the version and name of a package.

I started by moving information about nuget packages from packages.config files + cs/fsprojs to packagereferences in cs/fsproj. It based on adding the following entry to the csproj file:

{% highlight xml %}
<RestoreProjectStyle>
    PackageReference
</RestoreProjectStyle>
{% endhighlight %}

Then I copied the content of the packages.config file and rewrote it to a single command for Package Manager Console to install all the nuget packages, you can do it with the following script, which gets the name and version of the package from the file whose path I pass as the first argument for the project which name I pass as the second argument.

<script src="https://gist.github.com/MNie/35f43e66e68d300510ca3a6c308d06fd.js"></script>

Since the command is ready, I could delete the packages.config file, at this moment I'm ready to execute the previously prepared command. After doing this, all nuget package references should be in one place at the bottom of project file and should looks like this:

{% highlight xml %}
<ItemGroup>
    <PackageReference Include="Ben.Demystifier">
    <Version>0.0.5</Version>
</PackageReference>
{% endhighlight %}

Moving to the conversion from the old format of fsproj file to the new one, I decided to prepare a minimal set of xml elements for the new format, which I pasted at the very beginning of the existing file.

{% highlight xml %}
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net462</TargetFramework>
    <RootNamespace>DD.AA</RootNamespace>
    <AssemblyName>DD.AA</AssemblyName>
    <Name>DD.AA</Name>
    <OutputType>Library</OutputType>
    <AppDesignerFolder>Properties</AppDesignerFolder>
    <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
  </PropertyGroup>

  <ItemGroup>
    <Compile Include="..\GlobalAssemblyInfo.cs">
      <Link>Properties\GlobalAssemblyInfo.cs</Link>
    </Compile>
    <Compile Include="..\VersionInfo.cs">
      <Link>Properties\VersionInfo.cs</Link>
    </Compile>
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Util.Lib" Version="1.0.0.*" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\OtherProjectInSln.csproj" />
  </ItemGroup>

  <ItemGroup>
    <Reference Include="Reference to some windows libs" />
  </ItemGroup>
</Project>
{% endhighlight %}

Then I changed the name of the project, assembly and default namespace. I copied the itemgroup section containing information about the files included in the project. Then I added all the references to the nuget packages as the packagereference section.
I delete the rest of the file, thanks to which the file finally takes the following form.

{% highlight xml %}
<Project Sdk="FSharp.NET.Sdk;Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net462</TargetFramework>
    <RootNamespace>DD.Parser</RootNamespace>
    <AssemblyName>DD.Parser</AssemblyName>
    <Name>DD.Parser</Name>
    <OutputType>Library</OutputType>
  </PropertyGroup>

  <ItemGroup>
    <Compile Include="AssemblyInfo.fs" />
    <Compile Include="SomeFile.fs" />
    <Content Include="App.config" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="FSharp.NET.Sdk" Version="1.0.*" PrivateAssets="All" />
    <PackageReference Include="FSharp.Core" Version="4.1.*" />
    <PackageReference Include="Castle.Common.Lang" Version="3.0.13.*" />
    <PackageReference Include="FSharp.Data" Version="2.4.*" />
    <PackageReference Include="System.ValueTuple" Version="4.3.*" />
    <PackageReference Include="FSharp.Core" Version="4.1.*" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\DD.Contract\DD.Contract.csproj" />
  </ItemGroup>

</Project>
{% endhighlight %}

Now I am going to rewrite the csproj files, here I should point out one advantage of this migration in relation to f#, I do not have to specify files in the itemgroup section, but in this case I encountered some problems with files that remained in the repository, but were unplugged from the project, through which after changing to a new format appeared again in the project.

Simply overwriting projects for a new format or using packagesreferences instead of packages.config took some time, but it was rather a simple task. I could use programs that do it automatically for me, for example like [this](https://github.com/hvanbakel/CsprojToVs2017).

However, I preferred to do it on myself to have full control over what is happening in the file.
So, what are the advantages of switching to a new format?

* Information about the nuget packages is in one place, and not as it was before in packages.config what could cause problems when someone installed the package, but did not rebuild the project/did not save it and push their changes into the repository,

{% highlight xml %}
  <ItemGroup>
    <PackageReference Include="Newtonsoft.Json" Version="6.0.*" />
  </ItemGroup>
{% endhighlight %}

* We do not have to worry about whether the attached package is oriented to a good framework, because its version is deduced from the framework set in the project file,
* Transparency and readability of the file itself, thanks to which we can finally easily operate on it,

{% highlight xml %}
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net462</TargetFramework>
    <RootNamespace>DD.Database</RootNamespace>
    <AssemblyName>DD.Database</AssemblyName>
    <Name>DD.Database</Name>
    <OutputType>Library</OutputType>
    <AppDesignerFolder>Properties</AppDesignerFolder>
    <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Common" Version="1.0.0.*" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\OtherProjectInSln.csproj" />
  </ItemGroup>

  <ItemGroup>
    <Reference Include="Reference to some windows libs" />
  </ItemGroup>
</Project>
{% endhighlight %}

* The length of the file itself has dropped from about 200-1000 lines to 20-80 (in my solution),
![length](https://mnie.github.com/img/13-12-2017CsProjMigration/length.png)
* We can easily specify the range of packages that should be downloaded for a given project,

{% highlight xml %}
<ItemGroup>
    <PackageReference Include="Common" Version="1.0.*" />
</ItemGroup>
{% endhighlight %}

* We do not have to unload the project in Visual Studio to edit the project file,
![unload](https://mnie.github.com/img/13-12-2017CsProjMigration/unload.png)
* In the case of csproj files, we do not have to specify all files that are part of the project,
* It gives us instant information about bad versions of packages between projects,
![downgrade](https://mnie.github.com/img/13-12-2017CsProjMigration/downgrade.png)
* Quick information about cycles detection in project references,
![cycle](https://mnie.github.com/img/13-12-2017CsProjMigration/cycle.png)
* "Easy" migration to the .net core, potentially the only thing we have to do is to add a new targetframework,

{% highlight xml %}
<TargetFramework>net462;netstandard2.0;and so on</TargetFramework>
{% endhighlight %}

* Generating packages from the visual studio during the build process,
![generate](https://mnie.github.com/img/13-12-2017CsProjMigration/generate.png)

{% highlight xml %}
<PropertyGroup>
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild>
</PropertyGroup>
{% endhighlight %}

* Packages are downloaded per machine, not per solution as it was before.


Problems that I encountered while migrating?
* Generating the assemblyinfo based on the linked global files in the entire repository. In the case of my project, projects within the repository may have the same versions, by which they linked the same fileinfo and versioninfo files, in the case of a new csproj format the asseblyinfo file is automatically generated. To disable automatic generation, you have to add the following line at the beginning of the file csproj,

{% highlight xml %}
<PropertyGroup>
    <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
</PropertyGroup>
{% endhighlight %}

* Generating nuget packages using the nuget client from libraries based on the new project file format is [not yet possible](https://github.com/NuGet/Home/issues/4491),
* Migration of msbuild tasks is available, but not so [well documented](https://github.com/Microsoft/msbuild/issues/2746).
    - So if you have an xml like that:

    {% highlight xml %}
    <UsingTask TaskName="TransformXml" AssemblyFile="$(MSBuildExtensionsPath32)\Microsoft\VisualStudio\v$(VisualStudioVersion)\Web\Microsoft.Web.Publishing.Tasks.dll" />
    <Target Name="AfterCompile" Condition="Exists('App.$(Configuration).config')">
      <!--Generate transformed app config in the intermediate directory-->
      <TransformXml Source="App.config" Destination="$(IntermediateOutputPath)$(TargetFileName).config" Transform="App.$(Configuration).config" />
      <!--Force build process to use the transformed configuration file from now on.-->
      <ItemGroup>
        <AppConfigWithTargetPath Remove="App.config" />
        <AppConfigWithTargetPath Include="$(IntermediateOutputPath)$(TargetFileName).config">
          <TargetPath>$(TargetFileName).config</TargetPath>
        </AppConfigWithTargetPath>
      </ItemGroup>
    </Target>
    {% endhighlight %}

    - you have to migrate it to something like that

    {% highlight xml %}
    <UsingTask TaskName="TransformXml" AssemblyFile="$(MSBuildExtensionsPath32)\Microsoft\VisualStudio\v$(VisualStudioVersion)\Web\Microsoft.Web.Publishing.Tasks.dll" />
    <Target Name="ApplyConfigurationConfigFile" AfterTargets="PrepareForBuild" Condition="Exists('App.$(Configuration).config')">
      <ItemGroup>
        <AppConfigWithTargetPath Remove="App.config" />
        <AppConfigWithTargetPath Include="$(IntermediateOutputPath)$(TargetFileName).config">
          <TargetPath>$(TargetFileName).config</TargetPath>
        </AppConfigWithTargetPath>
      </ItemGroup>
      <TransformXml Source="App.config" Destination="$(IntermediateOutputPath)$(TargetFileName).config" Transform="App.$(Configuration).config" />
    </Target>
    {% endhighlight %}

* Blindly following the fact that the creators of the packages know, understand, respect and use the "Semantic Versioning" can lead us up the garden path
![sqlite](https://mnie.github.com/img/13-12-2017CsProjMigration/sqlite.png).
* Compiled sources no longer go to bin/* folders, instead they go to bin/{targerFramework}/* folders,
* Problems encountered with the Visual studio version, due to the fact that a version of at least 15.2 is required to make everything work well,
* Problem with resharper build, which can not cope the new csproj file format or references to nuget packages, which could be found only in project files,
* Migration of web projects targeting the .net framework to the new framework is not possible at the moment,
* Problem with detection of Machine.Specifications/XUnit specs by Resharper [issue](https://github.com/machine/machine.specifications.runner.resharper/issues/73), and [issue in resharper](https://youtrack.jetbrains.com/issue/RSRP-467721).

In summary, despite the large number of minuses, I think that the advantages prevail in the new project file format, thanks to which these projects are easier to manage and the files themselves are more concise and clear.

Thank you for reading :)

[Here is a presentation which I gave to my teammates](https://mnie.github.io/CsProjMigration/index#/)
