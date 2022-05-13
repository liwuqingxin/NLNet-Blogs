[toc]

> 参考文档：[MSBuild - MSBuild | Microsoft Docs](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild?view=vs-2022)，C++：[MSBuild on the command line - C++ | Microsoft Docs](https://docs.microsoft.com/en-us/cpp/build/msbuild-visual-cpp?view=msvc-170)
>
> 翻译人员：聂亮兵
>
> 日期：2022年4月12日
>
> 版本：Visual Studio 2022

> 待处理项目：
>
> 1. 整理Microsoft.Common.targets、Microsoft.Common.props、Directory.Build.props、Directory.Build.targets；
> 2. 

# 1. MSBuild

Microsoft Build Engine，也称为MSBuild。VS使用MSBuild，但是MSBuild不依赖VS。

以下的情况可以考虑在命令行使用MSBuild实现：

- 没有安装VS

- 需要运行64-bit的MSBuild

- 在多进程中进行构建，这在IDE中也可以达到相同的效果

- 需要修改编译系统，例如达到以下目的：
  - 在编译前预处理一些文件
  - 复制编译结果到不同目录
  - 对编译结果创建压缩文件
  - 做一个编译前处理，例如，需要对程序集进行版本设定

你可以考虑在VS中编写代码，在MSBuild中编译程序，也可以在开发环境中使用VS编译，在集成环境中使用MSBuild编译程序。也可以使用[.NET CLI | Microsoft Docs](https://docs.microsoft.com/en-us/dotnet/core/tools/)编译.Net Core项目。

> 关于 AZure Pipelines：[Azure Pipelines documentation - Azure DevOps | Microsoft Docs](https://docs.microsoft.com/en-us/azure/devops/pipelines/?view=azure-devops&preserve-view=true&viewFallbackFrom=vsts)
>

## 1.1. 在命令提示符处使用MSBuild

通过指定配置编译工程的命令：

```bash
MSBuild.exe MyProj.proj -property:Configuration=Debug
```

> 更多命令选项参考：[MSBuild Command-Line Reference - MSBuild | Microsoft Docs](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-command-line-reference?view=vs-2022)

## 1.2. Project File

MSBuild 使用简单且可扩展的基于 XML 的项目文件格式。 MSBuild 项目文件格式允许开发人员描述要构建的项目，以及如何为不同的操作系统和配置构建它们。此外，项目文件格式允许开发人员编写可重用的构建规则，这些规则可以分解到单独的文件中，以便可以在产品中的不同项目之间一致地执行构建。

Visual Studio 构建系统将项目特定的逻辑存储在您的项目文件本身中，并使用带有 .props 和 .targets 等扩展名的导入的 MSBuild XML 文件来定义标准构建逻辑。 .props 文件定义 MSBuild 属性，.targets 文件定义 MSBuild 目标。这些导入有时在 Visual Studio 项目文件中可见，但在较新的项目（如 .NET Core、.NET 5 和 .NET 6 项目）中，您看不到项目文件中的导入；相反，您会看到 SDK 参考。这些被称为 SDK 风格的项目。当您引用 .NET SDK 等 SDK 时，.props 和 .target 文件的导入由 SDK 隐式指定。

以下部分描述了 MSBuild 项目文件格式的一些基本元素。

### 1.2.1 Properties

属性表示可用于配置构建的键/值对。通过创建一个将属性名称作为 PropertyGroup 元素的子元素的元素来声明属性。例如，以下代码创建一个名为 BuildDir 的属性，其值为 Build。

```xml
<PropertyGroup>
    <BuildDir>Build</BuildDir>
</PropertyGroup>
```

您可以通过在元素中放置 Condition 属性来有条件地定义属性。除非条件评估为真，否则条件元素的内容将被忽略。在以下示例中，如果尚未定义 Configuration 元素，则定义它。

```xml
<Configuration  Condition=" '$(Configuration)' == '' ">Debug</Configuration>
```

可以使用语法 `$(<PropertyName>) `在整个项目文件中引用属性。例如，您可以使用`$(BuildDir)`和`$(Configuration)`来引用前面示例中的属性。

> 有关属性的详细信息，请参阅 [MSBuild Properties - MSBuild | Microsoft Docs](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-properties?view=vs-2022)。

### 1.2.2 Items

Items是构建系统的输入，通常代表文件。Items根据用户定义的Items名称分组为Items类型。这些Items类型可以用作任务的参数，这些任务使用各个Items来执行构建过程的步骤。

通过创建一个将Items类型的名称作为[ItemGroup](https://docs.microsoft.com/en-us/visualstudio/msbuild/itemgroup-element-msbuild?view=vs-2022)元素的子元素的元素，在项目文件中声明项目。例如，以下代码创建一个名为 Compile 的项类型，其中包括两个文件。

```xml
<ItemGroup>
    <Compile Include = "file1.cs"/>
    <Compile Include = "file2.cs"/>
</ItemGroup>
```

可以使用语法`@(<ItemType>) `在整个项目文件中引用Item类型。例如，示例中的Item类型将通过使用`@(Compile) `来引用。

在 MSBuild 中，元素和属性名称区分大小写。但是，属性、项目和元数据名称不是。以下示例创建Item类型 Compile、comPile 或任何其他案例变体，并为Item类型赋予值“one.cs;two.cs”。

```xml
<ItemGroup>
    <Compile Include="one.cs" />
    <Compile Include="two.cs" />
</ItemGroup>
```

> 可以使用通配符声明项目，并且可以包含更多元数据以用于更高级的构建方案。有关项目的更多信息，请参阅[Items](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-items?view=vs-2022)。

### 1.2.3 Tasks

任务是 MSBuild 项目用于执行构建操作的可执行代码单元。例如，任务可能会编译输入文件或运行外部工具。任务可以重复使用，并且可以由不同项目中的不同开发人员共享。

任务的执行逻辑以托管代码编写，并使用[UsingTask](https://docs.microsoft.com/en-us/visualstudio/msbuild/usingtask-element-msbuild?view=vs-2022)元素映射到 MSBuild。您可以通过编写实现 ITask 接口的托管类型来编写自己的任务。有关如何编写任务的更多信息，请参阅[Task writing](https://docs.microsoft.com/en-us/visualstudio/msbuild/task-writing?view=vs-2022)。

MSBuild 包括您可以修改以满足您的要求的常见任务。例如，复制文件的[Copy](https://docs.microsoft.com/en-us/visualstudio/msbuild/copy-task?view=vs-2022)、创建目录的[MakeDir](https://docs.microsoft.com/en-us/visualstudio/msbuild/makedir-task?view=vs-2022)和编译 Visual C# 源代码文件的[Csc](https://docs.microsoft.com/en-us/visualstudio/msbuild/csc-task?view=vs-2022)。有关可用任务的列表以及使用信息，请参阅[Task reference](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-task-reference?view=vs-2022)。

通过创建一个将任务名称作为 Target 元素的子元素的元素，在 MSBuild 项目文件中执行任务。任务通常接受参数，这些参数作为元素的属性传递。 MSBuild 属性和Item都可以用作参数。例如，以下代码调用 MakeDir 任务并将之前示例中声明的 BuildDir 属性的值传递给它。

```xml
<Target Name="MakeBuildDirectory">
    <MakeDir  Directories="$(BuildDir)" />
</Target>
```

> 更多关于任务的信息请参考[Tasks](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-tasks?view=vs-2022).

### 1.2.4 Targets

Targets 按特定顺序将任务组合在一起，并将项目文件的各个部分公开为构建过程的入口点。目标通常被分组为逻辑部分以增加可读性并允许扩展。将构建步骤分解为Targets可以让您从其他Targets调用构建过程的一部分，而无需将该部分代码复制到每个Targets中。例如，如果构建过程的多个入口点需要构建引用，您可以创建一个构建引用的Targets，然后从需要它的每个入口点运行该Targets。

使用[Target](https://docs.microsoft.com/en-us/visualstudio/msbuild/target-element-msbuild?view=vs-2022)元素在项目文件中声明目标。例如，以下代码创建一个名为 Compile 的目标，然后调用 Csc 任务，该任务具有前面示例中声明的项目列表。

```xml
<Target Name="Compile">
    <Csc Sources="@(Compile)" />
</Target>
```

> 在更高级的场景中，Targets可用于描述彼此之间的关系并执行依赖分析，以便如果该Targets是最新的，则可以跳过构建过程的整个部分。有关目标的更多信息，请参阅[Targets](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-targets?view=vs-2022)。

## 1.3. Build logs

您可以将构建错误、警告和消息记录到控制台或其他输出设备。

> 有关详细信息，请参阅[Obtaining build logs](https://docs.microsoft.com/en-us/visualstudio/msbuild/obtaining-build-logs-with-msbuild?view=vs-2022)和[Logging in MSBuild](https://docs.microsoft.com/en-us/visualstudio/msbuild/logging-in-msbuild?view=vs-2022)。

## 1.4. Use MSBuild in Visual Studio

Visual Studio 使用 MSBuild 项目文件格式来存储有关托管项目的生成信息。使用 Visual Studio 界面添加或更改的项目设置反映在为每个项目生成的 .*proj 文件中。 Visual Studio 使用 MSBuild 的托管实例来构建托管项目。这意味着可以在 Visual Studio 中或在命令提示符下（即使未安装 Visual Studio）构建托管项目，并且结果将是相同的。

> 有关如何在 Visual Studio 中使用 MSBuild 的教程，请参阅[Walkthrough: Using MSBuild](https://docs.microsoft.com/en-us/visualstudio/msbuild/walkthrough-using-msbuild?view=vs-2022)。

## 1.5. Multitargetiing

通过使用 Visual Studio，您可以编译应用程序以在多个 .NET Framework 版本中的任何一个上运行。例如，您可以编译应用程序以在 32 位平台上的 .NET Framework 2.0 上运行，也可以编译相同的应用程序以在 64 位平台上的 .NET Framework 4.5 上运行。编译到多个框架的能力称为多目标。

这些是多目标的一些好处：

* 您可以开发面向 .NET Framework 早期版本（例如 3.5 和 4.7.2）的应用程序。
* 您可以定位框架配置文件，它是目标框架的预定义子集。
* 如果发布了当前版本的 .NET Framework 的服务包，您可以将其作为目标。
* 多目标保证应用程序仅使用目标框架和平台中可用的功能。

> 有关详细信息，请参阅[Multitargeting](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-multitargeting-overview?view=vs-2022)。

# 2、MSBuild Concepts

MSBuild 提供了一个基本的 XML 架构，您可以使用它来控制构建平台如何构建软件。要指定构建中的组件以及如何构建它们，请使用 MSBuild 的这四个部分：Properties、Items、Task和Targets。

## 2.1. How MSBuild builds projects

MSBuild 实际上是如何工作的？在本文中，您将了解 MSBuild 如何处理您的项目文件，无论是从 Visual Studio 调用，还是从命令行或脚本调用。了解 MSBuild 的工作原理可以帮助您更好地诊断问题并更好地自定义构建过程。本文描述了构建过程，并且在很大程度上适用于所有项目类型。

完整的构建过程包括构建项目的目标和任务的[initial startup](https://docs.microsoft.com/en-us/visualstudio/msbuild/build-process-overview?view=vs-2022#startup)，[evaluation](https://docs.microsoft.com/en-us/visualstudio/msbuild/build-process-overview?view=vs-2022#evaluation-phase)和[execution](https://docs.microsoft.com/en-us/visualstudio/msbuild/build-process-overview?view=vs-2022#execution-phase) 。除了这些输入之外，外部导入还定义了构建过程的详细信息，包括[standard imports](https://docs.microsoft.com/en-us/visualstudio/msbuild/build-process-overview?view=vs-2022#standard-imports)（如 Microsoft.Common.targets）和解决方案或项目级别的[user-configurable imports](https://docs.microsoft.com/en-us/visualstudio/msbuild/build-process-overview?view=vs-2022#user-configurable-imports)。

### 2.1.1 Startup

MSBuild 可以通过 Microsoft.Build.dll 中的 MSBuild 对象模型从 Visual Studio 调用，或者通过直接在命令行或脚本中调用可执行文件，例如在 CI 系统中。在任何一种情况下，影响构建过程的输入都包括项目文件（或 Visual Studio 内部的项目对象）、可能的解决方案文件、环境变量和命令行开关或其等效对象模型。在启动阶段，命令行选项或对象模型等效项用于配置 MSBuild 设置，例如配置记录器。使用 -property 或 -p 开关在命令行上设置的属性设置为全局属性，这些属性会覆盖将在项目文件中设置的任何值，即使稍后会读入项目文件。

接下来的部分是关于输入文件的，例如解决方案文件或项目文件。

#### 2.1.1.1 Solutions and projects

MSBuild 实例可能包含一个项目，也可能包含作为解决方案一部分的多个项目。解决方案文件不是 MSBuild XML 文件，但 MSBuild 将其解释为了解需要为给定配置和平台设置构建的所有项目。当 MSBuild 处理此 XML 输入时，它被称为解决方案构建。它具有一些可扩展点，允许您在每个解决方案构建中运行某些内容，但由于此构建与单个项目构建是分开运行的，因此解决方案构建中的属性或目标定义设置与每个项目构建无关。

> 您可以在[Customize the solution build](https://docs.microsoft.com/en-us/visualstudio/msbuild/customize-your-build?view=vs-2022#customize-the-solution-build)中了解如何扩展解决方案构建。

#### 2.1.1.2 Visual Studio builds vs. MSBuild.exe builds

在 Visual Studio 中构建项目与直接调用 MSBuild（通过 MSBuild 可执行文件或使用 MSBuild 对象模型开始构建）之间存在一些显着差异。 Visual Studio 管理 Visual Studio 构建的项目构建顺序；它仅在单个项目级别调用 MSBuild，当它调用时，会设置几个布尔属性（BuildingInsideVisualStudio、BuildProjectReferences），这些属性会显着影响 MSBuild 的功能。在每个项目内部，执行与通过 MSBuild 调用时的执行方式相同，但在引用的项目中会出现差异。在 MSBuild 中，当需要引用的项目时，实际上会发生构建；也就是说，它运行任务和工具，并生成输出。当 Visual Studio 构建找到引用的项目时，MSBuild 仅返回来自引用的项目的预期输出；它允许 Visual Studio 控制其他项目的构建。 Visual Studio 确定构建顺序并单独调用 MSBuild（根据需要），所有这些都完全在 Visual Studio 的控制之下。

当使用解决方案文件调用 MSBuild 时，会出现另一个区别，MSBuild 会解析解决方案文件，创建标准 XML 输入文件，对其进行评估，然后将其作为项目执行。解决方案构建在任何项目之前执行。从 Visual Studio 构建时，这些都不会发生； MSBuild 永远不会看到解决方案文件。因此，解决方案构建自定义（使用 before.SolutionName.sln.targets 和 after.SolutionName.sln.targets）仅适用于 MSBuild.exe 或对象模型驱动，不适用于 Visual Studio 构建。

#### 2.1.1.3 Project SDKs

MSBuild 项目文件的 SDK 功能相对较新。在此更改之前，项目文件显式导入了定义特定项目类型的构建过程的 .targets 和 .props 文件。

> .NET Core 项目导入适合它们的 .NET SDK 版本。请参阅概述[.NET Core project SDKs](https://docs.microsoft.com/en-us/dotnet/core/project-sdk/overview) 和对[properties](https://docs.microsoft.com/en-us/dotnet/core/project-sdk/msbuild-props)的引用。

### 2.1.2 Evaluation phase

本节讨论如何处理和解析这些输入文件以生成内存中的对象，这些对象决定将构建什么。

评估阶段的目的是根据输入的 XML 文件和本地环境在内存中创建对象结构。评估阶段包括六个处理输入文件的过程，例如项目 XML 文件或导入的 XML 文件，通常命名为 .props 或 .targets 文件，具体取决于它们主要设置属性还是定义构建目标。每次传递都会构建内存中对象的一部分，这些对象稍后会在执行阶段用于构建项目，但在评估阶段不会发生实际的构建操作。在每个通道中，元素按照它们出现的顺序进行处理。

评估阶段的通行证如下：

- 评估 environment variables
- 评估 imports 和 properties
- 评估 item definitions
- 评估 item
- 评估 UsingTask 元素
- 评估 targets

> 这些通道的顺序具有重要意义，并且在自定义项目文件时知道这一点很重要。请参阅[Property and item evaluation order](https://docs.microsoft.com/en-us/visualstudio/msbuild/comparing-properties-and-items?view=vs-2022#property-and-item-evaluation-order)。

#### 2.1.2.1 Evaluate environment variables

在此阶段，环境变量用于设置等效属性。例如，PATH 环境变量作为属性`$(PATH)`提供。从命令行或脚本运行时，命令环境照常使用，而从 Visual Studio 运行时，使用 Visual Studio 启动时生效的环境。

#### 2.1.2.2 Evaluate imports and properties

在这个阶段，整个输入 XML 被读入，包括项目文件和整个导入链。 MSBuild 创建一个内存中的 XML 结构，该结构表示项目的 XML 和所有导入的文件。此时，不在目标中的属性将被评估和设置。

由于 MSBuild 在其进程的早期读取所有 XML 输入文件，因此在构建过程中对这些输入的任何更改都不会影响当前构建。

任何目标之外的属性的处理方式与目标内的属性不同。在此阶段，仅评估在任何目标之外定义的属性。

因为属性在属性传递中是按顺序处理的，所以输入中任何点的属性都可以访问输入中较早出现的属性值，但不能访问稍后出现的属性。

因为属性是在评估Item之前处理的，所以您无法在属性传递的任何部分访问任何Item的值。

#### 2.1.2.3 Evaluate item definitions

在此阶段，解释Item定义并创建这些定义的内存表示。

#### 2.1.2.4 Evaluate items

目标内定义的Items的处理方式与任何目标外的Items不同。在此阶段，将处理任何目标之外的Items及其相关元数据。Items定义设置的元数据被Items上设置的元数据覆盖。

因为Items是按照它们出现的顺序处理的，所以您可以引用之前定义的Items，但不能引用稍后出现的Items。

因为Items传递是在属性传递之后，所以如果Items在任何目标之外定义，则可以访问任何属性，而不管属性定义是否稍后出现。

#### 2.1.2.5 Evaluate UsingTask elements

在此阶段，读取 UsingTask 元素，并声明任务以供稍后在执行阶段使用。

#### 2.1.2.6 Evaluate targets

在这个阶段，所有目标对象结构都在内存中创建，为执行做准备。没有实际执行发生。

### 2.1.3 Execution phase

在执行阶段，目标被排序并运行，所有的任务都被执行。但首先，在目标中定义的属性和Items在一个阶段中按照它们出现的顺序一起评估。处理顺序与处理不在目标中的属性和Items的方式明显不同：首先是所有属性，然后是所有Items，分别在不同的通道中。目标中的属性和Items的更改可以在更改它们的目标之后观察到。

#### 2.1.3.1 Target build order

在单个项目中，目标按顺序执行。核心问题是如何确定构建所有内容的顺序，以便使用依赖项以正确的顺序构建目标。

目标构建顺序由在每个目标上使用 BeforeTargets、DependsOnTargets 和 AfterTargets 属性确定。如果较早的目标修改了在这些属性中引用的属性，则在执行较早的目标期间可能会影响较晚目标的顺序。

[Determine the target build order](https://docs.microsoft.com/en-us/visualstudio/msbuild/target-build-order?view=vs-2022#determine-the-target-build-order)中描述了排序规则。该过程由包含要构建的目标的堆栈结构确定。该任务顶部的目标开始执行，如果它依赖于其他任何东西，那么这些目标将被压入堆栈顶部，并开始执行。当有一个没有任何依赖关系的目标时，它会执行完成并且其父目标恢复。

#### 2.1.3.2 Project References

MSBuild 可以采用两种代码路径，一种是此处描述的正常路径，另一种是下一节中描述的图形选项。

单个项目通过 ProjectReference 项指定它们对其他项目的依赖关系。当堆栈顶部的项目开始构建时，它会到达 ResolveProjectReferences 目标的执行点，这是在公共目标文件中定义的标准目标。

ResolveProjectReferences 使用 ProjectReference 项的输入调用 MSBuild 任务以获取输出。 ProjectReference 项被转换为本地项，例如 Reference。当前项目的 MSBuild 执行阶段暂停，而执行阶段开始处理引用的项目（评估阶段根据需要首先完成）。仅在您开始构建依赖项目之后才会构建引用的项目，因此这会创建一个项目构建树。

Visual Studio 允许在解决方案 (.sln) 文件中创建项目依赖项。依赖项在解决方案文件中指定，并且仅在构建解决方案或在 Visual Studio 中构建时才会受到尊重。如果您构建单个项目，则会忽略这种类型的依赖关系。解决方案引用由 MSBuild 转换为 ProjectReference 项，然后以相同的方式处理。

#### 2.1.3.3 Graph option

如果您指定图形构建开关（-graphBuild 或 -graph），ProjectReference 将成为 MSBuild 使用的一流概念。 MSBuild 将解析所有项目并构建构建顺序图，即项目的实际依赖关系图，然后遍历该图以确定构建顺序。与单个项目中的目标一样，MSBuild 确保引用的项目是在它们所依赖的项目之后构建的。

### 2.1.4 Parallel execution

如果使用多处理器支持（-maxCpuCount 或 -m 开关），MSBuild 会创建节点，这些节点是使用可用 CPU 内核的 MSBuild 进程。每个项目都提交到一个可用节点。在一个节点内，单个项目构建顺序执行。

可以通过设置布尔变量[BuildInParallel](https://docs.microsoft.com/en-us/dotnet/api/microsoft.build.tasks.msbuild.buildinparallel)来启用任务以进行并行执行，该变量是根据 MSBuild 中 $(BuildInParallel) 属性的值设置的。对于启用了并行执行的任务，工作调度程序管理节点并将工作分配给节点。

> 请参阅[Building multiple projects in parallel with MSBuild](https://docs.microsoft.com/en-us/visualstudio/msbuild/building-multiple-projects-in-parallel-with-msbuild?view=vs-2022)

### 2.1.5 Standard imports

Microsoft.Common.props 和 Microsoft.Common.targets 均由 .NET 项目文件（在 SDK 样式项目中显式或隐式）导入，并且位于 Visual Studio 安装的 MSBuild\Current\bin 文件夹中。 

> C++ 项目有自己的导入层次结构；请参阅[MSBuild Internals for C++ projects](https://docs.microsoft.com/en-us/cpp/build/reference/msbuild-visual-cpp-overview)。

Microsoft.Common.props 文件设置您可以覆盖的默认值。它在项目文件的开头（显式或隐式）导入。这样，您的项目设置会出现在默认值之后，因此它们会覆盖它们。

Microsoft.Common.targets 文件及其导入的目标文件定义了 .NET 项目的标准构建过程。它还提供了可用于自定义构建的扩展点。

在实现中，Microsoft.Common.targets 是一个导入 Microsoft.Common.CurrentVersion.targets 的瘦包装器。此文件包含标准属性的设置，并定义构建过程的实际目标。 Build 目标在这里定义，但实际上它本身是空的。但是，构建目标包含 DependsOnTargets 属性，该属性指定构成实际构建步骤的各个目标，即 BeforeBuild、CoreBuild 和 AfterBuild。 Build 目标定义如下：

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

BeforeBuild 和 AfterBuild 是扩展点。它们在 Microsoft.Common.CurrentVersion.targets 文件中为空，但项目可以为其自己的 BeforeBuild 和 AfterBuild 目标提供需要在主构建过程之前或之后执行的任务。 AfterBuild 在无操作目标 Build 之前运行，因为 AfterBuild 出现在 Build 目标的 DependsOnTargets 属性中，但它发生在 CoreBuild 之后。

CoreBuild 目标包含对构建工具的调用，如下所示：

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

下表描述了这些目标；一些目标仅适用于某些项目类型。

| Target                          | Description                                                  |
| :------------------------------ | :----------------------------------------------------------- |
| BuildOnlySettings               | Settings for real builds only, not for when MSBuild is invoked on project load by Visual Studio. |
| PrepareForBuild                 | Prepare the prerequisites for building                       |
| PreBuildEvent                   | Extension point for projects to define tasks to execute before build |
| ResolveProjectReferences        | Analyze project dependencies and build referenced projects   |
| ResolveAssemblyReferences       | Locate referenced assemblies.                                |
| ResolveReferences               | Consists of `ResolveProjectReferences` and `ResolveAssemblyReferences` to find all the dependencies |
| PrepareResources                | Process resource files                                       |
| ResolveKeySource                | Resolve the strong name key used to sign the assembly and the certificate used to sign the [ClickOnce](https://docs.microsoft.com/en-us/visualstudio/deployment/clickonce-security-and-deployment?view=vs-2022) manifests. |
| Compile                         | Invokes the compiler                                         |
| ExportWindowsMDFile             | Generate a [WinMD](https://docs.microsoft.com/en-us/uwp/winrt-cref/winmd-files) file from the WinMDModule files generated by the compiler. |
| UnmanagedUnregistration         | Remove/clean the [COM Interop](https://docs.microsoft.com/en-us/dotnet/standard/native-interop/cominterop) registry entries from a previous build |
| GenerateSerializationAssemblies | Generate an XML serialization assembly using [sgen.exe](https://docs.microsoft.com/en-us/dotnet/standard/serialization/xml-serializer-generator-tool-sgen-exe). |
| CreateSatelliteAssemblies       | Create one satellite assembly for every unique culture in the resources. |
| Generate Manifests              | Generates [ClickOnce](https://docs.microsoft.com/en-us/visualstudio/deployment/clickonce-security-and-deployment?view=vs-2022) application and deployment manifests or a native manifest. |
| GetTargetPath                   | Return an item containing the build product (executable or assembly) for this project, with metadata. |
| PrepareForRun                   | Copy the build outputs to the final directory if they've changed. |
| UnmanagedRegistration           | Set registry entries for [COM Interop](https://docs.microsoft.com/en-us/dotnet/standard/native-interop/cominterop) |
| IncrementalClean                | Remove files that were produced in a prior build but weren't produced in the current build. This is necessary to make `Clean` work in incremental builds. |
| PostBuildEvent                  | Extension point for projects to define tasks to run after build |

上表中的许多目标都可以在特定于语言的导入中找到，例如 Microsoft.CSharp.targets。此文件定义特定于 C# .NET 项目的标准构建过程中的步骤。例如，它包含实际调用 C# 编译器的 Compile 目标。

### 2.1.6 User-configurable imports

除了标准导入之外，您还可以添加几个导入来自定义构建过程。

- Directory.Build.props
- Directory.Build.targets

这些文件由它们下任何子文件夹中的任何项目的标准导入读入。这通常在解决方案级别用于控制解决方案中的所有项目的设置，但也可能在文件系统中更高，直到驱动器的根目录。

Directory.Build.props 文件由 Microsoft.Common.props 导入，因此其中定义的属性在项目文件中可用。它们可以在项目文件中重新定义，以在每个项目的基础上自定义值。 Directory.Build.targets 文件在项目文件之后被读入。它通常包含目标，但您也可以在此处定义不希望单个项目重新定义的属性。

### 2.1.7 Customizations in a project file

当你在解决方案资源管理器、属性窗口或项目属性中进行更改时，Visual Studio 会更新你的项目文件，但你也可以通过直接编辑项目文件来进行自己的更改。

许多构建行为可以通过设置 MSBuild 属性来配置，或者在项目文件中设置项目的本地设置，或者如上一节所述，通过创建 Directory.Build.props 文件来全局设置整个项目文件夹的属性和解决方案。对于命令行上的临时构建或脚本，您还可以使用命令行上的 /p 选项来设置特定调用 MSBuild 的属性。

> 有关可以设置的属性的信息，请参阅[Common MSBuild project properties](https://docs.microsoft.com/en-us/visualstudio/msbuild/common-msbuild-project-properties?view=vs-2022)。

### 2.1.8 Next steps

除了此处描述的扩展点之外，MSBuild 过程还有其他几个扩展点。

> 请参阅[Customize your build](https://docs.microsoft.com/en-us/visualstudio/msbuild/customize-your-build?view=vs-2022)。以及[How to extend the Visual Studio build process](https://docs.microsoft.com/en-us/visualstudio/msbuild/how-to-extend-the-visual-studio-build-process?view=vs-2022)。

## 2.2. MSBuild properties

属性是可用于配置构建的名称-值对。属性对于将值传递给任务、评估条件以及存储将在整个项目文件中引用的值很有用。

### 2.2.1 Define and reference properties in a project file

通过创建一个将属性名称作为[PropertyGroup](https://docs.microsoft.com/en-us/visualstudio/msbuild/propertygroup-element-msbuild?view=vs-2022)元素的子元素的元素来声明属性。例如，以下 XML 创建一个名为 BuildDir 的属性，其值为 Build。

```xml
<PropertyGroup>
    <BuildDir>Build</BuildDir>
</PropertyGroup>x
```

在整个项目文件中，使用语法 `$(<PropertyName>)` 引用属性。例如，上一个示例中的属性是使用 `$(BuildDir) `引用的。
可以通过重新定义属性来更改属性值。可以使用以下 XML 为 BuildDir 属性赋予一个新值：

```xml
<PropertyGroup>
    <BuildDir>Alternate</BuildDir>
</PropertyGroup>
```

属性按照它们在项目文件中出现的顺序进行评估。 BuildDir 的新值必须在分配旧值之后声明。

### 2.2.2 Reserved properties

MSBuild 保留一些属性名称来存储有关项目文件和 MSBuild 二进制文件的信息。这些属性通过使用 $ 表示法来引用，就像任何其他属性一样。例如，`$(MSBuildProjectFile)` 返回项目文件的完整文件名，包括文件扩展名。

> 有关详细信息，请参阅如何：[How to: Reference the name or location of the project file](https://docs.microsoft.com/en-us/visualstudio/msbuild/how-to-reference-the-name-or-location-of-the-project-file?view=vs-2022)以及 [MSBuild reserved and well-known properties](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-reserved-and-well-known-properties?view=vs-2022)。

### 2.2.3 Environment properties

您可以像引用保留属性一样引用项目文件中的环境变量。例如，要在项目文件中使用 PATH 环境变量，请使用 $(Path)。如果项目包含与环境属性同名的属性定义，则项目中的属性将覆盖环境变量的值。

每个 MSBuild 项目都有一个隔离的环境块：它只能看到对自己的块的读取和写入。 MSBuild 仅在初始化属性集合时、在评估或构建项目文件之前读取环境变量。之后，环境属性是静态的，即每个生成的工具都以相同的名称和值开始。

要从衍生工具中获取环境变量的当前值，请使用[Property functions](https://docs.microsoft.com/en-us/visualstudio/msbuild/property-functions?view=vs-2022) System.Environment.GetEnvironmentVariable。但是，首选方法是使用任务参数[EnvironmentVariables](https://docs.microsoft.com/en-us/dotnet/api/microsoft.build.utilities.tooltask.environmentvariables)。在此字符串数组中设置的环境属性可以传递给生成的工具，而不会影响系统环境变量。

> 并非所有环境变量都被读入成为初始属性。任何名称不是有效 MSBuild 属性名称（例如“386”）的环境变量都会被忽略。

> 有关更多信息，请参阅[How to: Use environment variables in a build](https://docs.microsoft.com/en-us/visualstudio/msbuild/how-to-use-environment-variables-in-a-build?view=vs-2022)。

### 2.2.4 Registry properties

您可以使用以下语法读取系统注册表值，其中 `Hive`是注册表配置单元（例如 **HKEY_LOCAL_MACHINE**），`MyKey` 是键名，`MySubKey` 是子键名，`Value`是子键的值。

```xml
$(registry:Hive\MyKey\MySubKey@Value)
```

要获取默认子键值，请省略 Value。

```xml
$(registry:Hive\MyKey\MySubKey)
```

此注册表值可用于初始化构建属性。例如，要创建表示 Visual Studio Web 浏览器主页的生成属性，请使用以下代码：

```xml
<PropertyGroup>
  <VisualStudioWebBrowserHomePage>
    $(registry:HKEY_CURRENT_USER\Software\Microsoft\VisualStudio\14.0\WebBrowser@HomePage)
  </VisualStudioWebBrowserHomePage>
<PropertyGroup>
```

> In the .NET SDK version of MSBuild (`dotnet build`), registry properties are not supported.

### 2.2.5 [Global properties

MSBuild 允许您使用 -property（或 -p）开关在命令行上设置属性。这些全局属性值会覆盖在项目文件中设置的属性值。这包括环境属性，但不包括无法更改的保留属性。

以下示例将全局 Configuration 属性设置为 DEBUG。

```bash
msbuild.exe MyProj.proj -p:Configuration=DEBUG
```

还可以使用 MSBuild 任务的 Properties 属性为多项目构建中的子项目设置或修改全局属性。全局属性也会转发到子项目，除非 MSBuild 任务的 RemoveProperties 属性用于指定不转发的属性列表。

> 有关详细信息，请参阅 [MSBuild task](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-task?view=vs-2022)。

如果您在项目标记中使用 TreatAsLocalProperty 特性指定属性，则该全局属性值不会覆盖在项目文件中设置的属性值。

> 有关详细信息，请参阅[Project element (MSBuild)](https://docs.microsoft.com/en-us/visualstudio/msbuild/project-element-msbuild?view=vs-2022) 和[How to: Build the same source files with different options](https://docs.microsoft.com/en-us/visualstudio/msbuild/how-to-build-the-same-source-files-with-different-options?view=vs-2022)。

### 2.2.6 Property functions

从 .NET Framework 版本 4 开始，您可以使用属性函数来评估您的 MSBuild 脚本。您可以在构建脚本中读取系统时间、比较字符串、匹配正则表达式以及执行许多其他操作，而无需使用 MSBuild 任务。

您可以使用字符串（实例）方法对任何属性值进行操作，并且可以调用许多系统类的静态方法。例如，您可以将构建属性设置为今天的日期，如下所示。

```xml
<Today>$([System.DateTime]::Now.ToString("yyyy.MM.dd"))</Today>
```

> 有关详细信息和属性函数列表，请参阅[Property functions](https://docs.microsoft.com/en-us/visualstudio/msbuild/property-functions?view=vs-2022).

### 2.2.7 Create properties during execution

位于 Target 元素之外的属性在构建的评估阶段被赋值。在后续执行阶段，可以按如下方式创建或修改属性：

- 任何任务都可以发出属性。要发出属性，[Task](https://docs.microsoft.com/en-us/visualstudio/msbuild/task-element-msbuild?view=vs-2022)元素必须有一个具有 PropertyName 属性的子[Output](https://docs.microsoft.com/en-us/visualstudio/msbuild/output-element-msbuild?view=vs-2022)元素。

- [CreateProperty](https://docs.microsoft.com/en-us/visualstudio/msbuild/createproperty-task?view=vs-2022)任务可以发出一个属性。此用法已弃用。

- 从 .NET Framework 3.5 开始，Target 元素可能包含 PropertyGroup 元素，这些元素可能包含属性声明。

### 2.2.8 Store XML in properties

属性可以包含任意 XML，这有助于将值传递给任务或显示日志信息。以下示例显示了 ConfigTemplate 属性，该属性的值包含 XML 和其他属性引用。 MSBuild 使用它们各自的属性值替换属性引用。属性值按它们出现的顺序分配。因此，在此示例中，应该已经定义了 $(MySupportedVersion)、$(MyRequiredVersion) 和 $(MySafeMode)。

```xml
<PropertyGroup>
    <ConfigTemplate>
        <Configuration>
            <Startup>
                <SupportedRuntime
                    ImageVersion="$(MySupportedVersion)"
                    Version="$(MySupportedVersion)"/>
                <RequiredRuntime
                    ImageVersion="$(MyRequiredVersion)"
                    Version="$(MyRequiredVersion)"
                    SafeMode="$(MySafeMode)"/>
            </Startup>
        </Configuration>
    </ConfigTemplate>
</PropertyGroup>
```

## 2.3. How to: Use environment variables in a build

构建项目时，通常需要使用不在项目文件或构成项目的文件中的信息来设置构建选项。此信息通常存储在环境变量中。

### 2.3.1 Reference environment variables

Microsoft Build Engine (MSBuild) 项目文件可以使用所有环境变量作为属性。

> 如果项目文件包含与环境变量同名的属性的显式定义，则项目文件中的属性将覆盖环境变量的值。

#### 2.3.1.1 To use an environment variable in an MSBuild project

引用环境变量的方式与在项目文件中声明的变量相同。例如，以下代码引用 BIN_PATH 环境变量：

```xml
<FinalOutput>$(BIN_PATH)\MyAssembly.dll</FinalOutput>
```

如果未设置环境变量，您可以使用`Condition`属性为属性提供默认值。

#### 2.3.1.2 To provide a default value for a property

仅当属性没有值时，才对属性使用 Condition 属性来设置值。例如，仅当未设置 ToolsPath 环境变量时，以下代码才将 ToolsPath 属性设置为 c:\tools：

```xml
<ToolsPath Condition="'$(TOOLSPATH)' == ''">c:\tools</ToolsPath>
```

> 属性名称不区分大小写，因此 $(ToolsPath) 和 $(TOOLSPATH) 都引用相同的属性或环境变量。

### 2.3.2 Examples

以下项目文件使用环境变量来指定目录的位置。

```xml
<Project DefaultTargets="FakeBuild">
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

## 2.4. How to: Reference the name or location of the project file

您可以在项目文件本身中使用项目的名称或位置，而无需创建自己的属性。 MSBuild 提供了引用项目文件名和与项目相关的其他属性的保留属性。有关保留属性的详细信息，请参阅[MSBuild reserved and well-known properties](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-reserved-and-well-known-properties?view=vs-2022)。

### 2.4.1 Use the project properties

MSBuild 提供了一些保留属性，您可以在项目文件中使用这些属性，而无需每次都定义它们。例如，保留属性 MSBuildProjectName 提供对项目文件名的引用。保留属性 MSBuildProjectDirectory 提供对项目文件位置的引用。

使用 $() 表示法引用项目文件中的属性，就像使用任何属性一样。例如：

```xml
<CSC Sources = "@(CSFile)"
    OutputAssembly = "$(MSBuildProjectName).exe"/>
</CSC>
```

使用保留属性的一个优点是对项目文件名的任何更改都会自动合并。下次您构建项目时，输出文件将使用新名称，您无需采取进一步操作。

> 有关在文件或项目引用中使用特殊字符的详细信息，请参阅[MSBuild special characters](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-special-characters?view=vs-2022)。

> 保留的属性不能在项目文件中重新定义。

### 2.4.2 Example1

以下示例项目文件引用项目名称作为保留属性以指定输出名称。

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



### 2.4.3 Example2

以下示例项目文件使用 MSBuildProjectDirectory 保留属性在项目文件位置创建文件的完整路径。该示例使用 [Property function](https://docs.microsoft.com/en-us/visualstudio/msbuild/property-functions?view=vs-2022)语法调用静态 .NET Framework 方法[System.IO.Path.Combine](https://docs.microsoft.com/en-us/dotnet/api/system.io.path.combine)。

```xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

    <!-- Build the path to a file in the root of the project -->
    <PropertyGroup>
        <NewFilePath>$([System.IO.Path]::Combine($(MSBuildProjectDirectory), `BuildInfo.txt`))</NewFilePath>
    </PropertyGroup>
</Project>
```

## 2.5. How to: Build the same source files with different options

构建项目时，您经常使用不同的构建选项编译相同的组件。例如，您可以创建一个带有符号信息的调试版本或一个没有符号信息但启用优化的发布版本。或者您可以构建一个项目以在特定平台上运行，例如 x86 或 x64。在所有这些情况下，大多数构建选项都保持不变。只更改了几个选项来控制构建配置。使用 MSBuild，您可以使用属性和条件来创建不同的构建配置。

### 2.5.1  Use properties to control build settings

Property 元素定义了一个在项目文件中被多次引用的变量，例如临时目录的位置，或者用于设置在多个配置中使用的属性的值，例如 Debug 构建和 Release 构建。

> 有关属性的详细信息，请参阅 [MSBuild properties](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-properties?view=vs-2022)。

您可以使用属性来更改构建的配置，而无需更改项目文件。 Property 元素和 PropertyGroup 元素的 Condition 属性允许您更改属性的值。

> 有关 MSBuild 条件的详细信息，请参阅[Conditions](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-conditions?view=vs-2022)。

#### 2.5.1.1 To set a group of properties that depends on another property

在 PropertyGroup 元素中使用 Condition 属性，类似于以下内容：

```xml
<PropertyGroup Condition="'$(Flavor)'=='DEBUG'">
    <DebugType>full</DebugType>
    <Optimize>no</Optimize>
</PropertyGroup>
```

#### 2.5.1.2 To define a property that depends on another property

在 Property 元素中使用 Condition 属性，类似于以下内容：

```xml
<DebugType Condition="'$(Flavor)'=='DEBUG'">full</DebugType>
```

### 2.5.2 Specify properties on the command line

一旦您的项目文件被编写为接受多个配置，您就需要能够在构建项目时更改这些配置。 MSBuild 通过允许使用 -property 或 -p 开关在命令行上指定属性来提供此功能。

#### 2.5.2.1 To set a group of properties that depends on another property

将 -property 开关与属性和属性值一起使用。例如：

```bash
msbuild file.proj -property:Flavor=Debug
Msbuild file.proj -p:Flavor=Debug
```

#### 2.5.2.2 To define a property that depends on another property

对属性和属性值多次使用 -property 或 -p 开关，或者使用一个 -property 或 -p 开关并用分号 (;) 分隔多个属性。例如：

```bash
msbuild file.proj -p:Flavor=Debug;Platform=x86
msbuild file.proj -p:Flavor=Debug -p:Platform=x86
```

环境变量也被视为属性，并由 MSBuild 自动合并。

> 有关使用环境变量的更多信息，请参阅[How to: Use environment variables in a build](https://docs.microsoft.com/en-us/visualstudio/msbuild/how-to-use-environment-variables-in-a-build?view=vs-2022)。

在命令行上指定的属性值优先于在项目文件中为同一属性设置的任何值，并且项目文件中的值优先于环境变量中的值。

您可以通过在项目标记中使用 TreatAsLocalProperty 属性来更改此行为。对于与该属性一起列出的属性名称，在命令行中指定的属性值不会优先于项目文件中的值。您可以在本主题后面找到一个示例。

### 2.5.3 Example1

以下代码示例“Hello World”项目包含两个新的属性组，可用于创建 Debug 构建和 Release 构建。要构建此项目的调试版本，请键入：

```bash
msbuild consolehwcs1.proj -p:flavor=debug
```

要构建此项目的零售版本，请键入：

```bash
msbuild consolehwcs1.proj -p:flavor=retail
```

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

### 2.5.4 Example2

以下示例说明了如何使用 TreatAsLocalProperty 属性。 Color 属性在项目文件中的值为 Blue，在命令行中为 Green。在项目标记中使用 TreatAsLocalProperty="Color" 时，命令行属性（绿色）不会覆盖项目文件中定义的属性（蓝色）。

要构建项目，请输入以下命令：

```bash
msbuild colortest.proj -t:go -property:Color=Green
```

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

## 2.6. Property functions

属性函数是对出现在 MSBuild 属性定义中的 .NET Framework 方法的调用。与任务不同，属性函数可以在目标之外使用。每当属性或项展开时，即在任何目标运行任何目标之外的属性和项之前，或者在评估目标时，对目标内的属性组和项组评估属性函数。

在不使用 MSBuild 任务的情况下，您可以在构建脚本中读取系统时间、比较字符串、匹配正则表达式以及执行其他操作。 MSBuild 将尝试将字符串转换为数字和数字转换为字符串，并根据需要进行其他转换。

从属性函数返回的字符串值有[special characters](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-special-characters?view=vs-2022)转义。如果您希望将该值视为直接放入项目文件中，请使用`$([MSBuild]::Unescape())` 取消转义特殊字符。

属性函数可用于 .NET Framework 4 及更高版本。

### 2.6.1 Property function syntax

这是三种属性函数；每个函数都有不同的语法：

- 字符串（实例）属性函数

- 静态属性函数

- MSBuild 属性函数

#### 2.6.1.1 String property functions

所有构建属性值都只是字符串值。您可以使用字符串（实例）方法对任何属性值进行操作。例如，您可以使用以下代码从表示完整路径的构建属性中提取驱动器名称（前三个字符）：

```xml
$(ProjectOutputFolder.Substring(0,3))
```

#### 2.6.2.2 Static property functions

在您的构建脚本中，您可以访问许多系统类的静态属性和方法。要获取静态属性的值，请使用以下语法，其中 `<Class>` 是系统类的名称，`<Property>`是属性的名称。

```xml
$([Class]::Property)
```

例如，您可以使用以下代码将构建属性设置为当前日期和时间。

```xml
<Today>$([System.DateTime]::Now)</Today>
```

要调用静态方法，请使用以下语法，其中`<Class>`是系统类的名称，`<Method>`是方法的名称，`(<Parameters>)`是方法的参数列表：

```xml
$([Class]::Method(Parameters))
```

例如，要将构建属性设置为新的 GUID，您可以使用以下脚本：

```xml
<NewGuid>$([System.Guid]::NewGuid())</NewGuid>
```

在静态属性函数中，您可以使用这些系统类的任何静态方法或属性：

- [System.Byte](https://docs.microsoft.com/en-us/dotnet/api/system.byte)
- [System.Char](https://docs.microsoft.com/en-us/dotnet/api/system.char)
- [System.Convert](https://docs.microsoft.com/en-us/dotnet/api/system.convert)
- [System.DateTime](https://docs.microsoft.com/en-us/dotnet/api/system.datetime)
- [System.Decimal](https://docs.microsoft.com/en-us/dotnet/api/system.decimal)
- [System.Double](https://docs.microsoft.com/en-us/dotnet/api/system.double)
- [System.Enum](https://docs.microsoft.com/en-us/dotnet/api/system.enum)
- [System.Guid](https://docs.microsoft.com/en-us/dotnet/api/system.guid)
- [System.Int16](https://docs.microsoft.com/en-us/dotnet/api/system.int16)
- [System.Int32](https://docs.microsoft.com/en-us/dotnet/api/system.int32)
- [System.Int64](https://docs.microsoft.com/en-us/dotnet/api/system.int64)
- [System.IO.Path](https://docs.microsoft.com/en-us/dotnet/api/system.io.path)
- [System.Math](https://docs.microsoft.com/en-us/dotnet/api/system.math)
- [System.Runtime.InteropServices.OSPlatform](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.osplatform)
- [System.Runtime.InteropServices.RuntimeInformation](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.runtimeinformation)
- [System.UInt16](https://docs.microsoft.com/en-us/dotnet/api/system.uint16)
- [System.UInt32](https://docs.microsoft.com/en-us/dotnet/api/system.uint32)
- [System.UInt64](https://docs.microsoft.com/en-us/dotnet/api/system.uint64)
- [System.SByte](https://docs.microsoft.com/en-us/dotnet/api/system.sbyte)
- [System.Single](https://docs.microsoft.com/en-us/dotnet/api/system.single)
- [System.String](https://docs.microsoft.com/en-us/dotnet/api/system.string)
- [System.StringComparer](https://docs.microsoft.com/en-us/dotnet/api/system.stringcomparer)
- [System.TimeSpan](https://docs.microsoft.com/en-us/dotnet/api/system.timespan)
- [System.Text.RegularExpressions.Regex](https://docs.microsoft.com/en-us/dotnet/api/system.text.regularexpressions.regex)
- [System.UriBuilder](https://docs.microsoft.com/en-us/dotnet/api/system.uribuilder)
- [System.Version](https://docs.microsoft.com/en-us/dotnet/api/system.version)
- [Microsoft.Build.Utilities.ToolLocationHelper](https://docs.microsoft.com/en-us/dotnet/api/microsoft.build.utilities.toollocationhelper)

此外，您可以使用以下静态方法和属性：

- [System.Environment::CommandLine](https://docs.microsoft.com/en-us/dotnet/api/system.environment.commandline)
- [System.Environment::ExpandEnvironmentVariables](https://docs.microsoft.com/en-us/dotnet/api/system.environment.expandenvironmentvariables)
- [System.Environment::GetEnvironmentVariable](https://docs.microsoft.com/en-us/dotnet/api/system.environment.getenvironmentvariable)
- [System.Environment::GetEnvironmentVariables](https://docs.microsoft.com/en-us/dotnet/api/system.environment.getenvironmentvariables)
- [System.Environment::GetFolderPath](https://docs.microsoft.com/en-us/dotnet/api/system.environment.getfolderpath)
- [System.Environment::GetLogicalDrives](https://docs.microsoft.com/en-us/dotnet/api/system.environment.getlogicaldrives)
- [System.IO.Directory::GetDirectories](https://docs.microsoft.com/en-us/dotnet/api/system.io.directory.getdirectories)
- [System.IO.Directory::GetFiles](https://docs.microsoft.com/en-us/dotnet/api/system.io.directory.getfiles)
- [System.IO.Directory::GetLastAccessTime](https://docs.microsoft.com/en-us/dotnet/api/system.io.directory.getlastaccesstime)
- [System.IO.Directory::GetLastWriteTime](https://docs.microsoft.com/en-us/dotnet/api/system.io.directory.getlastwritetime)
- [System.IO.Directory::GetParent](https://docs.microsoft.com/en-us/dotnet/api/system.io.directory.getparent)
- [System.IO.File::Exists](https://docs.microsoft.com/en-us/dotnet/api/system.io.file.exists)
- [System.IO.File::GetCreationTime](https://docs.microsoft.com/en-us/dotnet/api/system.io.file.getcreationtime)
- [System.IO.File::GetAttributes](https://docs.microsoft.com/en-us/dotnet/api/system.io.file.getattributes)
- [System.IO.File::GetLastAccessTime](https://docs.microsoft.com/en-us/dotnet/api/system.io.file.getlastaccesstime)
- [System.IO.File::GetLastWriteTime](https://docs.microsoft.com/en-us/dotnet/api/system.io.file.getlastwritetime)
- [System.IO.File::ReadAllText](https://docs.microsoft.com/en-us/dotnet/api/system.io.file.readalltext)

#### 2.6.2.3 Calling instance methods on static properties

如果访问返回对象实例的静态属性，则可以调用该对象的实例方法。要调用实例方法，请使用以下语法，其中`<Class>`是系统类的名称，`<Property>`是属性的名称，`<Method>`是方法的名称，`(<Parameters>)`是该方法的参数列表：

```xml
$([Class]::Property.Method(Parameters))
```

类的名称必须使用命名空间完全限定。

例如，您可以使用以下代码将构建属性设置为今天的当前日期。

```xml
<Today>$([System.DateTime]::Now.ToString('yyyy.MM.dd'))</Today>
```

#### 2.6.2.4 MSBuild property functions

可以访问构建中的几个静态方法以提供算术、按位逻辑和转义字符支持。您可以使用以下语法访问这些方法，其中`<Method>`是方法的名称，` (<Parameters>)`是方法的参数列表。

```
$([MSBuild]::Method(Parameters))
```

例如，要将两个具有数值的属性相加，请使用以下代码。

```xml
$([MSBuild]::Add($(NumberOne), $(NumberTwo)))
```

下面是 MSBuild 属性函数的列表：

| Function signature                                           | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| double Add(double a, double b)                               | Add two doubles.                                             |
| long Add(long a, long b)                                     | Add two longs.                                               |
| double Subtract(double a, double b)                          | Subtract two doubles.                                        |
| long Subtract(long a, long b)                                | Subtract two longs.                                          |
| double Multiply(double a, double b)                          | Multiply two doubles.                                        |
| long Multiply(long a, long b)                                | Multiply two longs.                                          |
| double Divide(double a, double b)                            | Divide two doubles.                                          |
| long Divide(long a, long b)                                  | Divide two longs.                                            |
| double Modulo(double a, double b)                            | Modulo two doubles.                                          |
| long Modulo(long a, long b)                                  | Modulo two longs.                                            |
| string Escape(string unescaped)                              | Escape the string according to MSBuild escaping rules.       |
| string Unescape(string escaped)                              | Unescape the string according to MSBuild escaping rules.     |
| int BitwiseOr(int first, int second)                         | Perform a bitwise `OR` on the first and second (first \| second). |
| int BitwiseAnd(int first, int second)                        | Perform a bitwise `AND` on the first and second (first & second). |
| int BitwiseXor(int first, int second)                        | Perform a bitwise `XOR` on the first and second (first ^ second). |
| int BitwiseNot(int first)                                    | Perform a bitwise `NOT` (~first).                            |
| bool IsOsPlatform(string platformString)                     | Specify whether the current OS platform is `platformString`. `platformString` must be a member of [OSPlatform](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.osplatform). |
| bool IsOSUnixLike()                                          | True if current OS is a Unix system.                         |
| string NormalizePath(params string[] path)                   | Gets the canonicalized full path of the provided path and ensures it contains the correct directory separator characters for the current operating system. |
| string NormalizeDirectory(params string[] path)              | Gets the canonicalized full path of the provided directory and ensures it contains the correct directory separator characters for the current operating system while ensuring it has a trailing slash. |
| string EnsureTrailingSlash(string path)                      | If the given path doesn't have a trailing slash then add one. If the path is an empty string, does not modify it. |
| string GetPathOfFileAbove(string file, string startingDirectory) | Searches for and returns the full path to a file in the directory structure above the current build file's location, or based on `startingDirectory`, if specified. |
| GetDirectoryNameOfFileAbove(string startingDirectory, string fileName) | Locate and return the directory of a file in either the directory specified or a location in the directory structure above that directory. |
| string MakeRelative(string basePath, string path)            | Makes `path` relative to `basePath`. `basePath` must be an absolute directory. If `path` cannot be made relative, it is returned verbatim. Similar to `Uri.MakeRelativeUri`. |
| string ValueOrDefault(string conditionValue, string defaultValue) | Return the string in parameter 'defaultValue' only if parameter 'conditionValue' is empty, else, return the value conditionValue. |

###  2.6.2 Nested property functions

您可以组合属性函数以形成更复杂的函数，如下例所示。

```
$([MSBuild]::BitwiseAnd(32, $([System.IO.File]::GetAttributes(tempFile))))
```

此示例返回路径 tempFile 给出的文件的[FileAttributes](https://docs.microsoft.com/en-us/dotnet/api/system.io.fileattributes).Archive 位（32 或 0）的值。请注意，枚举数据值在某些上下文中不能按名称出现。在前面的示例中，必须改用数值 (32)。在其他情况下，根据所调用方法的预期，必须使用枚举数据值。在以下示例中，必须使用枚举值 [RegexOptions](https://docs.microsoft.com/en-us/dotnet/api/system.text.regularexpressions.regexoptions).ECMAScript，因为无法按此方法预期的方式转换数值。

```xml
<PropertyGroup>
  <GitVersionHeightWithOffset>$([System.Text.RegularExpressions.Regex]::Replace("$(PrereleaseVersion)", "^.*?(\d+)$", "$1", "System.Text.RegularExpressions.RegexOptions.ECMAScript"))</GitVersionHeightWithOffset>
</PropertyGroup>
```

元数据也可能出现在嵌套的属性函数中。有关详细信息，请参阅[Batching](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-batching?view=vs-2022)。

### 2.6.3 MSBuild DoesTaskHostExist

MSBuild 中的 DoesTaskHostExist 属性函数返回当前是否为指定的运行时和体系结构值安装了任务主机。

此属性函数具有以下语法：

```
$([MSBuild]::DoesTaskHostExist(string theRuntime, string theArchitecture))
```

### 2.6.4 MSBuild EnsureTrailingSlash

如果不存在斜杠，MSBuild 中的 EnsureTrailingSlash 属性函数会添加斜杠。

此属性函数具有以下语法：

```
$([MSBuild]::EnsureTrailingSlash('$(PathProperty)'))
```

### 2.6.5 MSBuild GetDirectoryNameOfFileAbove

MSBuild GetDirectoryNameOfFileAbove 属性函数向上搜索包含指定文件的目录，从（并包括）指定目录开始。如果找到，则返回包含该文件的最近目录的完整路径，否则返回空字符串。

此属性函数具有以下语法：

```
$([MSBuild]::GetDirectoryNameOfFileAbove(string startingDirectory, string fileName))
```

此示例显示如何仅在找到匹配项的情况下导入当前文件夹中或上方最近的 EnlistmentInfo.props 文件：

```xml
<Import Project="$([MSBuild]::GetDirectoryNameOfFileAbove($(MSBuildThisFileDirectory), EnlistmentInfo.props))\EnlistmentInfo.props" Condition=" '$([MSBuild]::GetDirectoryNameOfFileAbove($(MSBuildThisFileDirectory), EnlistmentInfo.props))' != '' " />
```

请注意，使用 GetPathOfFileAbove 函数可以更简洁地编写此示例：

```xml
<Import Project="$([MSBuild]::GetPathOfFileAbove(EnlistmentInfo.props))" Condition=" '$([MSBuild]::GetPathOfFileAbove(EnlistmentInfo.props))' != '' " />
```

### 2.6.6 MSBuild GetPathOfFileAbove

MSBuild GetPathOfFileAbove 属性函数向上搜索包含指定文件的目录，从（并包括）指定目录开始。如果找到，则返回最近匹配文件的完整路径，否则返回空字符串。

此属性函数具有以下语法：

```xml
$([MSBuild]::GetPathOfFileAbove(string file, [string startingDirectory]))
```

其中 file 是要搜索的文件的名称，startingDirectory 是开始搜索的可选目录。默认情况下，搜索将从当前文件自己的目录开始。

这个例子展示了如何在当前目录中或上面导入一个名为 dir.props 的文件，前提是找到匹配项：

```xml
<Import Project="$([MSBuild]::GetPathOfFileAbove(dir.props))" Condition=" '$([MSBuild]::GetPathOfFileAbove(dir.props))' != '' " />
```

这在功能上等同于

```xml
<Import Project="$([MSBuild]::GetDirectoryNameOfFileAbove($(MSBuildThisFileDirectory), dir.props))\dir.props" Condition=" '$([MSBuild]::GetDirectoryNameOfFileAbove($(MSBuildThisFileDirectory), dir.props))' != '' " />
```

但是，有时您需要在父目录中开始搜索，以避免匹配当前文件。此示例显示了 Directory.Build.props 文件如何在树的严格更高级别中导入最近的 Directory.Build.props 文件，而无需递归导入自身：

```xml
<Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../'))" />
```

这在功能上等同于

```xml
<Import Project="$([MSBuild]::GetDirectoryNameOfFileAbove('$(MSBuildThisFileDirectory)../', 'Directory.Build.props'))/Directory.Build.props" />
```

### 2.6.7 MSBuild GetRegistryValue

MSBuild GetRegistryValue 属性函数返回注册表项的值。该函数接受两个参数，键名和值名，并从注册表返回值。如果不指定值名称，则返回默认值。

以下示例显示了如何使用此函数：

```xml
$([MSBuild]::GetRegistryValue(`HKEY_CURRENT_USER\Software\Microsoft\VisualStudio\10.0\Debugger`, ``))                                  // default value
$([MSBuild]::GetRegistryValue(`HKEY_CURRENT_USER\Software\Microsoft\VisualStudio\10.0\Debugger`, `SymbolCacheDir`))
$([MSBuild]::GetRegistryValue(`HKEY_LOCAL_MACHINE\SOFTWARE\(SampleName)`, `(SampleValue)`))             // parens in name and value
```

> 在 .NET SDK 版本的 MSBuild（dotnet build）中，不支持此功能。

### 2.6.8 MSBuild GetRegistryValueFromView

MSBuild GetRegistryValueFromView 属性函数在给定注册表项、值和一个或多个有序注册表视图的情况下获取系统注册表数据。在每个注册表视图中按顺序搜索键和值，直到找到它们。

此属性函数的语法是：

```
[MSBuild]::GetRegistryValueFromView(string keyName, string valueName, object defaultValue, params object[] views)
```

Windows 64 位操作系统维护一个 HKEY_LOCAL_MACHINE\SOFTWARE\Wow6432Node 注册表项，该注册表项为 32 位应用程序提供 HKEY_LOCAL_MACHINE\SOFTWARE 注册表视图。

默认情况下，在 WOW64 上运行的 32 位应用程序访问 32 位注册表视图，而 64 位应用程序访问 64 位注册表视图。

可以使用以下注册表视图：

| Registry view           | Definition                                                   |
| :---------------------- | :----------------------------------------------------------- |
| RegistryView.Registry32 | The 32-bit application registry view.                        |
| RegistryView.Registry64 | The 64-bit application registry view.                        |
| RegistryView.Default    | The registry view that matches the process that the application is running on. |

下面是一个例子。

```
$([MSBuild]::GetRegistryValueFromView('HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Microsoft SDKs\Silverlight\v3.0\ReferenceAssemblies', 'SLRuntimeInstallPath', null, RegistryView.Registry64, RegistryView.Registry32))
```

获取 ReferenceAssemblies 键的 SLRuntimeInstallPath 数据，首先在 64 位注册表视图中查找，然后在 32 位注册表视图中查找。

> 在 .NET SDK 版本的 MSBuild（dotnet build）中，不支持此功能。

### 2.6.9 MSBuild MakeRelative

MSBuild MakeRelative 属性函数返回第二个路径相对于第一个路径的相对路径。每个路径可以是一个文件或文件夹。

此属性函数具有以下语法：

```
$([MSBuild]::MakeRelative($(FileOrFolderPath1), $(FileOrFolderPath2)))
```

以下代码是此语法的示例。

```xml
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

### 2.6.10 MSBuild ValueOrDefault

MSBuild ValueOrDefault 属性函数返回第一个参数，除非它为 null 或为空。如果第一个参数为 null 或为空，则函数返回第二个参数。

下面的例子展示了如何使用这个函数。

```xml
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

### 2.6.11 MSBuild TargetFramework and TargetPlatform functions

MSBuild 16.7 和更高版本定义了几个用于处理 [TargetFramework and TargetPlatform properties](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-target-framework-and-target-platform?view=vs-2022)。

| Function signature                                           | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| GetTargetFrameworkIdentifier(string targetFramework)         | Parse the TargetFrameworkIdentifier from the TargetFramework. |
| GetTargetFrameworkVersion(string targetFramework, int versionPartCount) | Parse the TargetFrameworkVersion from the TargetFramework.   |
| GetTargetPlatformIdentifier(string targetFramework)          | Parse the TargetPlatformIdentifier from the TargetFramework. |
| GetTargetPlatformVersion(string targetFramework, int versionPartCount) | Parse the TargetPlatformVersion from the TargetFramework.    |
| IsTargetFrameworkCompatible(string targetFrameworkTarget, string targetFrameworkCandidate) | Return 'True' if the candidate target framework is compatible with this target framework and false otherwise. |

GetTargetFrameworkVersion 和 GetTargetPlatformVersion 的 versionPartCount 参数的默认值为 2。

以下示例显示了如何使用这些函数。

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

Output:
Value1 = .NETCoreApp
Value2 = 5.0
Value3 = windows
Value4 = 7.0
Value5 = True
```

### 2.6.12 MSBuild version-comparison functions

MSBuild 16.5 和更高版本定义了几个用于比较表示版本的字符串的函数。

> 条件中的比较运算符可以比较可以解析为 System.Version 对象的字符串，但比较可能会产生意外结果。更喜欢属性函数。

| Function signature                             | Description                                                  |
| :--------------------------------------------- | :----------------------------------------------------------- |
| VersionEquals(string a, string b)              | Return `true` if versions `a` and `b` are equivalent according to the below rules. |
| VersionGreaterThan(string a, string b)         | Return `true` if version `a` is greater than `b` according to the below rules. |
| VersionGreaterThanOrEquals(string a, string b) | Return `true` if version `a` is greater than or equal to `b` according to the below rules. |
| VersionLessThan(string a, string b)            | Return `true` if version `a` is less than `b` according to the below rules. |
| VersionLessThanOrEquals(string a, string b)    | Return `true` if version `a` is less than or equal to `b` according to the below rules. |
| VersionNotEquals(string a, string b)           | Return `false` if versions `a` and `b` are equivalent according to the below rules. |

在这些方法中，版本的解析类似于[System.Version](https://docs.microsoft.com/en-us/dotnet/api/system.version)，但有以下例外：

- 前导 v 或 V 被忽略，这允许与`$(TargetFrameworkVersion) `进行比较。

- 从第一个“-”或“+”到版本字符串末尾的所有内容都将被忽略。这允许传入语义版本（semver），尽管顺序与 semver 不同。相反，预发布说明符和构建元数据没有任何排序权重。这可能很有用，例如，打开 >= x.y 的功能并在 x.y.z-pre 上启动它。

- 未指定的部分与零值部分相同。 （x == x.0 == x.0.0 == x.0.0.0）。

- 整数组件中不允许有空格。

- 仅主要版本有效（3 等于 3.0.0.0）

+ +不允许作为整数组件中的正号（它被视为 semver 元数据并被忽略）

> TargetFramework 属性的比较一般应该使用 IsTargetFrameworkCompatible，而不是提取和比较版本。这允许比较 TargetFrameworkIdentifier 和版本不同的 TargetFrameworks。

### 2.6.13 MSBuild condition functions

Exists 和 HasTrailingSlash 函数不是属性函数。它们可用于 Condition 属性。

> 请参阅 [MSBuild conditions](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-conditions?view=vs-2022)。

## 2.7. MSBuild items

## 2.8. How to: Select the files to build

## 2.9. How to: Exclude files from the build

## 2.10. How to: Display an item list separated with commas

## 2.11. Item definitions

## 2.12. Item functions

## 2.13. MSBuild targets

## 2.14. Target build order

## 2.15. Incremental builds

## 2.16. How to: Specify which target to build first

## 2.17. How to: Use the same target in multiple project files

## 2.18. How to: Build specific targets in solutions by using MSBuild.exe

## 2.19. How to: Build incrementally

## 2.20. How to: Clean a build

## 2.21. How to: Use MSBuild project SDKs

## 2.22. MSBuild tasks

## 2.23. Task writing

## 2.24. How to: Ignore errors in tasks

## 2.25. How to: Build a project that has resources

## 2.26. Tutorial: Create a custom task for code generation

## 2.27. Tutorial: Generate a REST API client

## 2.28. Tutorial: Test a custom task

## 2.29. MSBuild inline tasks

## 2.30. Walkthrough: Create an inline task

## 2.31. MSBuild inline tasks with RoslynCodeTaskFactory

## 2.32. Compare properties and items

## 2.33. MSBuild special characters

## 2.34. How to: Escape special characters in MSBuild

## 2.35. How to: Use reserved XML characters in project files











































# 3. Advanced Concepts

# 4. Logging

# 5. Walkthrough：Using MSBuild

# 6. Walkthrough：Creating an MSBuild project file from scratch

# 7. Use MSBuild programmatically

# 8. MSBuild glossary
