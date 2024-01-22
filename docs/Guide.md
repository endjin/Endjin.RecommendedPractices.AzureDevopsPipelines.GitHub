# Endjin.RecommendedPractices.AzureDevopsPipelines.GitHub usage guide

This repository contains various Azure DevOps Pipelines templates. It standardizes the build and release mechanisms [endjin](https://endjin.com) uses.

To use these scripts, a repository will contain a pipeline definition file, e.g., `azure-pipelines.yml`, and it will contain a `resources` section referring to this repository:

```yaml
resources:
  repositories:
    - repository: recommended_practices
      type: github
      name: endjin/Endjin.RecommendedPractices.AzureDevopsPipelines.GitHub
      endpoint: corvus-dotnet-github
```

The `endpoint` refers to an Azure DevOps Service Connection. This must be a [GitHub connection](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints?view=azure-devops&tabs=yaml#github-service-connection). This service connection can be defined in the Azure DevOps project, or it can be an organization-scoped shared connection that has been made available to the project.

With that `repository` definition in place, the pipeline can refer to the templates this repository defines. For example, a job might look like this:

```yaml
jobs:
- template: templates/build.and.release.scripted.yml@recommended_practices
  parameters:
    vmImage: 'ubuntu-latest'
    solution_to_build: 'Solutions/Corvus.JsonSchema.sln'
```

The `template` setting ends with `@recommended_practices` which is a reference to the `repository` entry defined earlier in the file. It refers to this repository The part before the `@` symbol is a path within this repository.

All of the templates this repository defines are in the [`templates` folder](../templates/), and this particular example refers to the `build.and.release.scripted.yml` template. The following sections describe each of the templates in that folder.

## Scripted Build and Release

The [build.and.release.scripted.yml](../templates/build.and.release.scripted.yml) automates the building and, where required, release to NuGet. This is the preferred build and release mechanism. New repositories using endjin's recommended practices with Azure DevOps should define a single pipeline that uses this template. The advantage this template offers over the unscripted version is that it is easier to debug build problems. Developers can run the build script on their local machines to execute the build, and since this pipeline runs the same script, the behaviour is likely to be the same. (There can of course be environmental differences between build agents and development boxes, but by using the same script, differences in behaviour are minimized.)

One downside of this scripted approach is that Azure DevOps has less insight into what our builds are doing. Because build, test, and packaging are all performed by running a script, we use command line tasks to execute these phases of the build, and not the tasks that Azure DevOps defines for these jobs. But we have to do this if we want developers to be able to execute builds locally in the same way that they run on the build agents, because Azure DevOps provides no tooling to enable us to execute the work those more specialized steps perform on a developer's machine. A script is the only way we can ensure that the build runs in the same way on the build agent and locally.

### Repository setup

To use the scripted build and release template, you will need to define an Azure DevOps pipeline and point it at a pipeline definition file. This file will need to include a `repository` element as shown earlier, and also an entry in `jobs` referring to this template. Here's a typical example:

```yaml
trigger:
  branches:
    include:
    - main
  tags:
    include:
    - '*'

resources:
  repositories:
    - repository: recommended_practices
      type: github
      name: endjin/Endjin.RecommendedPractices.AzureDevopsPipelines.GitHub
      endpoint: corvus-dotnet-github

jobs:
- template: templates/build.and.release.scripted.yml@recommended_practices
  parameters:
    vmImage: 'ubuntu-latest'
    service_connection_nuget_org: $(Endjin_Service_Connection_NuGet_Org)
    service_connection_github: $(Endjin_Service_Connection_GitHub)
    solution_to_build: 'Solutions/Corvus.JsonSchema.sln'
    additionalNetSdkVersions:
    - '7.x'
    - '8.x'
    compileTasksServiceConnection: endjin-acr-reader
```

The `trigger` section ensures that the pipeline runs any time there is a commit to `main`, and also time a tag is applied to any branch. Some repositories also include a `feature/*` trigger so that an automated build runs whenever changes are pushed to a feature branch, enabling you to discover whether you've got build failures before you create a PR. However most endjin repos also use a system called [PR autoflow](https://github.com/endjin/pr-autoflow), which means that builds run automatically once you create a PR, and many repos choose not to include the `feature/*` trigger because that ends up causing each push to a PR branch to build twice.

The `jobs` section typically includes just a single entry, a `template`-based job referring to the scripted build and release template. This template requires various parameters. At a minimum, it needs us to specify which agent VM to use (e.g. `windows-latest` or `ubuntu-latest`). It requires the Azure DevOps project to define service connections for GitHub and NuGet, and we specify the names of these connections as parameters. It also needs the path to the `.sln` file to be build.

There are optional parameters. This example has asked for some specific .NET SDKs to be installed.

TODO: I have no idea what `compileTasksServiceConnection` is for or whether it's universally required.

This template requires a `build.ps1` file to be present in the root of the repository. The means by which we add and update this is a little opaque. If you look at a typical endjin repository using this mechanism, you'll see from the [history for the `build.ps1` file](https://github.com/corvus-dotnet/Corvus.Testing/commits/main/build.ps1) that the file is regularly updated by an automated account called [`dependjinbot`](https://github.com/apps/dependjinbot).

These automated updates are intended to resemble those created by GitHub's own [Dependabot](https://github.com/dependabot). However, whereas Dependabot's operation is configured through a GitHub repository's settings, repositories that participate in [`dependjinbot`](https://github.com/apps/dependjinbot) updates offer no clues as to why `dependjinbot` knows it's meant to keep this up to date. PRs updating this file are created by `dependjinbot`, then automatically approved (apparently by the [`github-actions` bot](https://github.com/apps/github-actions) itself) and finally automatically merged (by `dependjinbot`). But you will find nothing in a repository's contents or configuration that can account for why that repository is receiving automated updates from [`dependjinbot`](https://github.com/apps/dependjinbot).

The reason it's basically impossible to work out why this happens just by looking at a repository where it does happen is because `dependjinbot` gets its instructions from a completely separate private repository containing. If you work for endjin, you'll be able to see the configuration that drives this process at https://github.com/endjin/endjin-codeops/tree/main/repo-level-processes/config/live and you can enroll a repository for these automated updates by adding it to any file in that folder. The [endijn/endjin-codeops](https://github.com/endjin/endjin-codeops) repository has a GitHub action that runs every night that inspects the list of repositories that are subject to these automated updates, and it ensures the `build.ps1` script is up to date.

What if you want to use this `build.and.release.scripted.yml` template without enrolling your repository for these automated updates? (Perhaps you don't work for endjin, meaning this isn't an option.) It is possible, but you need to be aware that there's an assumption that all repositories using the `build.and.release.scripted.yml` pipeline template are using the latest `build.ps1`. We often find it necessary to tweak the build process as build agents change over time. If your Azure DevOps pipeline definition refers to this template, you will get the latest behaviour each time, which means that if you don't keep `build.ps1` up to date, things may break. (It is possible for template references to include a commit hash, enabling you to pin your pipeline to a particular version of the template. But be aware that we don't maintain support for any older versions. The basic assumption is that if you're using this mechanism, you will always be on the latest template and the latest `build.ps1`.)

The `build.ps1` file can be found at https://github.com/endjin/Endjin.RecommendedPractices.Build.

If you want to run this script on your local development box, be aware that it depends on the [`Endjin.RecommendedPractices.Build` module](https://www.powershellgallery.com/packages/Endjin.RecommendedPractices.Build/) in the PowerShell gallery. The source for this is in the same [repo](https://github.com/endjin/Endjin.RecommendedPractices.Build) as the `build.ps1` file. The repository's readme describes how to install the module, and also how to use `build.ps1` itself.


TBD: setting up variables in the AzDevOps pipeline config?

### Using the scripted build and release

To publish a preview to NuGet off a PR branch, run the build setting Endjin.ForcePublish to true.

Tagging typically done by PR autoflow...I think.


## Build and Release (without script)

The [](../templates/build.and.release.yml)

## Tag for Release

The [](../templates/tag.for.release.yml)

Useful if you've not bought into PR autoflow?


## Files used internally by other templates

The following files are not normally used directly by other repositories. They are used by other templates in this repo

| Script | Purpose |
|---|---|
| [install-and-run-gitversion.yml](../templates/install-and-run-gitversion.yml) | |
| [install-dotnet-sdks.yml](../templates/install-dotnet-sdks.yml) | |
| [install.dotnet-global-tools.workaround.yml](../templates/install.dotnet-global-tools.workaround.yml) | Used by non-scripted build to work around what appeared to be an Azure DevOps bug back in the .NET 5.0 days; probably no longer necessary, but we don't use the non-scripted builds any more,  |
| [](../templates) | |


## Scripted Build and Release

The [](../templates/install-dotnet-sdks.yml)


## Scripted Build and Release

The [](../templates/install-dotnet-sdks.yml)


## Scripted Build and Release

The [](../templates/install.dotnet-global-tools.workaround.yml)


## Scripted Build and Release

The [](../templates/tag.for.release.yml)
