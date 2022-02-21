```
> dotnet --info
.NET SDK (reflecting any global.json):
 Version:   6.0.200
 Commit:    4c30de7899

Runtime Environment:
 OS Name:     Mac OS X
 OS Version:  12.0
 OS Platform: Darwin
 RID:         osx.12-x64
```
Let's begin
```
mkdir il-roundtrip && cd $_
dotnet new console --framework net5.0
dotnet build
ildasm bin/Debug/net5.0/il-roundtrip.dll -out=il-roundtrip.il
touch il-roundtrip.ilproj
dotnet new globaljson
```
TODO: where is `ildasm` on mac os


add
```json
{
  "msbuild-sdks": 
  {
    "Microsoft.NET.Sdk.IL": "6.0.0"
  }
}
```
to `global.json`
and
```xml
<Project Sdk="Microsoft.NET.Sdk.IL">
    <PropertyGroup>
        <OutputType>Exe</OutputType>
        <TargetFramework>net5.0</TargetFramework>
     </PropertyGroup>
</Project>
```
to `il-roundtrip.ilproj`. Now you can run it:
```
dotnet run --project il-roundtrip.ilproj 
```

TODO:
1. There will be `/usr/local/share/dotnet/sdk/6.0.200/Microsoft.Common.CurrentVersion.targets(4650,5): error : Expected file "obj/Debug/net6.0/refint/il-roundtrip.dll" does not exist` error if you set `TargetFramework` to `net6.0` and don't set `ProduceReferenceAssembly` to `false`