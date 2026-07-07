+++
title="WPF 창이 두 개 뜨는 문제"
date=2025-05-13

[taxonomies]
categories = ["post"]
tags = ["C#","WPF"]
+++

Visual Studio에서 WPF 프로그램을 실행할 때 window가 두 개 뜨는 경우가 있다.
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

`StartupUri="MainWindow.xaml"`은 application 시작 시 해당 UI를 자동으로 보여준다. 그런데 `OnStartup`에서도 `new MainWindow()`를 만들고 `Show()`를 호출하고 있으니 window가 두 개 생긴다.
둘 중 하나를 없애주면 된다. 보통은 `StartupUri`를 쓰거나, `StartupUri`를 지우고 `OnStartup`에서 직접 window를 만들거나 둘 중 하나만 선택한다.

[Problem With WPF C# App Spawning Two Main Windows](https://stackoverflow.com/questions/2923431/problem-with-wpf-c-sharp-app-spawning-two-main-windows)

### Ref

- Application.StartupUri: <https://learn.microsoft.com/en-us/dotnet/api/system.windows.application.startupuri>
- Application.MainWindow: <https://learn.microsoft.com/en-us/dotnet/api/system.windows.application.mainwindow>
- Application.OnStartup: <https://learn.microsoft.com/en-us/dotnet/api/system.windows.application.onstartup>
