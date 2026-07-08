+++
title="WPF"
date=2025-05-21

[taxonomies]
categories = ["post"]
tags = ["C#","WPF"]
+++

### namespace

In WPF examples, I often see things like `x:Type`. What is this `x`?
At the top of the file, there is usually something like this:

```
xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
```

Similar to writing a `using` namespace in C# code, XAML declares XML namespace prefixes with `xmlns`. It is not exactly the same concept, though. `xmlns:x` is the prefix for XAML language features such as `x:Class`, `x:Key`, `x:Name`, and `x:Type`. WPF XAML is case-sensitive, so element and property names must match casing too.

<br>

### ContentControl

`ContentControl` is a base control.
`App.xaml.cs` is the program entry point and runs `MainWindow`.
`MainWindow` inherits from `Window`, and `Window` inherits from `ContentControl`.
In practice, we use `ContentControl` without always noticing it.

```cs
<ContentControl Content="{Binding CurrentViewModel}" />
```

```cs
<ContentControl Content="{StaticResource person}" />
```

This sets the view that displays `UIViewModel` to `<views:UIView>` and shows how to put it inside a `DataTemplate`.

```cs
<ContentControl Content="{Binding CurrentViewModel}">
    <ContentControl.Resources>
        <DataTemplate DataType="{x:Type viewModels:UIViewModel}">
        <views:UIView>
        </views:UIView>
        </DataTemplate>
    </ContentControl.Resources>
</ContentControl>
```

Binding examples:

```text
Binding="{Binding Name}"
Binding="{Binding ElementName=Name}"
```

<br>

### Display data

When studying WPF, I often need to display data as tables or lists. The controls that show up often in examples are:

[DataGrid](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.datagrid)
vs [GridView](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/controls/gridview-overview)
vs [ItemsControl](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.itemscontrol)
vs [ListBox](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.listbox)
vs [ListView](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.listview)

<br>

The inheritance chain is:

<br>

ItemsControl -> Selector -> ListBox -> ListView

`GridView` is a view mode placed inside `ListView.View`. Unlike `DataGrid`, it is not a standalone item control. It inherits from `ViewBase` and displays `ListView` data in columns.

For more detail, see [the WPF control library](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/controls/control-library).

<br>

### {Binding PropertyName} and {Binding Path=PropertyName}

https://stackoverflow.com/questions/4306657/difference-between-binding-propertyname-and-binding-path-propertyname

<br>

### SelectedValue vs SelectedItem

<br>

### Textbox getting user input

```xml
<TextBox Width="200" Height="25" FontSize="14"
                 Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}" />
```

<br>

### Ref

- XAML overview: <https://learn.microsoft.com/en-us/dotnet/desktop/wpf/xaml/>
- ContentControl: <https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.contentcontrol>
- ItemsControl: <https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.itemscontrol>
- ListBox: <https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.listbox>
- ListView: <https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.listview>
- GridView: <https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.gridview>
