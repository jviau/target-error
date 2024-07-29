# MSBuild Target Ordering Issue

## Issue

`dotnet build func.sln --no-incremental` results in an error due to some [targets](https://github.com/Azure/azure-functions-dotnet-worker/blob/main/sdk/Sdk/Targets/Microsoft.Azure.Functions.Worker.Sdk.targets) not running as expected relative to `CopyFilesToOutputDirectory`. The target is [_FunctionsInnerBuild](https://github.com/Azure/azure-functions-dotnet-worker/blob/ac91781ad96cf3bb136ef73c376bfc0c7f135d45/sdk/Sdk/Targets/Microsoft.Azure.Functions.Worker.Sdk.targets#L67) where we have set `AfterTargets="CoreCompile" BeforeTargets="CopyFilesToOutputDirectory" DependsOnTargets="... the targets we want to run ..."`. The goal is to ensure that list of targets in `DependsOnTargets` runs after `CoreCompile` and before `CopyFilesToOutputDirectory`.

The error is due to that _earlier_ in the build, we have this [target](https://github.com/Azure/azure-functions-dotnet-worker/blob/ac91781ad96cf3bb136ef73c376bfc0c7f135d45/sdk/Sdk/Targets/Microsoft.Azure.Functions.Worker.Sdk.targets#L95): `<Target Name="_FunctionsAssignTargetPaths" DependsOnTargets="_FunctionsGetPaths" BeforeTargets="AssignTargetPaths">`, which is adding to the `None` item group with `TargetPath` already set some files that we will _eventually_ create via the `_FunctionsInnerBuild` target.

The expectation is that we will have always had our targets ran before `CopyFilesToOutputDirectory` and those files added to `None` item group will exist by the time `CopyFilesToOutputDirectory` attempts to copy them. This works in every build tested except one: `dotnet build --no-incremental` on a solution file. On a project with these targets, or a project referencing a project with this targets with `--no-incremental` succeeds.

**Repro:** `dotnet build func.sln --no-incremental` <br />
**Success:** `dotnet build test/Isolated.Tests/Isolated.Tests.csproj --no-incremental`
