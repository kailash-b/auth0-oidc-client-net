# .NET SDK 10.0.100 Marks net8.0-ios as EOL and Prevents Building

## Issue Summary

When building a project targeting `net8.0-ios` with .NET SDK 10.0.100, the build fails with error **NETSDK1135** because the SDK forcibly marks net8.0-ios as End of Life (EOL) and sets `TargetPlatformVersion` to an invalid value of `1.0`.

## Environment

- **.NET SDK Version**: 10.0.100
- **Workload Version**: 10.0.100 (specified in global.json)
- **Target Framework**: net8.0-ios
- **iOS Workload Pack**: Microsoft.iOS.Sdk.net10.0_26.0 (version 26.0.11017)
- **Operating System**: Windows
- **Global.json Configuration**:
  ```json
  {
    "sdk": {
      "version": "10.0.100",
      "rollForward": "latestPatch",
      "workloadVersion": "10.0.100"
    }
  }
  ```
- **Directory.Build.props** (attempted workaround):
  ```xml
  <PropertyGroup>
    <CheckEolWorkloads>false</CheckEolWorkloads>
    <CheckEolTargetFramework>false</CheckEolTargetFramework>
  </PropertyGroup>
  ```

## Build Error

```
C:\Program Files\dotnet\sdk\10.0.100\Sdks\Microsoft.NET.Sdk\targets\Microsoft.NET.TargetFrameworkInference.targets(252,5): 
error NETSDK1135: SupportedOSPlatformVersion 14.0 cannot be higher than TargetPlatformVersion 1.0.
```

## Root Cause Evidence

### 1. EOL Targets File Executed

From the detailed build log, the EOL targets file is imported and executed for net8.0-ios:

```
C:\Program Files\dotnet\packs\Microsoft.iOS.Sdk.net10.0_26.0\26.0.11017\targets\Microsoft.Sdk.Eol.targets(23,3): 
message : Property reassignment: $(WarningsAsErrors)=";NU1605;NETSDK1202" (previous value: ";NU1605")

C:\Program Files\dotnet\packs\Microsoft.iOS.Sdk.net10.0_26.0\26.0.11017\targets\Microsoft.Sdk.Eol.targets(30,3): 
message : Property reassignment: $(TargetPlatformVersion)="1.0" (previous value: "")
```

### 2. EOL Targets File Content

The `Microsoft.Sdk.Eol.targets` file located at:
```
C:\Program Files\dotnet\packs\Microsoft.iOS.Sdk.net10.0_26.0\26.0.11017\targets\Microsoft.Sdk.Eol.targets
```

Contains the following logic:

```xml
<!--
Imported only when using a .NET version that has reached EOL, contains settings to force the error:

error NETSDK1202: The workload 'ios' is out of support and will not receive security updates in the future. 
Please refer to https://aka.ms/maui-support-policy for more information about the support policy.

Things to note:
* $(WarningsAsErrors) includes NETSDK1202, so that the build stops
* Force $(TargetPlatformVersion) to just be 1.0, to make it past early error messages
* _ClearMissingWorkloads prevents undesired extra error messages
-->
<Project InitialTargets="_ClearMissingWorkloads">
    <PropertyGroup>
        <TargetPlatformSupported>true</TargetPlatformSupported>
        <WarningsAsErrors>$(WarningsAsErrors);NETSDK1202</WarningsAsErrors>
        <TargetPlatformVersion>1.0</TargetPlatformVersion>
    </PropertyGroup>
    <ItemGroup>
        <EolWorkload Include="$(TargetFramework)" Url="https://aka.ms/maui-support-policy" />
    </ItemGroup>
    <!-- ... -->
</Project>
```

### 3. Comparison with net9.0-ios and net10.0-ios

The same build successfully compiles `net9.0-ios` and `net10.0-ios` targets:

**net9.0-ios**: Uses `Microsoft.iOS.Sdk.net9.0_26.0` pack, which does **not** include EOL targets
```
C:\Program Files\dotnet\packs\Microsoft.iOS.Sdk.net9.0_26.0\26.0.9766\targets\Xamarin.Shared.Sdk.TargetFrameworkInference.props(15,5): 
message : Property reassignment: $(TargetPlatformVersion)="18.0" (previous value: "")
```
**Result**: ✅ Build succeeds

