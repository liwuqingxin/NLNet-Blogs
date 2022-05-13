[toc]

> 参考文档：[MSBuild - MSBuild | Microsoft Docs](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild?view=vs-2022)，C++：[MSBuild on the command line - C++ | Microsoft Docs](https://docs.microsoft.com/en-us/cpp/build/msbuild-visual-cpp?view=msvc-170)
>
> 编写人员：聂亮兵
>
> 日期：2022年4月12日
>
> 文件版本：v1.0.0
>
> MSBuild 版本：MSBuild 17.0

> 待处理项目：
>
> 1. 整理Microsoft.Common.targets、Microsoft.Common.props、Directory.Build.props、Directory.Build.targets；
> 2. 自定义日志生成器ILogger：[生成记录器 - MSBuild | Microsoft Docs](https://docs.microsoft.com/zh-cn/visualstudio/msbuild/build-loggers?view=vs-2022)，支持发送邮件
> 3. 多处理器下的日志记录：
>    1. [多处理器环境下的日志记录 - MSBuild | Microsoft Docs](https://docs.microsoft.com/zh-cn/visualstudio/msbuild/logging-in-a-multi-processor-environment?view=vs-2022)，
>    2. [编写多处理器可识别的记录器 - MSBuild | Microsoft Docs](https://docs.microsoft.com/zh-cn/visualstudio/msbuild/writing-multi-processor-aware-loggers?view=vs-2022)
>    3. [创建转发记录器 - MSBuild | Microsoft Docs](https://docs.microsoft.com/zh-cn/visualstudio/msbuild/creating-forwarding-loggers?view=vs-2022)

# MSBuild 介绍

## 一个Project文件

```xml
<Project DefaultTargets="Compile"
    xmlns="http://schemas.microsoft.com/developer/msbuild/2003" >

    <PropertyGroup>
        <builtdir>built</builtdir>
    </PropertyGroup>

    <ItemGroup>
        <CSFile Include="*.cs" Exclude="Form2.cs"/>

        <Reference Include="System.dll"/>
        <Reference Include="System.Data.dll"/>
        <Reference Include="System.Drawing.dll"/>
        <Reference Include="System.Windows.Forms.dll"/>
        <Reference Include="System.XML.dll"/>
    </ItemGroup>

    <Target Name="PreBuild">
        <Exec Command="if not exist $(builtdir) md $(builtdir)"/>
    </Target>

    <Target Name="Compile" DependsOnTargets="PreBuild">
        <Csc Sources="@(CSFile)"
            References="@(Reference)"
            OutputAssembly="$(builtdir)\$(MSBuildProjectName).exe"
            TargetType="exe" />
    </Target>
</Project>
```

## Complie Project

```bash
msbuild helloword.csproj -p:Configuration=Debug -fl -flp:logFile=helloword.log;v=diag
```

# 编译流程

```mermaid
graph LR
    1(statrup) --> 2(evaluate) --> 3(execution)
```

## initial startup

1. 从Microsoft.Build.dll启动
2. 从脚本或者命令行中启动可执行文件
3. 从VS中调用

> MSBuild和VS编译解决方案的不同
>
> 1. VS控制解决方案的生成流程，仅调用MSBuild生成Project；MSBuild自身控制解决方案生成，即将sln文件作为xml输入；
> 2. 对于依赖项，VS会通过MSBuild返回预期输出，而MSBuild会直接执行生成产生输出；

## evaluation

评估阶段处理输入文件，将静态导入整个xml和导入链，将输入转为内存对象模型。分为六个阶段：

- Evaluate environment variables
- Evaluate imports and properties
- Evaluate item definitions
- Evaluate items
- Evaluate [UsingTask](https://docs.microsoft.com/en-us/visualstudio/msbuild/usingtask-element-msbuild?view=vs-2022) elements
- Evaluate targets

> 以上过程是按序产生的，例如，Properties中不可访问后面的Item的部分，也不可访问后面出现的Properties。

## execution

- 对Target内的Properties和Items求值，首先是所有Properties，然后是所有Items；
- 生成项目；

> 1. 生成顺序依赖于Targets上的BeforeTargets、DependsOnTargets、AfterTargets属性。这个过程是由栈实现的。
>
> 2. ProjectReference项指定的依赖项，将通过标准文件中定义的标准Target`ResolveProjectReferences`来执行。
> 3. 解决方案文件sln可以创建项目依赖，对解决方案的编译，MSBuild会将其转换为ProjectReference，以相同的方式处理。
> 4. 如果使用`-graph`选项，ProjectReference会成为MSBuild的一级概念，MSBuild会分析所有项目，产生生成顺序图，然后遍历该关系图确定生成顺序。

```sequence
Proj1->MSBuild:编译
MSBuild->ResolveProjectReferences:处理依赖项
ResolveProjectReferences-->>MSBuild:编译依赖项
MSBuild-->>Proj1:继续执行编译
```

## 并行执行

并行执行`-m`/`-maxCpuCount`：[使用 MSBuild 并行生成多个项目](https://docs.microsoft.com/zh-cn/visualstudio/msbuild/building-multiple-projects-in-parallel-with-msbuild?view=vs-2022)。

## 标准导入

`Microsoft.Common.props`和`Microsoft.Common.targets`都是由 .NET 项目文件导入的（在 SDK 样式项目中显式或隐式导入），并且位于 Visual Studio 安装的 MSBuild\Current\bin 文件夹中。

`Microsoft.Common.targets`包装了`Microsoft.Common.CurrentVersion.targets`，其内部定义如下：

```xml
  <PropertyGroup>
    <BuildDependsOn>
      BeforeBuild;
      CoreBuild;
      AfterBuild
    </BuildDependsOn>
  </PropertyGroup>

  <Target
      Name="Build"
      Condition=" '$(_InvalidConfigurationWarning)' != 'true' "
      DependsOnTargets="$(BuildDependsOn)"
      Returns="@(TargetPathWithTargetPlatformMoniker)" />
```

其中CoreBuild定义如下：

```xml
  <PropertyGroup>
    <CoreBuildDependsOn>
      BuildOnlySettings;
      PrepareForBuild;
      PreBuildEvent;
      ResolveReferences;
      PrepareResources;
      ResolveKeySource;
      Compile;
      ExportWindowsMDFile;
      UnmanagedUnregistration;
      GenerateSerializationAssemblies;
      CreateSatelliteAssemblies;
      GenerateManifests;
      GetTargetPath;
      PrepareForRun;
      UnmanagedRegistration;
      IncrementalClean;
      PostBuildEvent
    </CoreBuildDependsOn>
  </PropertyGroup>
  <Target
      Name="CoreBuild"
      DependsOnTargets="$(CoreBuildDependsOn)">

    <OnError ExecuteTargets="_TimeStampAfterCompile;PostBuildEvent" Condition="'$(RunPostBuildEvent)'=='Always' or '$(RunPostBuildEvent)'=='OnOutputUpdated'"/>
    <OnError ExecuteTargets="_CleanRecordFileWrites"/>

  </Target>
```

上述依赖的Targets说明：

| 目标                            | 描述                                                         |
| :------------------------------ | :----------------------------------------------------------- |
| BuildOnlySettings               | 仅适用于实际生成的设置，而不适用于 Visual Studio 在项目加载时调用 MSBuild。 |
| PrepareForBuild                 | 准备生成的先决条件                                           |
| PreBuildEvent                   | 项目的扩展点，用于定义在生成之前要执行的任务                 |
| ResolveProjectReferences        | 分析项目依赖项并生成引用的项目                               |
| ResolveAssemblyReferences       | 找到引用的程序集。                                           |
| ResolveReferences               | 包含 `ResolveProjectReferences` 和 `ResolveAssemblyReferences`，用于查找所有依赖项 |
| PrepareResources                | 处理资源文件                                                 |
| ResolveKeySource                | 解析用于对程序集进行签名的强名称密钥以及用于对 [ClickOnce](https://docs.microsoft.com/zh-cn/visualstudio/deployment/clickonce-security-and-deployment?view=vs-2022) 清单进行签名的证书。 |
| Compile                         | 调用编译器                                                   |
| ExportWindowsMDFile             | 基于编译器生成的 WinMDModule 文件生成 [WinMD](https://docs.microsoft.com/zh-cn/uwp/winrt-cref/winmd-files) 文件。 |
| UnmanagedUnregistration         | 从上一个生成中删除/清除 [COM 互操作](https://docs.microsoft.com/zh-cn/dotnet/standard/native-interop/cominterop)注册表项 |
| GenerateSerializationAssemblies | 使用 [sgen.exe](https://docs.microsoft.com/zh-cn/dotnet/standard/serialization/xml-serializer-generator-tool-sgen-exe) 生成 XML 序列化程序集。 |
| CreateSatelliteAssemblies       | 为资源中的每个唯一区域性创建一个附属程序集。                 |
| 生成清单                        | 生成 [ClickOnce](https://docs.microsoft.com/zh-cn/visualstudio/deployment/clickonce-security-and-deployment?view=vs-2022) 应用程序和部署清单或本机清单。 |
| GetTargetPath                   | 使用元数据返回包含此项目的生成产品（可执行文件或程序集）的项。 |
| PrepareForRun                   | 将生成输出复制到最终目录（如果它们已更改）。                 |
| UnmanagedRegistration           | 设置 [COM 互操作](https://docs.microsoft.com/zh-cn/dotnet/standard/native-interop/cominterop)的注册表项 |
| IncrementalClean                | 删除在先前生成中生成但未在当前生成中生成的文件。 这是使 `Clean` 在增量生成中工作的必须条件。 |
| PostBuildEvent                  | 项目的扩展点，用于定义在生成之后运行的任务                   |

> C#的Target文件：Microsoft.CSharp.targets。

## 用户可配置导入

- Directory.Build.props
- Directory.Build.targets

这些文件由其下任何子文件夹中的任何项目的标准导入读入，这通常是解决方案级别的设置，用于控制解决方案中的所有项目，但是在文件系统中也可能更高，直至驱动器的根目录。

Directory.Build.props 文件由 Microsoft.Common.props 导入，因此项目文件中提供了其中定义的属性。 可以在项目文件中重新定义这些属性，以便基于每个项目自定义值。 将在项目文件后读入 Directory.Build.targets 文件。 它通常包含目标，但在这里，你还可以定义不希望个别项目重新定义的属性。

> 注意，`Directory.Build.props`依赖于标准导入`Microsoft.Common.props`

# Properties

> 预留属性：[MSBuild 保留和常见属性](https://docs.microsoft.com/zh-cn/visualstudio/msbuild/msbuild-reserved-and-well-known-properties?view=vs-2022)

## 注册表属性

```
// 获取系统注册表值
$(registry:Hive\MyKey\MySubKey@Value)

// 获取默认的子健值
$(registry:Hive\MyKey\MySubKey)
```

## 环境变量

```xml
// 引用环境变量
<FinalOutput>$(BIN_PATH)\MyAssembly.dll</FinalOutput>

// 提供默认值
<ToolsPath Condition="'$(TOOLSPATH)' == ''">c:\tools</ToolsPath>

// 以下项目文件使用环境变量来指定目录的位置
<Project DefaultTargets="FakeBuild" DefaultTargets="FakeBuild">
    <PropertyGroup>
        <FinalOutput>$(BIN_PATH)\myassembly.dll</FinalOutput>
        <ToolsPath Condition=" '$(ToolsPath)' == '' ">
            C:\Tools
        </ToolsPath>
    </PropertyGroup>
    <Target Name="FakeBuild">
        <Message Text="Building $(FinalOutput) using the tools at $(ToolsPath)..."/>
    </Target>
</Project>
```

> 注意，属性名称不区分大小写

## 引用项目名称或者位置

```xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003"
    DefaultTargets = "Compile">

    <!-- Specify the inputs -->
    <ItemGroup>
        <CSFile Include = "consolehwcs1.cs"/>
     </ItemGroup>
    <Target Name = "Compile">
        <!-- Run the Visual C# compilation using
        input files of type CSFile -->
        <CSC Sources = "@(CSFile)"
            OutputAssembly = "$(MSBuildProjectName).exe" >
            <!-- Set the OutputAssembly attribute of the CSC task
            to the name of the project -->
            <Output
                TaskParameter = "OutputAssembly"
                ItemName = "EXEFile" />
        </CSC>
        <!-- Log the file name of the output file -->
        <Message Text="The output file is @(EXEFile)"/>
    </Target>
</Project>
```

## 编译选项

```xml
<Project DefaultTargets = "Compile"
    xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

    <!-- Sets the default flavor if an environment variable called
    Flavor is not set or specified on the command line -->
    <PropertyGroup>
        <Flavor Condition="'$(Flavor)'==''">DEBUG</Flavor>
    </PropertyGroup>

    <!-- Define the DEBUG settings -->
    <PropertyGroup Condition="'$(Flavor)'=='DEBUG'">
        <DebugType>full</DebugType>
        <Optimize>no</Optimize>
    </PropertyGroup>

    <!-- Define the RETAIL settings -->
    <PropertyGroup Condition="'$(Flavor)'=='RETAIL'">
        <DebugType>pdbonly</DebugType>
        <Optimize>yes</Optimize>
    </PropertyGroup>

    <!-- Set the application name as a property -->
    <PropertyGroup>
        <appname>HelloWorldCS</appname>
    </PropertyGroup>

    <!-- Specify the inputs by type and file name -->
    <ItemGroup>
        <CSFile Include = "consolehwcs1.cs"/>
    </ItemGroup>

    <Target Name = "Compile">
        <!-- Run the Visual C# compilation using input files
        of type CSFile -->
        <CSC  Sources = "@(CSFile)"
            DebugType="$(DebugType)"
            Optimize="$(Optimize)"
            OutputAssembly="$(appname).exe" >

            <!-- Set the OutputAssembly attribute of the CSC
            task to the name of the executable file that is
            created -->
            <Output TaskParameter="OutputAssembly"
                ItemName = "EXEFile" />
        </CSC>
        <!-- Log the file name of the output file -->
        <Message Text="The output file is @(EXEFile)"/>
    </Target>
</Project>
```

```bash
msbuild consolehwcs1.proj -p:flavor=retail
msbuild consolehwcs1.proj -p:flavor=debug
```

## 优先使用本地属性

```xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003"
ToolsVersion="4.0" TreatAsLocalProperty="Color">

    <PropertyGroup>
        <Color>Blue</Color>
    </PropertyGroup>

    <Target Name="go">
        <Message Text="Color: $(Color)" />
    </Target>
</Project>

<!--
  Output with TreatAsLocalProperty="Color" in project tag:
     Color: Blue

  Output without TreatAsLocalProperty="Color" in project tag:
     Color: Green
-->
```

## 属性函数

从属性函数返回的字符串值已转义[特殊字符](https://docs.microsoft.com/zh-cn/visualstudio/msbuild/msbuild-special-characters?view=vs-2022)。 如果要将这些值视为如同直接置于项目文件中，请使用 `$([MSBuild]::Unescape())` 取消转义特殊字符。

```xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
    <!-- Build the path to a file in the root of the project -->
    <PropertyGroup>
        <NewFilePath>
            $([System.IO.Path]::Combine($(MSBuildProjectDirectory), `BuildInfo.txt`))
        </NewFilePath>
        
        <Today>$([System.DateTime]::Now.ToString("yyyy.MM.dd"))</Today>
    </PropertyGroup>
</Project>
```

### 字符串属性函数

```
$(ProjectOutputFolder.Substring(0,3))
```

### 静态属性函数

```xml
$([Class]::Property)
<Today>$([System.DateTime]::Now)</Today>

$([Class]::Method(Parameters))
<NewGuid>$([System.Guid]::NewGuid())</NewGuid>

$([Class]::Property.Method(Parameters))
<Today>$([System.DateTime]::Now.ToString('yyyy.MM.dd'))</Today>
```

在静态属性函数中，可以使用以下系统类的任何静态方法或属性：

- [System.Byte](https://docs.microsoft.com/zh-cn/dotnet/api/system.byte)
- [System.Char](https://docs.microsoft.com/zh-cn/dotnet/api/system.char)
- [System.Convert](https://docs.microsoft.com/zh-cn/dotnet/api/system.convert)
- [System.DateTime](https://docs.microsoft.com/zh-cn/dotnet/api/system.datetime)
- [System.Decimal](https://docs.microsoft.com/zh-cn/dotnet/api/system.decimal)
- [System.Double](https://docs.microsoft.com/zh-cn/dotnet/api/system.double)
- [System.Enum](https://docs.microsoft.com/zh-cn/dotnet/api/system.enum)
- [System.Guid](https://docs.microsoft.com/zh-cn/dotnet/api/system.guid)
- [System.Int16](https://docs.microsoft.com/zh-cn/dotnet/api/system.int16)
- [System.Int32](https://docs.microsoft.com/zh-cn/dotnet/api/system.int32)
- [System.Int64](https://docs.microsoft.com/zh-cn/dotnet/api/system.int64)
- [System.IO.Path](https://docs.microsoft.com/zh-cn/dotnet/api/system.io.path)
- [System.Math](https://docs.microsoft.com/zh-cn/dotnet/api/system.math)
- [System.Runtime.InteropServices.OSPlatform](https://docs.microsoft.com/zh-cn/dotnet/api/system.runtime.interopservices.osplatform)
- [System.Runtime.InteropServices.RuntimeInformation](https://docs.microsoft.com/zh-cn/dotnet/api/system.runtime.interopservices.runtimeinformation)
- [System.UInt16](https://docs.microsoft.com/zh-cn/dotnet/api/system.uint16)
- [System.UInt32](https://docs.microsoft.com/zh-cn/dotnet/api/system.uint32)
- [System.UInt64](https://docs.microsoft.com/zh-cn/dotnet/api/system.uint64)
- [System.SByte](https://docs.microsoft.com/zh-cn/dotnet/api/system.sbyte)
- [System.Single](https://docs.microsoft.com/zh-cn/dotnet/api/system.single)
- [System.String](https://docs.microsoft.com/zh-cn/dotnet/api/system.string)
- [System.StringComparer](https://docs.microsoft.com/zh-cn/dotnet/api/system.stringcomparer)
- [System.TimeSpan](https://docs.microsoft.com/zh-cn/dotnet/api/system.timespan)
- [System.Text.RegularExpressions.Regex](https://docs.microsoft.com/zh-cn/dotnet/api/system.text.regularexpressions.regex)
- [System.UriBuilder](https://docs.microsoft.com/zh-cn/dotnet/api/system.uribuilder)
- [System.Version](https://docs.microsoft.com/zh-cn/dotnet/api/system.version)
- [Microsoft.Build.Utilities.ToolLocationHelper](https://docs.microsoft.com/zh-cn/dotnet/api/microsoft.build.utilities.toollocationhelper)

此外，还可以使用以下静态方法和属性：

- [System.Environment::CommandLine](https://docs.microsoft.com/zh-cn/dotnet/api/system.environment.commandline)
- [System.Environment::ExpandEnvironmentVariables](https://docs.microsoft.com/zh-cn/dotnet/api/system.environment.expandenvironmentvariables)
- [System.Environment::GetEnvironmentVariable](https://docs.microsoft.com/zh-cn/dotnet/api/system.environment.getenvironmentvariable)
- [System.Environment::GetEnvironmentVariables](https://docs.microsoft.com/zh-cn/dotnet/api/system.environment.getenvironmentvariables)
- [System.Environment::GetFolderPath](https://docs.microsoft.com/zh-cn/dotnet/api/system.environment.getfolderpath)
- [System.Environment::GetLogicalDrives](https://docs.microsoft.com/zh-cn/dotnet/api/system.environment.getlogicaldrives)
- [System.IO.Directory::GetDirectories](https://docs.microsoft.com/zh-cn/dotnet/api/system.io.directory.getdirectories)
- [System.IO.Directory::GetFiles](https://docs.microsoft.com/zh-cn/dotnet/api/system.io.directory.getfiles)
- [System.IO.Directory::GetLastAccessTime](https://docs.microsoft.com/zh-cn/dotnet/api/system.io.directory.getlastaccesstime)
- [System.IO.Directory::GetLastWriteTime](https://docs.microsoft.com/zh-cn/dotnet/api/system.io.directory.getlastwritetime)
- [System.IO.Directory::GetParent](https://docs.microsoft.com/zh-cn/dotnet/api/system.io.directory.getparent)
- [System.IO.File::Exists](https://docs.microsoft.com/zh-cn/dotnet/api/system.io.file.exists)
- [System.IO.File::GetCreationTime](https://docs.microsoft.com/zh-cn/dotnet/api/system.io.file.getcreationtime)
- [System.IO.File::GetAttributes](https://docs.microsoft.com/zh-cn/dotnet/api/system.io.file.getattributes)
- [System.IO.File::GetLastAccessTime](https://docs.microsoft.com/zh-cn/dotnet/api/system.io.file.getlastaccesstime)
- [System.IO.File::GetLastWriteTime](https://docs.microsoft.com/zh-cn/dotnet/api/system.io.file.getlastwritetime)
- [System.IO.File::ReadAllText](https://docs.microsoft.com/zh-cn/dotnet/api/system.io.file.readalltext)

### MSBuild属性函数

```
$([MSBuild]::Method(Parameters))

// 加法计算
$([MSBuild]::Add($(NumberOne), $(NumberTwo)))
```

下面列出了 MSBuild 属性函数：

| 函数签名                                                     | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| double Add(双精度型值 a, 双精度型值 b)                       | 将两个双精度型值相加。                                       |
| long Add(长型值 a, 长型值 b)                                 | 将两个长型值相加。                                           |
| double Subtract(双精度型值 a, 双精度型值 b)                  | 将两个双精度型值相减。                                       |
| long Subtract(长型值 a, 长型值 b)                            | 将两个长型值相减。                                           |
| double Multiply(双精度型值 a, 双精度型值 b)                  | 将两个双精度型值相乘。                                       |
| long Multiply(长型值 a, 长型值 b)                            | 将两个长型值相乘。                                           |
| double Divide(双精度型值 a, 双精度型值 b)                    | 将两个双精度型值相除。                                       |
| long Divide(长型值 a, 长型值 b)                              | 将两个长型值相除。                                           |
| double Modulo(双精度型值 a, 双精度型值 b)                    | 对两个双精度型值取模。                                       |
| long Modulo(长型值 a, 长型值 b)                              | 对两个长型值取模。                                           |
| string Escape(未转义字符串)                                  | 根据 MSBuild 转义规则对字符串进行转义。                      |
| string Unescape(已转义字符串)                                | 根据 MSBuild 转义规则取消对字符串进行转义。                  |
| int BitwiseOr(第一个整型值, 第二个整型值)                    | 对第一个值和第二个值执行按位 `OR`（第一个值 \| 第二个值）。  |
| int BitwiseAnd(第一个整型值, 第二个整型值)                   | 对第一个值和第二个值执行按位 `AND`（第一个值 & 第二个值）。  |
| int BitwiseXor(第一个整型值, 第二个整型值)                   | 对第一个值和第二个值执行按位 `XOR`（第一个值 ^ 第二个值）。  |
| int BitwiseNot(第一个整型值)                                 | 执行按位 `NOT`（~第一个值）。                                |
| bool IsOsPlatform(string platformString)                     | 指定当前 OS 平台是否为 `platformString`。 `platformString` 必须属于 [OSPlatform](https://docs.microsoft.com/zh-cn/dotnet/api/system.runtime.interopservices.osplatform)。 |
| bool IsOSUnixLike()                                          | 如果当前 OS 是 Unix 系统，则为 True。                        |
| string NormalizePath(params string[] path)                   | 获取指定路径的规范化完整路径，并确保其包含当前操作系统的正确目录分隔符。 |
| string NormalizeDirectory(params string[] path)              | 获取指定目录的规范化完整路径，确保其包含当前操作系统的正确目录分隔符，并确保其具有尾部反斜杠。 |
| string EnsureTrailingSlash(string path)                      | 如果给定路径没有尾部反斜杠，请添加一个。 如果路径为空字符串，请勿其进行修改。 |
| string GetPathOfFileAbove(string file, string startingDirectory) | 在当前生成文件位置上级的目录结构中搜索并返回文件的完整路径，如果指定，则基于 `startingDirectory`。 |
| GetDirectoryNameOfFileAbove(string startingDirectory, string fileName) | 在指定目录或该目录上级的目录结构位置中查找并返回文件目录。   |
| string MakeRelative(string basePath, string path)            | 将 `path` 关联到 `basePath`。 `basePath` 必须是绝对目录。 如果无法关联 `path`，则会返回逐字字符串。 类似于 `Uri.MakeRelativeUri`。 |
| string ValueOrDefault(string conditionValue, string defaultValue) | 仅当“conditionValue”为空时在参数“defaultValue”中返回字符串，否则返回值 conditionValue。 |

### 嵌套的属性函数

```
$([MSBuild]::BitwiseAnd(32, $([System.IO.File]::GetAttributes(tempFile))))
```

此示例返回路径 [FileAttributes](https://docs.microsoft.com/zh-cn/dotnet/api/system.io.fileattributes) 提供的 文件的 . 位（32 或 0）值。 请注意，枚举的数据值在某些上下文中不能按名称显示。 在上一示例中，必须使用数值 (32)。 在其他情况下，根据所调用方法的预期结果，必须使用枚举数据值。 在下面的示例中，必须使用枚举值 [RegexOptions](https://docs.microsoft.com/zh-cn/dotnet/api/system.text.regularexpressions.regexoptions).`ECMAScript`， 因为无法按此方法预期那样转换数值。

```xml
<PropertyGroup>
    <GitVersionHeightWithOffset>$([System.Text.RegularExpressions.Regex]::Replace("$(PrereleaseVersion)", "^.*?(\d+)$", "$1", "System.Text.RegularExpressions.RegexOptions.ECMAScript"))</GitVersionHeightWithOffset>
</PropertyGroup>
```

### MSBuild EnsureTrailingSlash

```
// 确保尾部有反斜杠
$([MSBuild]::EnsureTrailingSlash('$(PathProperty)'))
```

### MSBuild GetDirectoryNameOfFileAbove

```xml
// 从指定目录开始向上查找包含指定文件的目录，返回完整路径
$([MSBuild]::GetDirectoryNameOfFileAbove(string startingDirectory, string fileName))

// 在当前文件夹中或更高一级中导入最近的 EnlistmentInfo.props 文件（仅当找到匹配项时）：
<Import Project="$([MSBuild]::GetDirectoryNameOfFileAbove($(MSBuildThisFileDirectory), EnlistmentInfo.props))\EnlistmentInfo.props" Condition=" '$([MSBuild]::GetDirectoryNameOfFileAbove($(MSBuildThisFileDirectory), EnlistmentInfo.props))' != '' " />

// 请注意，此示例可以通过使用 GetPathOfFileAbove 函数来更简洁地编写：
<Import Project="$([MSBuild]::GetPathOfFileAbove(EnlistmentInfo.props))" Condition=" '$([MSBuild]::GetPathOfFileAbove(EnlistmentInfo.props))' != '' " />
```

### MSBuild GetPathOfFileAbove

```xml
// 从指定目录开始向上查找包含指定文件的完整路径，默认情况下，搜索将当前文件自身的目录开始
$([MSBuild]::GetPathOfFileAbove(string file, [string startingDirectory]))

此示例演示如何在当前目录中或更高一级中导入名为“dir.props”的文件（仅当找到匹配项时）：
<Import Project="$([MSBuild]::GetPathOfFileAbove(dir.props))" Condition=" '$([MSBuild]::GetPathOfFileAbove(dir.props))' != '' " />

// 从父目录开始，避免导入文件自身
<Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../'))" />
```

### MSBUild MSBuild GetRegistryValue

```
// default value
$([MSBuild]::GetRegistryValue(`HKEY_CURRENT_USER\Software\Microsoft\VisualStudio\10.0\Debugger`, ``))

$([MSBuild]::GetRegistryValue(`HKEY_CURRENT_USER\Software\Microsoft\VisualStudio\10.0\Debugger`, `SymbolCacheDir`))

// parens in name and value
$([MSBuild]::GetRegistryValue(`HKEY_LOCAL_MACHINE\SOFTWARE\(SampleName)`, `(SampleValue)`))
```

### MSBuild GetRegistryValueFromView

MSBuild `GetRegistryValueFromView` 属性函数在给定了注册表项、值以及一个或多个经过排序的注册表视图的情况下，获取系统注册表数据。 该属性函数将按顺序在每个注册表视图中搜索注册表项和值，直至找到它们。

该属性函数的语法是：

```
[MSBuild]::GetRegistryValueFromView(string keyName, string valueName, object defaultValue, params object[] views)
```

Windows 64 位操作系统维护一个 HKEY_LOCAL_MACHINE\SOFTWARE\Wow6432Node 注册表项，它表示 32 位应用程序的 HKEY_LOCAL_MACHINE\SOFTWARE 注册表视图。

默认情况下，在 WOW64 上运行的 32 位应用程序将访问 32 位注册表视图，而 64 位应用程序将访问 64 位注册表视图。

以下这些注册表视图是可用的：

| 注册表视图              | 定义                                           |
| :---------------------- | :--------------------------------------------- |
| RegistryView.Registry32 | 32 位应用程序注册表视图。                      |
| RegistryView.Registry64 | 64 位应用程序注册表视图。                      |
| RegistryView.Default    | 与应用程序正在其中运行的进程匹配的注册表视图。 |

下面是一个示例。

```
$([MSBuild]::GetRegistryValueFromView('HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Microsoft SDKs\Silverlight\v3.0\ReferenceAssemblies', 'SLRuntimeInstallPath', null, RegistryView.Registry64, RegistryView.Registry32))
```

首先在 64 位注册表视图中查找，然后在 32 位注册表视图中查找，以获取 ReferenceAssemblies 项的 SLRuntimeInstallPath 数据。

### MSBuild MakeRelative

```xml
// MSBuild MakeRelative 属性函数将返回第二条路径的相对路径（相对于第一条路径）。 每条路径可以是文件或文件夹。
$([MSBuild]::MakeRelative($(FileOrFolderPath1), $(FileOrFolderPath2)))

<PropertyGroup>
    <Path1>c:\users\</Path1>
    <Path2>c:\users\username\</Path2>
</PropertyGroup>

<Target Name = "Go">
    <Message Text ="$([MSBuild]::MakeRelative($(Path1), $(Path2)))" />
    <Message Text ="$([MSBuild]::MakeRelative($(Path2), $(Path1)))" />
</Target>

<!--
Output:
   username\
   ..\
-->
```

### MSBuild ValueOrDefault

```xml
// MSBuild ValueOrDefault 属性函数将返回第一个参数，除非它为 null 或空，此时该函数将返回第二个参数。

<Project ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

    <PropertyGroup>
        <Value1>$([MSBuild]::ValueOrDefault('$(UndefinedValue)', 'a'))</Value1>
        <Value2>$([MSBuild]::ValueOrDefault('b', '$(Value1)'))</Value2>
    </PropertyGroup>

    <Target Name="MyTarget">
        <Message Text="Value1 = $(Value1)" />
        <Message Text="Value2 = $(Value2)" />
    </Target>
</Project>

<!--
Output:
  Value1 = a
  Value2 = b
-->
```

### MSBuild TargetFramework 和 TargetPlatform 函数

MSBuild 16.7 及更高版本定义了几个用于处理 [TargetFramework 和 TargetPlatform 属性](https://docs.microsoft.com/zh-cn/visualstudio/msbuild/msbuild-target-framework-and-target-platform?view=vs-2022)的函数。

| 函数签名                                                     | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| GetTargetFrameworkIdentifier(string targetFramework)         | 从 TargetFramework 分析 TargetFrameworkIdentifier。          |
| GetTargetFrameworkVersion(string targetFramework, int versionPartCount) | 从 TargetFramework 分析 TargetFrameworkVersion。             |
| GetTargetPlatformIdentifier(string targetFramework)          | 从 TargetFramework 分析 TargetPlatformIdentifier。           |
| GetTargetPlatformVersion(string targetFramework, int versionPartCount) | 从 TargetFramework 分析 TargetPlatformVersion。              |
| IsTargetFrameworkCompatible(string targetFrameworkTarget, string targetFrameworkCandidate) | 如果候选目标框架与此目标框架兼容，则返回“True”，否则返回 false。 |

```xml
<Project ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

    <PropertyGroup>
        <Value1>$([MSBuild]::GetTargetFrameworkIdentifier('net5.0-windows7.0'))</Value1>
        <Value2>$([MSBuild]::GetTargetFrameworkVersion('net5.0-windows7.0'))</Value2>
        <Value3>$([MSBuild]::GetTargetPlatformIdentifier('net5.0-windows7.0'))</Value3>
        <Value4>$([MSBuild]::GetTargetPlatformVersion('net5.0-windows7.0'))</Value4>
        <Value5>$([MSBuild]::IsTargetFrameworkCompatible('net5.0-windows', 'net5.0'))</Value5>
    </PropertyGroup>

    <Target Name="MyTarget">
        <Message Text="Value1 = $(Value1)" />
        <Message Text="Value2 = $(Value2)" />
        <Message Text="Value3 = $(Value3)" />
        <Message Text="Value4 = $(Value4)" />
        <Message Text="Value5 = $(Value5)" />
    </Target>
</Project>

输出
Value1 = .NETCoreApp
Value2 = 5.0
Value3 = windows
Value4 = 7.0
Value5 = True
```

### MSBuild版本比较函数

MSBuild 16.5 及更高版本定义了若干函数，用于比较表示版本的字符串。

> 条件中的比较运算符[可以比较可分析为 `System.Version` 对象](https://docs.microsoft.com/zh-cn/visualstudio/msbuild/msbuild-conditions?view=vs-2022#comparing-versions) 的字符串，但比较可能会产生意外结果。 首选属性函数。

| 函数签名                                       | 描述                                                     |
| :--------------------------------------------- | :------------------------------------------------------- |
| VersionEquals(string a, string b)              | 如果版本 `a` 和 `b` 等效，则根据以下规则返回 `true`。    |
| VersionGreaterThan(string a, string b)         | 如果版本 `a` 大于 `b`，则根据以下规则返回 `true`。       |
| VersionGreaterThanOrEquals(string a, string b) | 如果版本 `a` 大于或等于 `b`，则根据以下规则返回 `true`。 |
| VersionLessThan(string a, string b)            | 如果版本 `a` 小于 `b`，则根据以下规则返回 `true`。       |
| VersionLessThanOrEquals(string a, string b)    | 如果版本 `a` 小于或等于 `b`，则根据以下规则返回 `true`。 |
| VersionNotEquals(string a, string b)           | 如果版本 `a` 和 `b` 等效，则根据以下规则返回 `false`。   |

在这些方法中，版本的分析类似于 [System.Version](https://docs.microsoft.com/zh-cn/dotnet/api/system.version)，但以下情况例外：

- 忽略前导 `v` 或 `V`，以便可以与 `$(TargetFrameworkVersion)` 进行比较。
- 从第一个“-”或“+”到版本字符串末尾的所有内容都将被忽略。 虽然顺序与 SemVer 不同，但允许传递语义版本 (SemVer)。 相反，预发布说明符和生成元数据没有任何排序权重。 例如，若要打开 `>= x.y` 的功能并使其在 `x.y.z-pre` 上启动，这很有用。
- 未指定部分与零值部分相同。 (`x == x.0 == x.0.0 == x.0.0.0`).
- 整数组件中不允许含有空格。
- 仅主版本有效（`3` 等于 `3.0.0.0`）
- `+` 不允许作为整数组件中的正号（它被视为 SemVer 元数据并被忽略）

> 要比较 [TargetFramework 属性](https://docs.microsoft.com/zh-cn/visualstudio/msbuild/msbuild-target-framework-and-target-platform?view=vs-2022)，通常应使用 [IsTargetFrameworkCompatible](https://docs.microsoft.com/zh-cn/visualstudio/msbuild/property-functions?view=vs-2022#TargetFramework)，而不是提取和比较版本。 这允许比较在 `TargetFrameworkIdentifier` 和版本中不同的 `TargetFramework`。

# Items

## MSBuild Items

### 在项目文件中引用Items

```xml
// 指定分隔符
@(<ItemType>, '<separator>')

// 通配符
// ? 通配符匹配单个字符。
// * 通配符匹配零个或多个字符。
// ** 通配符序列匹配部分路径。
<CSFile Include="*.cs"/>
<VBFile Include="D:/**/*.vb"/>    

// 声明两个项
<Compile Include="MyFile.cs;MyClass.cs"/>
    
// 声明一个带有分号的项
<Compile Include="MyFile.cs%3BMyClass.cs"/>
<Compile Include="$([MSBuild]::Escape('MyFile.cs;MyClass.cs'))" /> 

// 排除指定文件
<CSFile  Include="*.cs"  Exclude="DoNotBuild.cs"/>
    
// 元数据，如果将元数据设置为空值，则可有效将其从生成中删除
<ItemGroup>
    <CSFile Include="one.cs;two.cs">
        <Culture>Fr</Culture>
    </CSFile>
</ItemGroup>    
    
// 使用Display元数据对Message任务进行批处理    
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
    <ItemGroup>
        <Stuff Include="One.cs" >
            <Display>false</Display>
        </Stuff>
        <Stuff Include="Two.cs">
            <Display>true</Display>
        </Stuff>
    </ItemGroup>
    <Target Name="Batching">
        <Message Text="@(Stuff)" Condition=" '%(Display)' == 'true' "/>
    </Target>
</Project>    
```

> 向项类型添加项时，会向该项分配一些常见元数据。 例如，所有项都具有常见元数据`%(<Filename>)`，其值是项的文件名（不带扩展名）。 有关详细信息，请参阅[常见项元数据](https://docs.microsoft.com/zh-cn/visualstudio/msbuild/msbuild-well-known-item-metadata?view=vs-2022)。

### 使用元数据转换项类型

通过使用元数据，可将项列表转换为新的项列表。 例如，可使用 `@(CppFiles -> '%(Filename).obj')` 表达式将含有表示 .cpp 文件的项的项类型 `CppFiles` 转换为相应的 .obj 文件列表 。

以下代码创建 `CultureResource` 项类型，其中含有所有带 `Culture` 元数据的 `EmbeddedResource` 项的副本。 `Culture` 元数据值将成为新的 `CultureResource.TargetDirectory` 元数据的值。

XML复制

```xml
<Target Name="ProcessCultureResources">
    <ItemGroup>
        <CultureResource Include="@(EmbeddedResource)"
            Condition="'%(EmbeddedResource.Culture)' != ''">
            <TargetDirectory>%(EmbeddedResource.Culture) </TargetDirectory>
        </CultureResource>
    </ItemGroup>
</Target>
```

有关项的更多操作，请参阅MSBuild[函数](https://docs.microsoft.com/zh-cn/visualstudio/msbuild/item-functions?view=vs-2022)和[转换](https://docs.microsoft.com/zh-cn/visualstudio/msbuild/msbuild-transforms?view=vs-2022)。

### 项定义

从.NET Framework 3.5 起，即可使用 [ItemDefinitionGroup 元素](https://docs.microsoft.com/zh-cn/visualstudio/msbuild/itemdefinitiongroup-element-msbuild?view=vs-2022)向任何项类型添加默认元数据。 与已知元数据一样，默认元数据与指定项类型的所有项关联。 可在项定义中显式重写默认元数据。 例如，下面的 XML 向 `Compile` 项“one.cs”和“three.cs”提供值为“Monday”的 `BuildDay` 元数据 。 代码会向“two.cs”项提供值为“Tuesday”的 `BuildDay` 元数据。

```xml
<ItemDefinitionGroup>
    <Compile>
        <BuildDay>Monday</BuildDay>
    </Compile>
</ItemDefinitionGroup>
<ItemGroup>
    <Compile Include="one.cs;three.cs" />
    <Compile Include="two.cs">
        <BuildDay>Tuesday</BuildDay>
    </Compile>
</ItemGroup>
```

### Targets中的ItemGroup中的Items的Properties

```xml
// 以下示例从 Compile 项类型删除所有 .config 文件。
<Target>
    <ItemGroup>
        <Compile Remove="*.config"/>
    </ItemGroup>
</Target>

// B Remove="@(A)" MatchOnMetadata="M" 匹配规则：从 B 中删除具有元数据 M 的所有项，其 M 的元数据值 V 与 A 中 M 元数据值为 V 的任何项匹配。
<Project>
  <ItemGroup>
    <A Include='a1' M1='1' M2='a' M3="e"/>
    <A Include='b1' M1='2' M2='x' M3="f"/>
    <A Include='c1' M1='3' M2='y' M3="g"/>
    <A Include='d1' M1='4' M2='b' M3="h"/>

    <B Include='a2' M1='x' m2='c' M3="m"/>
    <B Include='b2' M1='2' m2='x' M3="n"/>
    <B Include='c2' M1='2' m2='x' M3="o"/>
    <B Include='d2' M1='3' m2='y' M3="p"/>
    <B Include='e2' M1='3' m2='Y' M3="p"/>
    <B Include='f2' M1='4'        M3="r"/>
    <B Include='g2'               M3="s"/>

    <B Remove='@(A)' MatchOnMetadata='M1;M2'/>
  </ItemGroup>

  <Target Name="PrintEvaluation">
    <Message Text="%(B.Identity) M1='%(B.M1)' M2='%(B.M2)' M3='%(B.M3)'" />
  </Target>
</Project>

// 输出：
//  a2 M1='x' M2='c' M3='m'
//  e2 M1='3' M2='Y' M3='p'
//  f2 M1='4' M2='' M3='r'
//  g2 M1='' M2='' M3='s'

// msbuild 公用 sdk 中 MatchOnMetadata 的示例用法：
// 指定 MatchOnMetadata 用于匹配项之间的元数据值的字符串匹配策略（元数据名称匹配始终不区分大小写）。 可能的值为 CaseSensitive、CaseInsensitive 或 PathLike。 默认值为 CaseSensitive。
// PathLike 对如下值应用路径感知标准化：如规范化斜杠方向、忽略尾随斜杠、消除 . 和 ..，以及将当前目录的所有相对路径设置为绝对路径。
<_TransitiveItemsToCopyToOutputDirectory Remove="@(_ThisProjectItemsToCopyToOutputDirectory)" MatchOnMetadata="TargetPath" MatchOnMetadataOptions="PathLike" />

// 如果项在目标中生成，则项元素中会包含 KeepMetadata 属性。 如果指定了此属性，则只会将在由分号分隔的名称列表中指定的元数据从源项传输到目标项。 若此属性为空值，相当于不进行指定。 KeepMetadata 属性是在 .NET Framework 4.5 中引入的。
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003"
ToolsVersion="4.0">

    <ItemGroup>
        <FirstItem Include="rhinoceros">
            <Class>mammal</Class>
            <Size>large</Size>
        </FirstItem>

    </ItemGroup>
    <Target Name="MyTarget">
        <ItemGroup>
            <SecondItem Include="@(FirstItem)" KeepMetadata="Class" />
        </ItemGroup>

        <Message Text="FirstItem: %(FirstItem.Identity)" />
        <Message Text="  Class: %(FirstItem.Class)" />
        <Message Text="  Size:  %(FirstItem.Size)"  />

        <Message Text="SecondItem: %(SecondItem.Identity)" />
        <Message Text="  Class: %(SecondItem.Class)" />
        <Message Text="  Size:  %(SecondItem.Size)"  />
    </Target>
</Project>

<!--
Output:
  FirstItem: rhinoceros
    Class: mammal
    Size:  large
  SecondItem: rhinoceros
    Class: mammal
    Size:
-->

// 如果项在目标中生成，则项元素中会包含 RemoveMetadata 属性。 如果指定了此属性，则除包含在由分号分隔的名称列表中的元数据外，会将所有元数据从源项传输到目标项。 若此属性为空值，相当于不进行指定。 RemoveMetadata 属性是在 .NET Framework 4.5 中引入的。
<Project ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

    <PropertyGroup>
        <MetadataToRemove>Size;Material</MetadataToRemove>
    </PropertyGroup>

    <ItemGroup>
        <Item1 Include="stapler">
            <Size>medium</Size>
            <Color>black</Color>
            <Material>plastic</Material>
        </Item1>
    </ItemGroup>

    <Target Name="MyTarget">
        <ItemGroup>
            <Item2 Include="@(Item1)" RemoveMetadata="$(MetadataToRemove)" />
        </ItemGroup>

        <Message Text="Item1: %(Item1.Identity)" />
        <Message Text="  Size:     %(Item1.Size)" />
        <Message Text="  Color:    %(Item1.Color)" />
        <Message Text="  Material: %(Item1.Material)" />
        <Message Text="Item2: %(Item2.Identity)" />
        <Message Text="  Size:     %(Item2.Size)" />
        <Message Text="  Color:    %(Item2.Color)" />
        <Message Text="  Material: %(Item2.Material)" />
    </Target>
</Project>

<!--
Output:
  Item1: stapler
    Size:     medium
    Color:    black
    Material: plastic
  Item2: stapler
    Size:
    Color:    black
    Material:
-->


// 如果项在目标中生成，则项元素中会包含 KeepDuplicates 属性。 KeepDuplicates 是一个 Boolean 属性，当项是现有项的完全相同的副本时，该属性可指定是否应将该项添加到目标组中。

// 如果源项和目标项的 Include 值相同，但元数据不同，那么即使将 KeepDuplicates 设为 false 仍会添加项。 若此属性为空值，相当于不进行指定。 KeepDuplicates 属性是在 .NET Framework 4.5 中引入的。
<Project ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

    <ItemGroup>
        <Item1 Include="hourglass;boomerang" />
        <Item2 Include="hourglass;boomerang" />
    </ItemGroup>

    <Target Name="MyTarget">
        <ItemGroup>
            <Item1 Include="hourglass" KeepDuplicates="false" />
            <Item2 Include="hourglass" />
        </ItemGroup>

        <Message Text="Item1: @(Item1)" />
        <Message Text="  %(Item1.Identity)  Count: @(Item1->Count())" />
        <Message Text="Item2: @(Item2)" />
        <Message Text="  %(Item2.Identity)  Count: @(Item2->Count())" />
    </Target>
</Project>

<!--
Output:
  Item1: hourglass;boomerang
    hourglass  Count: 1
    boomerang  Count: 1
  Item2: hourglass;boomerang;hourglass
    hourglass  Count: 2
    boomerang  Count: 1
-->
```

> 更多元数据相关：[MSBuild 项 - MSBuild | Microsoft Docs](https://docs.microsoft.com/zh-cn/visualstudio/msbuild/msbuild-items?view=vs-2022#updating-metadata-on-items-in-an-itemgroup-outside-of-a-target)

## 选择要生成的文件

### 使用通配符指定输入

```xml
// 包括 Images 目录和子目录中的所有 .jpg 文件
Include="Images\**\*.jpg"

// 包括所有以“img”开头的 .jpg 文件
Include="Images\**\img*.jpg"

// 包括目录中名称以“jpgs”结尾的所有文件
Include="Images\**\*jpgs\*.*"
Include="Images\**\*jpgs\*"

// 包括除 Version2 目录中的 .jpg 文件以外，图像目录子目录中的所有 .jpg 文件
// 必须指定这两个属性的路径。 如果你使用绝对路径在 Include 属性中指定文件位置，那么在 Exclude 属性中也必须使用绝对路径；如果你在 Include 属性中使用相对路径，那么在 Exclude 属性中也必须使用相对路径。
<JPGFile
    Include="Images\**\*.jpg"
    Exclude = "Images\**\Version2\*.jpg"/>

// 包括除 Form2 和 Form3 以外的所有.cs 或.vb 文件
<VBFile Include="*.vb" Exclude="Form2.vb;Form3.vb"/>

// 仅在发布生成中包括文件 Formula.vb
<Compile
    Include="Formula.vb"
    Condition=" '$(Configuration)' == 'Release' " />
```

## Items Definition

*ItemDefinitionGroup* 元素紧跟在项目文件的 [Project](https://docs.microsoft.com/zh-cn/visualstudio/msbuild/project-element-msbuild?view=vs-2022) 元素之后。 项定义提供以下功能：

- 可为目标外部的项定义全局默认元数据。 也就是说，相同的元数据适用于指定类型的所有项。
- 项目类型可以有多个定义。 当其他元数据规范添加到该类型中时，最新的规范优先。 (元数据遵循与属性相同的导入顺序。)
- 元数据可具有累加性。 例如，根据要设置的属性，CDefines 值可以有条件地累积。 例如 `MT;STD_CALL;DEBUG;UNICODE`。
- 可删除元数据。
- 可使用条件来控制元数据的包含。

```xml
<ItemDefinitionGroup>
    <i>
        <m>m1</m>
        <n>n1</n>
    </i>
</ItemDefinitionGroup>
<ItemGroup>
    <i Include="a">
        <o>o1</o>
        <n>n2</n>
    </i>
</ItemGroup>
```

> XML 元素和参数名均区分大小写。 项元数据和项/属性名均不区分大小写。 因此，应将名称中仅大小写不同的 ItemDefinitionGroup 项视作同一个 ItemGroup。

### 累加性和多个定义

当添加定义或使用多个 ItemDefinitionGroup 时，请记住以下几点：

- 将其他元数据规范添加到此类型中。
- 最新规范优先。

当拥有多个 ItemDefinitionGroup 时，每个后续规范将其元数据添加至先前的定义。 例如：

```xml
// 在本例中，向“m”和“n”添加元数据“o”。
<ItemDefinitionGroup>
    <i>
        <m>m1</m>
        <n>n1</n>
    </i>
</ItemDefinitionGroup>
<ItemDefinitionGroup>
    <i>
        <o>o1</o>
    </i>
</ItemDefinitionGroup>

// 在本例中，元数据“m”先前定义的值 (m1) 被添加至新值 (m2)，因此最终值为“m1;m2”。
<ItemDefinitionGroup>
    <i>
        <m>m1</m>
    </i>
</ItemDefinitionGroup>
<ItemDefinitionGroup>
    <i>
        <m>%(m);m2</m>
    </i>
</ItemDefinitionGroup>

// 当替代先前定义的元数据时，最新规范优先。 在下列示例中，元数据“m”的最终值从“m1”变为“m1a”。
<ItemDefinitionGroup>
    <i>
        <m>m1</m>
    </i>
</ItemDefinitionGroup>
<ItemDefinitionGroup>
    <i>
        <m>m1a</m>
    </i>
</ItemDefinitionGroup>

// 可使用 ItemDefinitionGroup 中的条件来控制元数据的包含。例如：
<ItemDefinitionGroup Condition="'$(Configuration)'=='Debug'">
    <i>
        <m>m1</m>
    </i>
</ItemDefinitionGroup>

// 对项（而非定义组）而言，在先前的 ItemDefinitionGroup 中定义的元数据的引用是本地的。 也就是说，引用的范围特定于项。 例如：在示例中，项“i”在其条件中引用项“test”。 此种情况永远不会为 true，因为 MSBuild 将 ItemDefinitionGroup 中对另一个项的元数据的引用解释为空字符串。 因此，“m”将设置为“m0”。
<ItemDefinitionGroup>
    <test>
        <yes>1</yes>
    </test>
    <i>
        <m>m0</m>
        <m Condition="'%(test.yes)'=='1'">m1</m>
    </i>
</ItemDefinitionGroup>

// “m”将设置为值“m1”，作为项“yes”的条件引用项“i”的元数据值。
<ItemDefinitionGroup>
    <i>
        <m>m0</m>
        <yes>1</yes>
        <m Condition="'%(i.yes)'=='1'">m1</m>
    </i>
</ItemDefinitionGroup>

// 项“i”仍包含元数据“m”，但此时它的值为空。
<ItemDefinitionGroup>
    <i>
        <m>m1</m>
    </i>
</ItemDefinitionGroup>
<ItemDefinitionGroup>
    <i>
        <m></m>
    </i>
</ItemDefinitionGroup>

// ItemDefinitionGroup 中的默认元数据定义可自引用。 例如，以下示例使用了一个简单的元数据引用：
<ItemDefinitionGroup>
    <i>
        <m>m1</m>
        <m>%(m);m2</m>
    </i>
</ItemDefinitionGroup>

// 还可使用限定的元数据引用：
<ItemDefinitionGroup>
    <i>
        <m>m1</m>
        <m>%(i.m);m2</m>
    </i>
</ItemDefinitionGroup>

// 但是，以下项无效：
<ItemDefinitionGroup>
    <i>
        <m>m1</m>
        <m>@(x)</m>
    </i>
</ItemDefinitionGroup>

// 从 MSBuild 3.5 开始，ItemGroups 可以自引用。 例如：
<ItemGroup>
    <item Include="a">
        <m>m1</m>
        <m>%(m);m2</m>
    </item>
</ItemGroup>
```













# 日志

## 日志级别

### -verbosity (**-v**)

| Message type / Verbosity              | Quiet | Minimal | Normal | Detailed | Diagnostic |
| :------------------------------------ | :---: | :-----: | :----: | :------: | :--------: |
|                                       | -v:q  |  -v:m   |  -v:n  |   -v:d   |  -v:diag   |
| Errors                                |   ✅   |    ✅    |   ✅    |    ✅     |     ✅      |
| Warnings                              |   ✅   |    ✅    |   ✅    |    ✅     |     ✅      |
| High-importance Messages              |       |    ✅    |   ✅    |    ✅     |     ✅      |
| Normal-importance Messages            |       |         |   ✅    |    ✅     |     ✅      |
| Low-importance Messages               |       |         |        |    ✅     |     ✅      |
| Additional MSBuild-engine information |       |         |        |          |     ✅      |

## 日志文件

### 输出日志文件：-fileLogger (**fl**)

```bash
msbuild MyProject.proj -fl
```

### 日志文件参数：-fileLoggerParameters (`flp`)

```bash
msbuild MyProject.proj -t:go -fl -flp:logfile=MyProjectOutput.log;verbosity=diagnostic
msbuild MyProject.proj -t:go -fl -flp:logfile=MyProjectOutput.log;verbosity=diag
msbuild MyProject.proj -t:go -fl -flp:logfile=MyProjectOutput.log;v=diag
```

### 多日志文件

```bash
msbuild helloword.csproj -p:Configuration=Debug -v:diag -fl1 -fl2 -fl4 -flp1:logfile=normal.log;v=diag -flp2:logfile=JustErrors.log;errorsonly -flp4:logfile=JustWarning.log;waringsonly
```
