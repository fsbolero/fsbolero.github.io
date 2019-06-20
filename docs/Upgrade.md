---
title: Upgrade guide
subtitle: How to update a project for newer releases
---

### From v0.4 to v0.5

![Blazor 3.0](../img/blazor-icon.png)

Bolero 0.5 doesn't bring any breaking API changes.
However, it upgrades the dependency on Blazor from 0.7 to 3.0-preview5; and by doing so, it also upgrades the dependency on .NET Core from 2.1 to 3.0-preview5.
This brings a number of necessary changes to projects. Here are the necessary steps:

* Install [.NET Core 3.0-preview5](https://dotnet.microsoft.com/download/dotnet-core).

* If your solution contains a `global.json` file with an explicit SDK version, either remove the file or update its SDK version to `3.0.100-preview5`.

* In `paket.dependencies`:
  * In the `framework` declaration, replace `netcoreapp2.1` with `netcoreapp3.0`.
  * Remove the following `nuget` lines:
    * `Microsoft.AspNetCore.App` (In ASP.NET Core 3.0, it is automatically included by the SDK)
    * `Microsoft.AspNetCore.Razor.Design`
    * `Microsoft.AspNetCore.SignalR`
  * Replace `clitool Microsoft.AspNetCore.Blazor.Cli` with `nuget Microsoft.AspNetCore.Blazor.DevServer`.
  * For all the `Microsoft.AspNetCore.Blazor*` and `Bolero*` dependencies, remove version constraints if there are any (such as `~> 0.7.0`), and add `prerelease` as a version constraint.

* In `src/*.Client/paket.references`:
  * Replace `Microsoft.AspNetCore.Blazor.Cli` with `Microsoft.AspNetCore.Blazor.DevServer`.

* In `src/*.Server/paket.references`:
  * Remove `Microsoft.AspNetCore.App`.

* In both `*.fsproj` files, replace the `TargetFramework` from `netcoreapp2.1` to `netcoreapp3.0`.

You can now run `.paket/paket update` to update the packages.
There are also a few code changes to apply:

* In `src/*.Client/Startup.fs`:
  * Replace `Microsoft.AspNetCore.Blazor.Builder` with `Microsoft.AspNetCore.Components.Builder`.
  * Replace `IBlazorApplicationBuilder` with `IComponentsApplicationBuilder`.

* In `src/*.Server/Startup.fs`:
  * Replace `IHostingEnvironment` with `IWebHostEnvironment`.

Your solution should now build and run successfully.
If your code uses Blazor APIs directly, please check the Blazor [0.8](https://devblogs.microsoft.com/aspnet/blazor-0-8-0-experimental-release-now-available/), [0.9](https://devblogs.microsoft.com/aspnet/blazor-0-9-0-experimental-release-now-available/) and [3.0](https://devblogs.microsoft.com/aspnet/blazor-now-in-official-preview/) upgrade guides for more code changes you might need to make.

If you are having any issues with this upgrade, don't hesitate to [post an issue on GitHub!](https://github.com/fsbolero/bolero/issues)
