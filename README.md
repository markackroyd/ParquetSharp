# ParquetSharp 

ParquetSharp is a .NET library for reading and writing Apache [Parquet][1] files.

It is implemented in C# as a [PInvoke][2] wrapper around [apache-parquet-cpp][3] to provide high performance and compatibility.

[1]: https://parquet.apache.org
[2]: https://docs.microsoft.com/en-us/cpp/dotnet/how-to-call-native-dlls-from-managed-code-using-pinvoke
[3]: https://github.com/apache/arrow

|     | Status |
| ---: | --- |
| **Release Nuget** | [![NuGet latest release](https://img.shields.io/nuget/v/ParquetSharp.svg)](https://www.nuget.org/packages/ParquetSharp) |
| **Pre-Release Nuget** | [![NuGet latest pre-release](https://img.shields.io/nuget/vpre/ParquetSharp.svg)](https://www.nuget.org/packages/ParquetSharp) |
| **Windows Build** | [![Build status](https://ci.appveyor.com/api/projects/status/wac4f57a46ecg3gi?svg=true)](https://ci.appveyor.com/project/G-Research/parquetsharp) |
| **Linux Build** | [![Build status](https://ci.appveyor.com/api/projects/status/a551k0wrf5r6afj0?svg=true)](https://ci.appveyor.com/project/G-Research/parquetsharp-w98lm) |

## Examples

Both examples below output a Parquet file with three columns representing a timeseries of object-value pairs ordered by datetime and object id.

### Row-oriented API

The row-oriented API offers a convenient way to abstract the column-oriented nature of Parquet files at the expense of memory, speed and flexibility. It lets one write a whole row in a single call, often resulting in more readable code:

```csharp
var timestamps = new DateTime[] { /* ... */ };
var objectIds = new int[] { /* ... */ };
var values = timestamps.Select(t => objectIds.Select(o => (float) rand.NextDouble()).ToArray()).ToArray();
var columns = new[] {"Timestamp", "ObjectId", "Value"};

using (var rowWriter = ParquetFile.CreateRowWriter<(DateTime, int, float)>("float_timeseries.parquet", columns))
{
    for (int i = 0; i != timestamps.Length; ++i)
    {
        for (int j = 0; j != objectIds.Length; ++j)
        {
            rowWriter.WriteRow((timestamps[i], objectIds[j], values[i][j]));
        }
    }
}
```

### Low-level API

This closely maps to the API of apache-parquet-cpp. It also provides reader and writer abstractions (`LogicalColumnReader` and `LogicalColumnWriter` respectively) to convert between .NET types and Parquet representations.

```csharp
var timestamps = new DateTime[] { /* ... */ };
var objectIds = new int[] { /* ... */ };
var values = timestamps.Select(t => objectIds.Select(o => (float) rand.NextDouble()).ToArray()).ToArray();

var columns = new Column[]
{
    new Column<DateTime>("Timestamp"),
    new Column<int>("ObjectId"),
    new Column<float>("Value")
};

using (var file = new ParquetFileWriter("float_timeseries.parquet", columns))
using (var rowGroup = file.AppendRowGroup())
{
    using (var timestampWriter = rowGroup.NextColumn().LogicalWriter<DateTime>())
    {
        for (int i = 0; i != timestamps.Length; ++i)
        {
            timestampWriter.WriteBatch(Enumerable.Repeat(timestamps[i], objectIds.Length).ToArray());
        }
    }

    using (var objectIdWriter = rowGroup.NextColumn().LogicalWriter<int>())
    {
        for (int i = 0; i != timestamps.Length; ++i)
        {
            objectIdWriter.WriteBatch(objectIds);
        }
    }

    using (var valueWriter = rowGroup.NextColumn().LogicalWriter<float>())
    {
        for (int i = 0; i != timestamps.Length; ++i)
        {
            valueWriter.WriteBatch(values[i]);
        }
    }
}
```

## Rationale

We desired a Parquet implementation with the following properties:
- Cross platform (i.e. Windows and Linux).
- Callable from .NET Core.
- Good performance.
- Well maintained.
- Close to official Parquet reference implementations.

Not finding an existing solution meeting these requirements, we decided to implement a .NET wrapper around apache-parquet-cpp starting at version 1.4.0. The library tries to stick closely to the existing C++ API, although it does provide higher level APIs to facilitate its usage from .NET. The user should always be able to access the lower-level API.

## Known Limitations

Because this library is a thin wrapper around the C++ apache-parquet-cpp library, misuse can cause native memory access violations.

Typically this can arise when attempting to access an instance whose owner has been disposed. Because some objects and properties are exposed by apache-parquet-cpp via regular pointers (instead of consistently using `std::shared_ptr`), dereferencing these after the owner class instance has been destructed will lead to an invalid pointer access.

## Building

Building ParquetSharp for Windows requires the following dependencies:
- Visual Studio 2017 (15.9 or higher)
- Apache Arrow (0.13.0)

For building arrow (including parquet) and its dependencies, we recommend using Microsoft's [vcpkg](https://github.com/Microsoft/vcpkg). Note that the Windows build needs to be done in a Visual Studio x64 Native Tools Command Prompt for the build script to succeed.

**Windows (Visual Studio 2017 Win64 solution)**
```
> vcpkg_windows.bat
> build_windows.bat
```
**Linux**
```
> ./vcpkg_linux.sh
> ./build_linux.sh
> dotnet build csharp.test --configuration=Release
```

We have had to write our own `FindPackage` macros for most of the dependencies to get us going - it clearly needs more love and attention and is likely to be redundant with some vcpkg helper tools. The build step aboves will lead to CMake not finding the right debug library paths for several dependencies, you can manually fix these paths using CMake-GUI or equivalent (otherwise the build will fail in Debug).

## Contributing

We welcome new contributors! We will happily receive PRs for bug fixes or small changes. If you're contemplating something larger please get in touch first by opening a GitHub Issue describing the problem and how you propose to solve it.

## License

Copyright 2018 G-Research

Licensed under the Apache License, Version 2.0 (the "License"); you may not use these files except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
