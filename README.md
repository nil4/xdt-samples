# XDT samples for .NET Core/ASP.NET Core

This repository contains examples of applying [XML Document Transformation](https://msdn.microsoft.com/en-us/library/dd465326.aspx)
to ASP.NET Core projects, using [`dotnet-transform-xdt`](https://github.com/nil4/dotnet-transform-xdt).

`dotnet-transform-xdt` is a generic [dotnet CLI](https://github.com/dotnet/cli) that transforms XML
files. It is typically used on ASP.NET `Web.config` files at publish time, but can be used for any purpose.
It is a port of <http://xdt.codeplex.com/> compatible with [.NET Core](http://dotnet.github.io/).

This repository contains samples for two use cases:

 1. [Remove aspNetCore handlers from Web.config when publishing](#remove-handlers)
 2. [Configure IIS settings for production deployments](#production-iis-config)

Read more details below, or see the commit history for each of the sample app sub-folders.

### <a name="remove-handlers"></a> 1. Remove system.webServer/handlers from Web.config

When publishing an ASP.NET Core project, either from the command line (`dotnet publish`) or inside Visual Studio,
the tooling adds the following to the published `Web.config` file, even if the developer removes these tags from the
`Web.config` file included in their project:

```xml
<system.webServer>
  <handlers>
    <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModule" resourceType="Unspecified"/>
  </handlers>
  <aspNetCore processPath="%LAUNCHER_PATH%" arguments="%LAUNCHER_ARGS%" stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout" forwardWindowsAuthToken="false"/>
</system.webServer>
```

This is fine for most standalone applications. However, for those that want to host their ASP.NET Core application
as an IIS sub-application or child site, this does not work, because IIS only allows handlers to be defined by the
top-level site/application. Example issues: https://github.com/dotnet/sdk/issues/462 and
https://github.com/aspnet/DotNetTools/issues/175.

Here is how you can use `dotnet-transform-xdt` to remove the `<handlers>` section on publish. Create a
`Web.RemoveHandlers.config` file next to the `Web.config` file in your project, and set its content to:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration xmlns:xdt="http://schemas.microsoft.com/XML-Document-Transform">
  <system.webServer>
    <handlers xdt:Transform="Remove" />
  </system.webServer>
</configuration>
```

Note the `xdt:Transform="Remove"` attribute on the `<handlers>` element. This requests that the element
be removed when applying this XDT transform file. See the [MSDN XDT reference](https://msdn.microsoft.com/en-us/library/dd465326.aspx)
for the full transformation options and syntax.

Now, edit your .csproj file and add:

```xml
<ItemGroup>
    <DotNetCliToolReference Include="Microsoft.DotNet.Xdt.Tools" Version="2.0.0-preview1" />
</ItemGroup>

<Target Name="RemoveHandlersFromWebConfig" AfterTargets="_TransformWebConfig">
  <PropertyGroup>
    <_SourceWebConfig>$(PublishDir)Web.config</_SourceWebConfig>
    <_XdtTransform>$(MSBuildThisFileDirectory)Web.RemoveHandlers.config</_XdtTransform>
    <_TargetWebConfig>$(PublishDir)Web.config</_TargetWebConfig>
  </PropertyGroup>
  <Exec Command="dotnet transform-xdt --xml &quot;$(_SourceWebConfig)&quot; --transform &quot;$(_XdtTransform)&quot; --output &quot;$(_TargetWebConfig)&quot;" />
</Target>
```

The `<DotNetCliToolReference>` element adds the `dotnet-transform-xml` tool to your project. You need to
run `dotnet restore` after making this change to install the tool from nuget.org. After restore, you can run
`dotnet transform-xml` in your project folder to see the available tool options.

The `<Target>` element defines an MSBuild target that runs *after* the built-in publish target that adds
the `aspNetCore` handler (i.e. `AfterTargets="_TransformWebConfig"`). It invokes the `dotnet-transform-xdt`
tool, applying the file we defined earlier (`Web.RemoveHandlers.config`) to the `Web.config` file in the publish folder,
removing the `<handlers>` section, and then writing the transformed result back to the publish folder.

Note that the `Web.config` file in your project is not changed; only the published file has the transformation
applied to it.

To test this, run `dotnet publish` from your project folder (or publish your project from within
Visual Studio 2017). Examine the `Web.config` file in your publish folder (`bin\Debug\netcoreapp2.0\publish\Web.config`
if you used `dotnet publish`); it should look like this:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <aspNetCore processPath="dotnet" arguments=".\RemoveAspNetCoreHandlerSample.dll" stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout" forwardWindowsAuthToken="false" />
  </system.webServer>
</configuration>
```

### <a name="production-iis-config"></a> 2. Configure IIS settings for production deployments

IIS configuration settings for production deployments are often different than the settings we use on our local
PCs when developing applications. Examples include authentication methods, request limits (e.g. file upload sizes),
HTTPS settings or [IIS URL Rewrite rules](https://www.iis.net/downloads/microsoft/url-rewrite).

ASP.NET Core removed the need for many Web.config settings that were previously required (e.g. everything under `<system.web>`),
however IIS settings are not among these; many values under [`<system.webServer>`](https://www.iis.net/configreference/system.webserver)
are still relevant.

These IIS settings can vary between environments, and with ASP.NET Core today there isn't a good solution in place.
There are however [plans to add official XDT support](https://github.com/aspnet/Tooling/issues/780) to support these scenarios.

This sample shows how to add fictitious production-specific IIS settings without impacting local development.
Add a new file (`Web.Release.config`) to your project, next to your `Web.config`. We will then configure
your project to only apply this XDT transform when explicitly publishing for the `Release` configuration.

Assuming our production server is set up to serve HTTPS, we will insert three IIS settings:

```xml
<?xml version="1.0"?>
<configuration xmlns:xdt="http://schemas.microsoft.com/XML-Document-Transform">
  <system.webServer>
    <security xdt:Transform="Insert">
      <requestFiltering>
        <!-- Set max request size to 10 MB -->
        <requestLimits maxAllowedContentLength="10485760" />
      </requestFiltering>
    </security>
    <httpProtocol xdt:Transform="Insert">
      <customHeaders>
        <!-- Instruct browsers to always use HTTPS when connecting to this site (https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security) -->
        <add name="Strict-Transport-Security" value="max-age=3840; includeSubDomains" />
      </customHeaders>
    </httpProtocol>
    <rewrite xdt:Transform="Insert">
      <rules>
        <clear />
        <!-- Redirect any HTTP requests to the same URL but over HTTPS -->
        <rule name="Force HTTPS" stopProcessing="true">
          <match url="(.*)" />
          <conditions>
            <add input="{HTTPS}" pattern="off" />
          </conditions>
          <action type="Redirect" url="https://{HTTP_HOST}/{R:1}" appendQueryString="true" redirectType="Permanent" />
        </rule>
      </rules>
    </rewrite>
    </system.webServer>
</configuration>
```

Note the `xdt:Transform="Insert"` attribute on the three sections. The first limits the maximum request size to 10 MB.

The second adds a response header that instructs clients to always connect over HTTPS
(see [`Strict-Transport-Security` on MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security) for details).

The third accounts for legacy clients that do not understand the `Strict-Transport-Security` header;
it uses an [IIS URL Rewrite module rule](https://www.iis.net/learn/extensions/url-rewrite-module/url-rewrite-module-configuration-reference)
to permanently redirect any HTTP request to its HTTPS equivalent.

Now, edit your .csproj file and add:

```xml
<ItemGroup>
    <DotNetCliToolReference Include="Microsoft.DotNet.Xdt.Tools" Version="2.0.0-preview1" />
</ItemGroup>

<Target Name="ApplyXdtTransform" BeforeTargets="_TransformWebConfig">
  <PropertyGroup>
    <_SourceWebConfig>$(MSBuildThisFileDirectory)Web.config</_SourceWebConfig>
    <_XdtTransform>$(MSBuildThisFileDirectory)Web.$(Configuration).config</_XdtTransform>
    <_TargetWebConfig>$(PublishDir)Web.config</_TargetWebConfig>
  </PropertyGroup>
  <Exec
    Command="dotnet transform-xdt --xml &quot;$(_SourceWebConfig)&quot; --transform &quot;$(_XdtTransform)&quot; --output &quot;$(_TargetWebConfig)&quot;"
    Condition="Exists('$(_XdtTransform)')" />
</Target>
```

The `<DotNetCliToolReference>` element adds the `dotnet-transform-xml` tool to your project.
You need to run `dotnet restore` after making this change to install the tool.

The `<Target>` element defines an MSBuild target that runs *before* the built-in publish target that adds
the `aspNetCore` handler (i.e. `BeforeTargets="_TransformWebConfig"`). We want this target to run earlier
because the built-in target (`_TransformWebConfig`) then replaces the `<aspNetCore>` element `processPath` and
`arguments` attribute placeholders with actual values, and also formats the published Web.config nicely indented.

Our `ApplyXdtTransform` target invokes the `dotnet-transform-xdt` tool,
applying the transform file for the active publish configuration (`Web.$(Configuration).config`) to the `Web.config` file
in your project, but only if that transform file actually exists. That is, publishing for the `Release` configuration will
apply the `Web.Release.config` transform, but publishing for `Debug` configuration will do nothing, since we don't have
a `Web.Debug.config` file.

It's worth noting that any other MSBuild property (built-in or passed through `dotnet publish /p:CustomProp=Value`)
can be used to determine the XDT transform file to apply.

To test this, run `dotnet publish` from your project folder (by default the `Debug` configuration is used and thus
no transformation is applied). Examine the `Web.config` file in your publish folder (`bin\Debug\netcoreapp2.0\publish\Web.config`);
it will be the same as `Web.config` in your project.

Now publish using `dotnet publish -c Release` and examine the `Web.config` file in your publish folder
(`bin\Release\netcoreapp2.0\publish\Web.config`); it should contain the production settings,
including the comments we added:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <handlers>
      <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModule" resourceType="Unspecified" />
    </handlers>
    <aspNetCore processPath="dotnet" arguments=".\WebServerSettingsSample.dll" stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout" forwardWindowsAuthToken="false" />
    <security>
      <requestFiltering>
        <!-- Set max request size to 10 MB -->
        <requestLimits maxAllowedContentLength="10485760" />
      </requestFiltering>
    </security>
    <httpProtocol>
      <customHeaders>
        <!-- Instruct browsers to always use HTTPS when connecting to this site (https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security) -->
        <add name="Strict-Transport-Security" value="max-age=3840; includeSubDomains" />
      </customHeaders>
    </httpProtocol>
    <rewrite>
      <rules>
        <clear />
        <!-- Redirect any HTTP requests to the same URL but over HTTPS -->
        <rule name="Force HTTPS" stopProcessing="true">
          <match url="(.*)" />
          <conditions>
            <add input="{HTTPS}" pattern="off" />
          </conditions>
          <action type="Redirect" url="https://{HTTP_HOST}/{R:1}" appendQueryString="true" redirectType="Permanent" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
```

### Feedback and reporting issues

Please [log an issue](https://github.com/nil4/dotnet-transform-xdt/issues) to send your feedback or report issues
you encounter when using `dotnet-transform-xdt`.
