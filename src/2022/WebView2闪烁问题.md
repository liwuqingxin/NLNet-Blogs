# WebView2 Flickering Problems

I have some WebView2 in TabControls in WPF. It gets flickering when I open new tab, close tab or swtich tabs. The detailed questions are:

1. The WebView2 flashes a blank white background when I create a new TabItem with a WebView2 that has initialized with a source within it.
2. It behaves the same when I close a TabItem. The TabControl auto switch a new TabItem as selected item when I close one.
3. The same as I switch the TabItems.

## Why write this post in English?

I have searched this problem in google and I found a [post : WebView2 Flashing when changing TabControl Tabs](https://weblog.west-wind.com/posts/2021/Oct/04/WebView2-Tab-Flashes-when-changing-TabControl-Tabs) write by **[Rick Strahl](https://github.com/RickStrahl)** that analyzed this problem and provided a solution to solve this problem.

I also found the issue of [Flash when using Multiple WebViews in Tab Controls · Issue #1412 · MicrosoftEdge/WebView2Feedback (github.com)](https://github.com/MicrosoftEdge/WebView2Feedback/issues/1412).

> NOW HERE I have **something else** to post for this problem, about some reasons for flickering and new solution for it. I would like to discuss it with everybody.

## What's the problem actrually?

In the gif #1 below we can see the flash when I change the TabItem. **It sucks**! What is it when flash happend?

The first flash that happens when I switch from page 1 to page 2 is white. I guess, the same as Rick do,  it is the default background of WebView2. I changed the background of WebView2 for page 3 to **RED** and it prove that.

##### Gif #1:

![Original Behavior](https://cdn.jsdelivr.net/gh/liwuqingxin/nlnet-blogs@main/src/2020/imgs/Original Behavior.gif)

In the picture #2 we can see the details. There are three state between 'original page' and 'target page'. They are:

1. **Flashing #1**. It is blanck, which means there is nothing in the TabItem. it shows the background color of the window.
2. **Flashing #2**. It is red. That is the background of WebView2 I set. Remember? In this state, WebView2 has been shown and web page in it has not been rendered.
3. **Target Page Loading**. Why does I said this frame is 'Target Page Loading'? If your eyes are like a microscope, then you can see that there is a scroll bar inside the page.

##### Pcture #2:

![](https://cdn.jsdelivr.net/gh/liwuqingxin/nlnet-blogs@main/src/2020/imgs/Origial Behavior 2.png)

## Solution for the three state

### State 1.

I'll talk about it later because it is the **most complex**.

### State 2.

Try to hide the WebView2 background color so that it's not conspicuous.

> **Conclusion #1**
>
> We can **change the default background** of WebView2 to **transparent** to avoid kind of flash. We can see the background of window or other host. By this way, we can combine state #1 and state #2 into one state.

### State 3.

We can not ask web developers to keep pages fast enough to make loading disapear. Instead we can provide a loading when page is navigating. To do this, we can show loading until `WebView2.NavigationCompleted` fired. I will provide gif later to reflect the effect.

> **Conclusion #2**
>
> We can use loading to beautify state #3.

### Back to State 1.

How does it happend? We know that TabControl auto unloads content of TabItem when the TabItem is not the selected item again. When We swtich the TabItems, TabControl unload previous content and then load new one. Between the load and unload actions, WebView2 disappers and shows. The second WebView2 has not been rendered until the first one disappered. So we can see the background of the host.

I must mention that **Rick** didn't note this state. What he mainly mentioned is to solve the problem of the state #2. 

**I don't think that if we can hide the first WebView2 before the swtiching the problem can be resolved**. This is more likely to lead to the occurrence of state #1.

Then what can we do about this? 

> **Conclusion #3**
>
> We should keep the previous WebView2 visible until the new one rendered to skip the state #1.

I come up with a idea to do this. I don't use binding like Rick does. I override the metadata of SelectedItemProperty of TabControl and do my task in the coerce fuction. In this function, SelectedItem has not been changed until it ends.

And then we can remove the ContentPresenter in the template of TabControl to block the mechanism that TabControl controls the selected item, in which TabControl unloads old content and loads new content. We can realize the content management by ourself.

Now I show the code here.

``` csharp
/// <summary>
/// This is an extended TabControl that can avoid unexpected unloading when switches TabItems.
/// This is especially effective for the flickering problem caused by switching between multiple WebView2s that exist in a TabControl.
/// </summary>
public class TabControlEx : TabControl
{
    private Grid _panel;

    static TabControlEx()
    {
        DefaultStyleKeyProperty.OverrideMetadata(typeof(TabControlEx), new FrameworkPropertyMetadata(typeof(TabControlEx)));
        SelectedItemProperty.OverrideMetadata(typeof(TabControlEx), new FrameworkPropertyMetadata(default, FrameworkPropertyMetadataOptions.None, PropertyChangedCallback, CoerceValueCallback));
    }

    private static void PropertyChangedCallback(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        
    }

    private static object CoerceValueCallback(DependencyObject d, object baseValue)
    {
        return ((TabControlEx)d).OnCoerceSelectedItem(baseValue);
    }

    private object OnCoerceSelectedItem(object baseValue)
    {
        if (_panel != null)
        {
            var tabItem  = baseValue as TabItem;
            BuildContent(tabItem);
        }

        return baseValue;
    }

    private void BuildContent(TabItem item)
    {
        switch (item?.Content)
        {
            case null:
                return;
            case FrameworkElement fe when fe.Tag == null:
            {
                // if content of TabItem has not been loaded ever, just load it after clear remained items
                fe.Tag = new object();
                _panel.Children.Clear();
                var host = new ContentControl
                {
                    Content = item.Content
                };

                _panel.Children.Add(host);
                break;
            }
            default:
            {
                if (ReferenceEquals(SelectedItem, item))
                {
                    return;
                }

                // host new content
                var host = new ContentControl
                {
                    Content = item.Content
                };
                _panel.Children.Add(host);

                // remove first one if children count is more than 3 and remain second one as background to prevent flashing
                if (_panel.Children.Count > 2)
                {
                    _panel.Children.RemoveAt(0);
                }

                // remove second one after 200ms to keep memory clean. You can change the duration for your situation
                // even if there are some pages whose loading time is more than the duration you provide, we can ignore those or make duration longner.
                Task.Delay(200).ContinueWith(t =>
                {
                    Dispatcher.BeginInvoke(new Action(() =>
                    {
                        if (_panel.Children.Count > 1)
                        {
                            _panel.Children.RemoveAt(0);
                        }
                    }), DispatcherPriority.Background);
                });
                break;
            }
        }
    }

    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();

        this._panel = Template.FindName("PART_SelectedContentPanel", this) as Grid;
    }
}
```

The Template of TabControl is:

``` xaml
<ControlTemplate TargetType="baseControls:TabControlEx">
    <Grid x:Name="templateRoot"
          Background="{TemplateBinding Background}"
          ClipToBounds="True"
          KeyboardNavigation.TabNavigation="Local"
          SnapsToDevicePixels="True">
        <Grid.ColumnDefinitions>
            <ColumnDefinition x:Name="Col0" Width="*" />
            <ColumnDefinition x:Name="Col1" Width="Auto" />
        </Grid.ColumnDefinitions>
        <Grid.RowDefinitions>
            <RowDefinition x:Name="Row0" Height="Auto" />
            <RowDefinition x:Name="Row1" Height="*" />
        </Grid.RowDefinitions>
        <TabPanel x:Name="headerPanel"
                  Grid.Row="0"
                  Grid.Column="0"
                  Margin="1"
                  Panel.ZIndex="1"
                  Background="Transparent"
                  IsItemsHost="True"
                  KeyboardNavigation.TabIndex="1" />
        <Border x:Name="contentPanel"
                Grid.Row="1"
                Grid.Column="0"
                HorizontalAlignment="Stretch"
                VerticalAlignment="Stretch"
                Background="{TemplateBinding Background}"
                BorderBrush="{TemplateBinding BorderBrush}"
                BorderThickness="{TemplateBinding BorderThickness}"
                KeyboardNavigation.DirectionalNavigation="Contained"
                KeyboardNavigation.TabIndex="2"
                KeyboardNavigation.TabNavigation="Local">
            
            <!-- Here I remove the ContentPresenter and use Grid as Contents host. -->
            
            <Grid x:Name="PART_SelectedContentPanel">
                <ContentControl x:Name="PART_SelectedContentHost1"
                                Margin="0"
                                SnapsToDevicePixels="{TemplateBinding SnapsToDevicePixels}" />
                <ContentControl x:Name="PART_SelectedContentHost2"
                                Margin="0"
                                SnapsToDevicePixels="{TemplateBinding SnapsToDevicePixels}" />
            </Grid>
        </Border>
    </Grid>
</ControlTemplate>
```

## Extended about closing and opening

Now maybe you will note that the state #3 will happen when we open a TabItem and  the state #1 will also happend because the TabItem to close will disapper before the next TabItem rendering when we close a TabItem.

### Open

Like the **Conclusion #1, #2**, We can use transparent background and use a loading to hide stage of state #3.

### Close

Oh! My code solved this problems automatically... it really excites me. :)

There are three change events fired when we close a TabItem. In first event it is current item and we can skip it by `if (ReferenceEquals(SelectedItem, item))`. In the second one it is null and we also ignore it. In the third one it is the new Item to select. In this event, we dit the same task as state #1.

## Enjoy :)

![](https://cdn.jsdelivr.net/gh/liwuqingxin/nlnet-blogs@main/src/2020/imgs/New Behavior2.gif)
