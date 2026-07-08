+++
title="wpf"
date=2025-05-13

[taxonomies]
categories = ["post"]
tags = ["C#","WPF"]
+++

Sometimes two windows appear when running a WPF application in Visual Studio.

App.xaml

```xml hl_lines=5
<Application x:Class="Trading.WPFClient.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:local="clr-namespace:Trading.WPFClient"
             StartupUri="MainWindow.xaml">
    <Application.Resources>

    </Application.Resources>
</Application>
```

App.xaml.cs

```cs hl_lines=5 8
 public partial class App : Application
 {
     protected override void OnStartup(StartupEventArgs e)
     {
         MainWindow = new MainWindow()
         {
             DataContext = new MainViewModel()
         };
         MainWindow.Show();
         base.OnStartup(e);
     }
 }
```

`StartupUri="MainWindow.xaml"` automatically shows that UI when the application starts. But `OnStartup` also creates `new MainWindow()` and calls `Show()`, so two windows are created.

Remove one of them. Usually, choose either `StartupUri`, or remove `StartupUri` and create the window directly in `OnStartup`. Do not use both.

[Problem With WPF C# App Spawning Two Main Windows](https://stackoverflow.com/questions/2923431/problem-with-wpf-c-sharp-app-spawning-two-main-windows)

### Ref

- Application.StartupUri: <https://learn.microsoft.com/en-us/dotnet/api/system.windows.application.startupuri>
- Application.MainWindow: <https://learn.microsoft.com/en-us/dotnet/api/system.windows.application.mainwindow>
- Application.OnStartup: <https://learn.microsoft.com/en-us/dotnet/api/system.windows.application.onstartup>
