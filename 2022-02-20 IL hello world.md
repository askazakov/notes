```bash
mkdir il-hello && cd $_
dotnet new sln
dotnet new globaljson
```
Add
```json
{
    "msbuild-sdks": 
    {
      "Microsoft.NET.Sdk.IL": "6.0.0"
    }
}
```
to global.json.

```
mkdir SampleIL
touch SampleIL/SampleIL.ilproj
touch SampleIL/hello.il
```
modify `SampleIL.ilproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk.IL">
    <PropertyGroup>
        <TargetFramework>netstandard2.0</TargetFramework>
     </PropertyGroup>
</Project>
```
add
```
.assembly extern mscorlib
{
    .ver 4:0:0:0
    .publickeytoken = (B7 7A 5C 56 19 34 E0 89)
}

.assembly SampleIL
{
  .ver 1:0:0:0
}
 
.module SampleIL.dll

.class public auto ansi beforefieldinit Hello
  extends System.Object
{
  .method public hidebysig static string World() cil managed
  {
    ldstr      "Hello World!"
    ret
  }
}
```
to `hello.il`.

Now you have library that you could use for example from ordinary console project.
```
dotnet sln il-hello.sln add SampleIL/SampleIL.ilproj
dotnet new console --output ConsoleApp --framework net6.0
dotnet sln il-hello.sln add ConsoleApp/ConsoleApp.csproj
dotnet add ConsoleApp/ConsoleApp.csproj reference SampleIL/SampleIL.ilproj
```
replace `Program.cs` with
```csharp
Console.WriteLine(Hello.World());
```
Build and run
```
dotnet build
dotnet run --project ConsoleApp/ConsoleApp.csproj 
```

TODO:
1. there is 
```
hello.il(10): warning : Reference to undeclared extern assembly 'mscorlib'. Attempting autodetect
```
warning if remove `extern mscorlib` block form hello.il 