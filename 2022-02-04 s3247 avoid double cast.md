Recently I've encountered Sonar's [rule](https://rules.sonarsource.com/csharp/RSPEC-3247), which suggests to replace
```csharp
if (x is Fruit)  // Noncompliant
{
  var f = (Fruit)x; // or x as Fruit
  // ...
}
```
with
```csharp
// C# 7
if (x is Fruit fruit)
{
  // ...
}
```
in order to avoid double casting.

I've checked [System.Private.CoreLib](https://github.com/dotnet/runtime/tree/c5c7967ddccc46c84c98c0e8c7e00c3009c65894/src/libraries/System.Private.CoreLib) and it's triggered on https://github.com/dotnet/runtime/blob/c5c7967ddccc46c84c98c0e8c7e00c3009c65894/src/libraries/System.Private.CoreLib/src/System/MemoryExtensions.cs#L2169

Some git blaming and I've found out that Stephen Toub is author of this code. Hm-m... This guy should understand such tricks. Therefore I should to dig deeper. Comment
```csharp
// constrained call avoiding boxing for value types
```
is very interesting.

Create benchmark project
```shell
dotnet new -i BenchmarkDotNet.Templates
mkdir benchmark && cd "$_"
dotnet new benchmark --console-app
```

add code below to Benchmark.cs
```csharp
using System;
using BenchmarkDotNet.Attributes;

namespace benchmark
{
    public struct ImplStruct : IFormattable
    {
        public string ToString(string format, IFormatProvider formatProvider)
        {
            return string.Empty;
        }
    }

    public class ImplClass : IFormattable
    {
        public string ToString(string format, IFormatProvider formatProvider)
        {
            return string.Empty;
        }
    }

    public static class SUT
    {
        public static string SingleCast<T>(T value)
        {
            string s = null;
            if (value is IFormattable formattable)
            {
                s = formattable.ToString(format: null, null);
            }

            return s;
        }

        public static string DoubleCast<T>(T value)
        {
            string s = null;
            if (value is IFormattable)
            {
                s = ((IFormattable)value).ToString(null, null);
            }

            return s;
        }
    }

    [MemoryDiagnoser]
    public class Benchmarks
    {
        private static readonly ImplClass ImplClass = new();
        private static readonly ImplStruct ImplStruct = new();

        [Benchmark]
        public void SingleCaseValueTypeNotImplement()
        {
            SUT.SingleCast(1);
        }

        [Benchmark]
        public void DoubleCastValueTypeNotImplement()
        {
            SUT.DoubleCast(1);
        }

        [Benchmark]
        public void SingleCaseReferenceTypeNotImplement()
        {
            SUT.SingleCast("test");
        }

        [Benchmark]
        public void DoubleCastReferenceTypeNotImplement()
        {
            SUT.DoubleCast("test");
        }

        [Benchmark]
        public void SingleCaseValueTypeImplement()
        {
            SUT.SingleCast(ImplStruct);
        }

        [Benchmark]
        public void DoubleCastValueTypeImplement()
        {
            SUT.DoubleCast(ImplStruct);
        }

        [Benchmark]
        public void SingleCaseReferenceTypeImplement()
        {
            SUT.SingleCast(ImplClass);
        }

        [Benchmark]
        public void DoubleCastReferenceTypeImplement()
        {
            SUT.DoubleCast(ImplClass);
        }

    }
}
```

then
```
dotnet run
```
and result

|                              Method |      Mean |     Error |    StdDev |  Gen 0 | Gen 1 | Gen 2 | Allocated |
|------------------------------------ |----------:|----------:|----------:|-------:|------:|------:|----------:|
|     SingleCaseValueTypeNotImplement | 12.893 ns | 0.1447 ns | 0.1283 ns | 0.0076 |     - |     - |      24 B |
|     DoubleCastValueTypeNotImplement |  4.358 ns | 0.0499 ns | 0.0467 ns |      - |     - |     - |         - |
| SingleCaseReferenceTypeNotImplement |  6.946 ns | 0.1211 ns | 0.0945 ns |      - |     - |     - |         - |
| DoubleCastReferenceTypeNotImplement |  7.099 ns | 0.0682 ns | 0.0605 ns |      - |     - |     - |         - |
|        SingleCaseValueTypeImplement |  9.281 ns | 0.0578 ns | 0.0541 ns | 0.0076 |     - |     - |      24 B |
|        DoubleCastValueTypeImplement |  1.640 ns | 0.0119 ns | 0.0099 ns |      - |     - |     - |         - |
|    SingleCaseReferenceTypeImplement |  5.885 ns | 0.0579 ns | 0.0541 ns |      - |     - |     - |         - |
|    DoubleCastReferenceTypeImplement |  5.881 ns | 0.0556 ns | 0.0520 ns |      - |     - |     - |         - |

Looks like single cast is not the best choice for reference type 

There is IDE0038
https://docs.microsoft.com/en-us/dotnet/fundamentals/code-analysis/style-rules/ide0020-ide0038

TODO:

- [ ] Investigate IL code

- [ ] Look at PVS Studio and other static analyzers
