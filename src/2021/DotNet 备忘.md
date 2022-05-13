<br/>

<details>
<summary>1. [WPF] 代码构建 ItemsPanel</summary>

```csharp
var itemsPanelTemplate = new ItemsPanelTemplate();
var factory = new FrameworkElementFactory(typeof(AnimatingStackPanel));
factory.SetValue(StackPanel.OrientationProperty, Orientation.Vertical);
itemsPanelTemplate.VisualTree = factory;
```

</details>

<details>
<summary>2. [WPF] 统一Xaml命名空间</summary>

```csharp
// 声明命名空间，注意如果命名空间不存在，会出现编译错误，且比较难以定位
[assembly: XmlnsDefinition("http://ssm.app.com", "ssm.ui.Assists")]

// 声明命名控件缩写前缀
[assembly: XmlnsPrefix("http://ssm.app.com", "ssm")]
```

</details>

<details>
<summary>2.1. [WPF] .net core 工程添加 XmlnsDefinition 特性</summary>

**参考** [WPF attributes coming to csproj or remain in AssemblyInfo? · Issue #294 · dotnet/wpf (github.com)](https://github.com/dotnet/wpf/issues/294#issuecomment-457943054)

方案：在任意cs文件中添加如下代码即可。建议添加单独的Markup.cs文件用于存放这些代码。

```csharp
[assembly: XmlnsDefinition("https://github.com/liwuqingxin", "NLNet.Kit.MVVM.NSConverters")]
[assembly: XmlnsPrefix("https://github.com/liwuqingxin", "nlnet")]
```

</details>

<details>
<summary>3. [WPF] 使用 WinForm，设置样式可用</summary>

**参考** [为什么Winform控件在WPF中的样式和真正WinForm窗体内的不同？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/48925705?from=profile_question_card)

```csharp
System.Windows.Forms.Application.EnableVisualStyles();
System.Windows.Forms.Application.SetCompatibleTextRenderingDefault(false);
```

</details>

<details>
<summary>4. [WPF] 设置动画帧数</summary>

**参考** [WPF:关于Animation的CPU消耗 - 木木子 - 博客园 (cnblogs.com)](https://www.cnblogs.com/lnetmor/archive/2012/06/13/2548448.html)

```csharp
Timeline.DesiredFrameRateProperty.OverrideMetadata(typeof(Timeline), new FrameworkPropertyMetadata
{
    DefaultValue = framePerSecond
});
```

</details>

<details>
<summary>5. [WPF] TextBox控件对于鼠标单击获取焦点后的全选</summary>

**参考** [WPF中TextBox控件对于鼠标单击获取焦点后的全选 - 罗天大兵 - 博客园 (cnblogs.com)](https://www.cnblogs.com/shuilan/archive/2012/06/27/2566258.html)

```csharp
/// <summary>
/// 键盘焦点处于 TextBox 时，选中所有文本。
/// </summary>
private static void TbxSelectAllOnGotKeyboardFocus(object sender, KeyboardFocusChangedEventArgs e)
{
    if (sender.CheckIfIsNot<TextBox>(out var tbx)) return;

    tbx.SelectAll();
    tbx.PreviewMouseDown -= TbxSelectAll_OnPreviewMouseDown;
}

/// <summary>
/// 失去键盘焦点时，添加鼠标按下前事件。
/// </summary>
private static void TbxSelectAllOnLostKeyboardFocus(object sender, KeyboardFocusChangedEventArgs e)
{
    if (e.Source.CheckIfIsNot<TextBox>(out var tbx)) return;

    tbx.PreviewMouseDown += TbxSelectAll_OnPreviewMouseDown;
}

/// <summary>
/// 鼠标按下前事件。
/// </summary>
private static void TbxSelectAllOnPreviewMouseDown(object sender, MouseButtonEventArgs e)
{
    if (e.Source.CheckIfIsNot<TextBox>(out var tbx)) return;

    tbx.Focus();

    // BUG 这里不设置 Handled，选中后会被取消。而设置 Handled，会导致内部点击逻辑无法完成，比如内部按钮的点击，第一次无法被触发。
    e.Handled = true;
}
```

</details>

<details>
<summary>6. [WPF] DataGrid 判断行占位符</summary>

```csharp
if (e.Row.Item.Equals(CollectionView.NewItemPlaceholder))
{
    return;
}
```

</details>

<details>
<summary>7. [WPF] RichTextBox 自适应尺寸/取消自动换行</summary>

**参考**

[[WPF]根据内容自动设置大小的RichTextBox - 周银辉 - 博客园 (cnblogs.com)](https://www.cnblogs.com/zhouyinhui/archive/2010/06/22/1762633.html)

[C#/WPF: Disable Text-Wrap of RichTextBox - Stack Overflow](https://stackoverflow.com/questions/1368047/c-wpf-disable-text-wrap-of-richtextbox)

```csharp
private double AdjustSizeByContent(FlowDocument document, double margin)
{
    var formattedText = GetFormattedText(document);
    document.PageWidth = Math.Min(MaxWidth, Math.Max(MinWidth, formattedText.WidthIncludingTrailingWhitespace + margin));
    return document.PageWidth;
}

private static FormattedText GetFormattedText(FlowDocument doc)
{
    var output = new FormattedText(
        GetText(doc),
        CultureInfo.CurrentCulture,
        doc.FlowDirection,
        new Typeface(doc.FontFamily, doc.FontStyle, doc.FontWeight, doc.FontStretch),
        doc.FontSize,
        doc.Foreground);

    var offset = 0;

    foreach (var textElement in GetRunsAndParagraphs(doc))
    {
        if (textElement is Run run)
        {
            var count = run.Text.Length;

            output.SetFontFamily(run.FontFamily, offset, count);
            output.SetFontSize(run.FontSize, offset, count);
            output.SetFontStretch(run.FontStretch, offset, count);
            output.SetFontStyle(run.FontStyle, offset, count);
            output.SetFontWeight(run.FontWeight, offset, count);
            output.SetForegroundBrush(run.Foreground, offset, count);
            output.SetTextDecorations(run.TextDecorations, offset, count);

            offset += count;
        }
        else
        {
            offset += Environment.NewLine.Length;
        }
    }

    return output;
}

private static IEnumerable<TextElement> GetRunsAndParagraphs(FlowDocument doc)
{
    for (var position = doc.ContentStart; position != null && position.CompareTo(doc.ContentEnd) <= 0; position = position.GetNextContextPosition(LogicalDirection.Forward))
    {
        if (position.GetPointerContext(LogicalDirection.Forward) != TextPointerContext.ElementEnd) continue;

        switch (position.Parent)
        {
            case Run run:
                yield return run;
                break;
            case Paragraph para:
                yield return para;
                break;
            case LineBreak lineBreak:
                yield return lineBreak;
                break;
        }
    }
}
```

</details>

<details>
<summary>8. [WPF] Xaml动画情景：计时器剩余时间的状态，最后10s高亮</summary>

```xml
<!-- 计时器剩余时间的状态，最后10s高亮 -->
<DataTrigger Binding="{Binding Path=Timer.State}" Value="Notice">
    <Setter TargetName="TbkTimer" Property="FontSize" Value="{DynamicResource Title4}"/>
</DataTrigger>
<DataTrigger Binding="{Binding Path=Timer.State}" Value="Attention">
    <Setter TargetName="TbkTimer" Property="FontStyle" Value="Italic"/>
    <DataTrigger.EnterActions>
        <BeginStoryboard>
            <Storyboard>
                <ColorAnimation Duration="0:0:0.5" To="#FFFF6E41" 
                                Storyboard.TargetName="TbkTimer" 
                                Storyboard.TargetProperty="(Control.Foreground).(SolidColorBrush.Color)"/>
                <ColorAnimation Duration="0:0:0.5" To="#FFFF6E41"
                                Storyboard.TargetName="TbkTimerText" 
                                Storyboard.TargetProperty="(Control.Foreground).(SolidColorBrush.Color)"/>
                <DoubleAnimation Duration="0:0:0.2" To="72"
                                 Storyboard.TargetName="TbkTimer" 
                                 Storyboard.TargetProperty="FontSize">
                    <DoubleAnimation.EasingFunction>
                        <ExponentialEase EasingMode="EaseInOut" Exponent="0.75"/>
                    </DoubleAnimation.EasingFunction>
                </DoubleAnimation>
                <ThicknessAnimation Duration="0:0:0.2" To="15,0"
                                    Storyboard.TargetName="TbkTimer" 
                                    Storyboard.TargetProperty="Margin">
                    <ThicknessAnimation.EasingFunction>
                        <ExponentialEase EasingMode="EaseInOut" Exponent="0.75"/>
                    </ThicknessAnimation.EasingFunction>
                </ThicknessAnimation>
            </Storyboard>
        </BeginStoryboard>
    </DataTrigger.EnterActions>
    <DataTrigger.ExitActions>
        <BeginStoryboard>
            <Storyboard>
                <ColorAnimation Duration="0:0:0.5"
                                Storyboard.TargetName="TbkTimer" 
                                Storyboard.TargetProperty="(Control.Foreground).(SolidColorBrush.Color)"/>
                <ColorAnimation Duration="0:0:0.5"
                                Storyboard.TargetName="TbkTimerText" 
                                Storyboard.TargetProperty="(Control.Foreground).(SolidColorBrush.Color)"/>
                <DoubleAnimation Duration="0:0:0.2"
                                 Storyboard.TargetName="TbkTimer" 
                                 Storyboard.TargetProperty="FontSize">
                    <DoubleAnimation.EasingFunction>
                        <ExponentialEase EasingMode="EaseInOut" Exponent="0.75"/>
                    </DoubleAnimation.EasingFunction>
                </DoubleAnimation>
                <ThicknessAnimation Duration="0:0:0.2"
                                    Storyboard.TargetName="TbkTimer" 
                                    Storyboard.TargetProperty="Margin">
                    <ThicknessAnimation.EasingFunction>
                        <ExponentialEase EasingMode="EaseInOut" Exponent="0.75"/>
                    </ThicknessAnimation.EasingFunction>
                </ThicknessAnimation>
            </Storyboard>
        </BeginStoryboard>
    </DataTrigger.ExitActions>
</DataTrigger>
```

</details>

<details>
<summary>9. [WPF] 设计模式资源</summary>

**参考** [Trick To Use A ResourceDictionary Only When In Design Mode - TechNet Articles - United States (English) - TechNet Wiki (microsoft.com)](https://social.technet.microsoft.com/wiki/contents/articles/23287.trick-to-use-a-resourcedictionary-only-when-in-design-mode.aspx)

注意，

1. **不要覆盖Source属性**，这会导致设计器无法正常加载；
2. **不要引用资源工程**，这会导致设计器无法正常加载，且报错“未找到路径.....的一部分”。
3. iconfont的引用，如果字体文件不是在同工程，则设计模式下显示为▯。以**链接的形式**添加该字体文件到使用的工程即可解决，如果仅设计模式需要，将文件属性改为**DesignData**。

```csharp
public class DesignResourceDictionary : ResourceDictionary
{
    /// <summary>
    /// Gets or sets the design time source.
    /// </summary>
    /// <value>
    /// The design time source.
    /// </value>
    public Uri DesignSource
    {
        get => Source;
        set
        {
            //if ((bool) DesignerProperties.IsInDesignModeProperty.GetMetadata(typeof(DependencyObject)).DefaultValue)
            //{
            //    Source = value;
            //}
            if (VisualUtil.IsInDesignMode2)
            {
                Source = value;
            }
        }
    }
}
```

</details>

<details>
<summary>10. [C#] 调整（扩展）屏幕亮度</summary>

参考：[emoacht/Monitorian: A Windows desktop tool to adjust the brightness of multiple monitors with ease (github.com)](https://github.com/emoacht/Monitorian)

```csharp
using System;
using System.Collections.Generic;
using System.Runtime.InteropServices;
using System.Text.RegularExpressions;
using System.Windows;
// ReSharper disable FieldCanBeMadeReadOnly.Global
// ReSharper disable MemberCanBePrivate.Global
// ReSharper disable ClassWithVirtualMembersNeverInherited.Global
// ReSharper disable InconsistentNaming
// ReSharper disable ClassNeverInstantiated.Global

// 参考：https://github.com/emoacht/Monitorian

namespace ScreenManager
{
    [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Auto)]
    public struct PHYSICAL_MONITOR
    {
        public IntPtr hPhysicalMonitor;

        [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 128)]
        public string szPhysicalMonitorDescription;
    }

    [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Auto)]
    public struct MONITORINFOEX
    {
        public uint cbSize;
        public RECT rcMonitor;
        public RECT rcWork;
        public uint dwFlags;

        [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 32)]
        public string szDevice;
    }

    [StructLayout(LayoutKind.Sequential)]
    public struct RECT
    {
        public int left;
        public int top;
        public int right;
        public int bottom;

        public static implicit operator Rect(RECT rect)
        {
            if ((rect.right < rect.left) || (rect.bottom < rect.top))
                return Rect.Empty;

            return new Rect(
                rect.left,
                rect.top,
                rect.right - rect.left,
                rect.bottom - rect.top);
        }

        public static implicit operator RECT(Rect rect)
        {
            return new RECT
            {
                left = (int)rect.Left,
                top = (int)rect.Top,
                right = (int)rect.Right,
                bottom = (int)rect.Bottom
            };
        }
    }

    public class SafePhysicalMonitorHandle : SafeHandle
    {
        public SafePhysicalMonitorHandle(IntPtr handle) : base(IntPtr.Zero, true)
        {
            this.handle = handle; // IntPtr.Zero may be a valid handle.
        }

        public override bool IsInvalid => false; // The validity cannot be checked by the handle.

        protected override bool ReleaseHandle()
        {
            return ScreenHelper.DestroyPhysicalMonitor(handle);
        }
    }

    public class HandleItem
    {
        public byte   DisplayIndex;
        public RECT   MonitorRect;
        public IntPtr MonitorHandle;
        public HandleItem(byte displayIndex, RECT monitorRect, IntPtr monitorHandle)
        {
            DisplayIndex  = displayIndex;
            MonitorRect   = monitorRect;
            MonitorHandle = monitorHandle;
        }
    }

    public class ScreenHelper : IDisposable
    {
        #region Win32

        [DllImport("User32.dll")]
        [return: MarshalAs(UnmanagedType.Bool)]
        private static extern bool EnumDisplayMonitors(
            IntPtr hdc,
            IntPtr lprcClip,
            MonitorEnumProc lpfnEnum,
            IntPtr dwData);

        [return: MarshalAs(UnmanagedType.Bool)]
        private delegate bool MonitorEnumProc(
            IntPtr hMonitor,
            IntPtr hdcMonitor,
            IntPtr lprcMonitor,
            IntPtr dwData);

        [DllImport("User32.dll", EntryPoint = "GetMonitorInfoW")]
        [return: MarshalAs(UnmanagedType.Bool)]
        private static extern bool GetMonitorInfo(
            IntPtr hMonitor,
            ref MONITORINFOEX lpmi);

        [DllImport("user32.dll", EntryPoint = "MonitorFromWindow")]
        public static extern IntPtr MonitorFromWindow([In] IntPtr hwnd, uint dwFlags);

        [DllImport("dxva2.dll", EntryPoint = "DestroyPhysicalMonitors")]
        [return: MarshalAs(UnmanagedType.Bool)]
        public static extern bool DestroyPhysicalMonitors(uint dwPhysicalMonitorArraySize, ref PHYSICAL_MONITOR[] pPhysicalMonitorArray);

        [DllImport("dxva2.dll", EntryPoint = "GetNumberOfPhysicalMonitorsFromHMONITOR")]
        [return: MarshalAs(UnmanagedType.Bool)]
        public static extern bool GetNumberOfPhysicalMonitorsFromHMONITOR(IntPtr hMonitor, ref uint pdwNumberOfPhysicalMonitors);

        [DllImport("dxva2.dll", EntryPoint = "GetPhysicalMonitorsFromHMONITOR")]
        [return: MarshalAs(UnmanagedType.Bool)]
        public static extern bool GetPhysicalMonitorsFromHMONITOR(IntPtr hMonitor, uint dwPhysicalMonitorArraySize, [Out] PHYSICAL_MONITOR[] pPhysicalMonitorArray);

        [DllImport("dxva2.dll", EntryPoint = "GetMonitorBrightness")]
        [return: MarshalAs(UnmanagedType.Bool)]
        public static extern bool GetMonitorBrightness(IntPtr handle, ref uint minimumBrightness, ref uint currentBrightness, ref uint maxBrightness);

        [DllImport("dxva2.dll", EntryPoint = "SetMonitorBrightness")]
        [return: MarshalAs(UnmanagedType.Bool)]
        public static extern bool SetMonitorBrightness(IntPtr handle, uint newBrightness);

        [DllImport("Dxva2.dll", SetLastError = true)]
        [return: MarshalAs(UnmanagedType.Bool)]
        private static extern bool SetMonitorBrightness(SafePhysicalMonitorHandle hMonitor, uint dwNewBrightness);

        [DllImport("Dxva2.dll", SetLastError = true)]
        [return: MarshalAs(UnmanagedType.Bool)]
        public static extern bool DestroyPhysicalMonitor(IntPtr hMonitor);

        #endregion



        private PHYSICAL_MONITOR[] _physicalMonitorArray;
        private readonly uint _physicalMonitorsCount = 0;
        private readonly IntPtr _firstMonitorHandle;
        private readonly uint _minValue = 0;
        private readonly uint _maxValue = 0;
        private uint _currentValue = 0;



        public ScreenHelper(/*IntPtr windowHandle*/)
        {
            var handleItems = GetMonitorHandles();
            //var dwFlags = 0u;
            //var ptr = MonitorFromWindow(windowHandle, dwFlags);

            foreach (var handleItem in handleItems)
            {
                if (!GetNumberOfPhysicalMonitorsFromHMONITOR(handleItem.MonitorHandle, ref _physicalMonitorsCount))
                {
                    Console.WriteLine("Cannot get monitor count!");
                    continue;
                    throw new Exception("Cannot get monitor count!");
                }
                _physicalMonitorArray = new PHYSICAL_MONITOR[_physicalMonitorsCount];

                if (!GetPhysicalMonitorsFromHMONITOR(handleItem.MonitorHandle, _physicalMonitorsCount, _physicalMonitorArray))
                {
                    Console.WriteLine("Cannot get physical monitor handle!");
                    continue;
                    throw new Exception("Cannot get physical monitor handle!");
                }
                _firstMonitorHandle = _physicalMonitorArray[0].hPhysicalMonitor;

                if (!GetMonitorBrightness(_firstMonitorHandle, ref _minValue, ref _currentValue, ref _maxValue))
                {
                    Console.WriteLine("Cannot get monitor brightness!");
                    continue;
                    throw new Exception("Cannot get monitor brightness!");
                }
            }
        }

        private static bool TryGetDisplayIndex(string deviceName, out byte index)
        {
            // The typical format of device name is as follows:
            // EnumDisplayDevices (display), GetMonitorInfo : \\.\DISPLAY[index starting at 1]
            // EnumDisplayDevices (monitor)                 : \\.\DISPLAY[index starting at 1]\Monitor[index starting at 0]

            var match = Regex.Match(deviceName, @"DISPLAY(?<index>\d{1,2})\s*$");
            if (match.Success)
            {
                index = byte.Parse(match.Groups["index"].Value);
                return true;
            }
            index = 0;
            return false;
        }

        public static HandleItem[] GetMonitorHandles()
        {
            var handleItems = new List<HandleItem>();

            if (EnumDisplayMonitors(IntPtr.Zero, IntPtr.Zero, Proc, IntPtr.Zero))
            {
                return handleItems.ToArray();
            }
            return Array.Empty<HandleItem>();

            bool Proc(IntPtr monitorHandle, IntPtr hdcMonitor, IntPtr lprcMonitor, IntPtr dwData)
            {
                var monitorInfo = new MONITORINFOEX
                {
                    cbSize = (uint)Marshal.SizeOf<MONITORINFOEX>()
                };

                if (GetMonitorInfo(monitorHandle, ref monitorInfo))
                {
                    if (TryGetDisplayIndex(monitorInfo.szDevice, out byte displayIndex))
                    {
                        handleItems.Add(new HandleItem(
                            displayIndex: displayIndex,
                            monitorRect: monitorInfo.rcMonitor,
                            monitorHandle: monitorHandle));
                    }
                }
                return true;
            }
        }

        public void SetBrightness(int newValue) // 0 ~ 100
        {
            newValue = Math.Min(newValue, Math.Max(0, newValue));
            _currentValue = (_maxValue - _minValue) * (uint)newValue / 100u + _minValue;
            SetMonitorBrightness(_firstMonitorHandle, _currentValue);
        }

        /// <summary>
        /// Sets raw brightness not represented in percentage.
        /// </summary>
        /// <param name="physicalMonitorHandle">Physical monitor handle</param>
        /// <param name="brightness">Raw brightness (not always 0 to 100)</param>
        /// <param name="isHighLevelBrightnessSupported">Whether high level function is supported</param>
        /// <returns>Result</returns>
        public void SetBrightness(SafePhysicalMonitorHandle physicalMonitorHandle, uint brightness, bool isHighLevelBrightnessSupported = true)
        {
            SetMonitorBrightness(physicalMonitorHandle, brightness);
        }

        public void Dispose()
        {
            Dispose(true);
            GC.SuppressFinalize(this);
        }

        protected virtual void Dispose(bool disposing)
        {
            if (disposing)
            {
                if (_physicalMonitorsCount > 0)
                {
                    DestroyPhysicalMonitors(_physicalMonitorsCount, ref _physicalMonitorArray);
                }
            }
        }
    }
}
```

</details>