**net10.0-ios**: Uses `Microsoft.iOS.Sdk.net10.0_26.0` pack
```
C:\Program Files\dotnet\packs\Microsoft.iOS.Sdk.net10.0_26.0\26.0.11017\targets\Xamarin.Shared.Sdk.TargetFrameworkInference.props(15,5): 
message : Property reassignment: $(TargetPlatformVersion)="26.0" (previous value: "")
```
**Result**: ✅ Build succeeds

**net8.0-ios**: Uses `Microsoft.iOS.Sdk.net10.0_26.0` pack with EOL targets
```
C:\Program Files\dotnet\packs\Microsoft.iOS.Sdk.net10.0_26.0\26.0.11017\targets\Microsoft.Sdk.Eol.targets(30,3): 
message : Property reassignment: $(TargetPlatformVersion)="1.0" (previous value: "")
```
**Result**: ❌ Build fails with NETSDK1135

### 4. Build Log Sequence

The critical sequence from the build log shows:

1. **Initial TargetPlatformVersion set to empty**:
   ```
   Microsoft.NET.TargetFrameworkInference.targets(81,5): $(TargetPlatformVersion)="" (previous value: "0.0")
   ```

2. **EOL Targets forcibly override to "1.0"**:
   ```
   Microsoft.Sdk.Eol.targets(30,3): $(TargetPlatformVersion)="1.0" (previous value: "")
   ```

3. **SDK propagates the invalid value**:
   ```
   Microsoft.NET.Sdk.targets(1502,5): $(EffectiveTargetPlatformVersion)="1.0" (previous value: "")
   ```

4. **Validation fails**:
   ```
   Target "_CheckForSupportedOSPlatformVersionHigherThanTargetPlatformVersion" FAILED
   error NETSDK1135: SupportedOSPlatformVersion 14.0 cannot be higher than TargetPlatformVersion 1.0.
   ```

## Attempted Workarounds (All Failed)

1. ✗ Setting `CheckEolWorkloads=false` in Directory.Build.props (only suppresses warnings, not the EOL targets execution)
2. ✗ Setting `CheckEolTargetFramework=false` in Directory.Build.props (property not recognized)
3. ✗ Explicitly setting `TargetPlatformVersion` in project PropertyGroup (overridden by EOL targets)
4. ✗ Using MSBuild Target with `BeforeTargets="PrepareForBuild"` (EOL targets execute earlier during SDK import)
5. ✗ Creating local Directory.Build.props with overrides (interferes with root configuration)

## Impact

This prevents building .NET 8 iOS applications or libraries with .NET SDK 10, even though:
- .NET 8 is still in **Standard Support** until **November 2026**
- Many projects need to maintain compatibility with .NET 8 for their users
- The same workloads (iOS SDK 26.0) work fine for net9.0-ios and net10.0-ios

## Expected Behavior

**Expected**: Projects targeting `net8.0-ios` should build successfully with .NET SDK 10.0.100, as .NET 8 is still within its supported lifecycle.

**Actual**: Build fails with NETSDK1135 because the iOS workload pack forcibly sets `TargetPlatformVersion="1.0"` for net8.0-ios targets.

## Possible Solutions

1. **Remove or conditionally exclude EOL targets** for net8.0-ios in the `Microsoft.iOS.Sdk.net10.0_26.0` workload pack
2. **Provide a property to disable EOL enforcement** (beyond just warning suppression)
3. **Allow explicit TargetPlatformVersion override** to take precedence over EOL targets
4. **Document that net8.0-ios requires SDK 9** and cannot be built with SDK 10 (breaking change)

## Related Files

- Full build log: `build-net8-ios-evidence.log`
- EOL Targets: `C:\Program Files\dotnet\packs\Microsoft.iOS.Sdk.net10.0_26.0\26.0.11017\targets\Microsoft.Sdk.Eol.targets`
- Project file: `Auth0.OidcClient.iOS.csproj`

## Question for Microsoft

Is the EOL enforcement for net8.0-ios intentional in .NET SDK 10? If so:
- Why is net8.0-ios marked as EOL when .NET 8 itself has Standard Support until November 2026?
- What is the recommended approach for libraries that need to support both .NET 8 and .NET 10 users?
- Should we use multi-stage builds with different SDK versions?
