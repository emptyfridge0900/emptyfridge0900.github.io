+++
title="wpf"
date=2025-05-13

[taxonomies]
categories = ["post"]
tags = ["C#","WPF"]
+++

Visual studio에서 wpf 프로그램을 실행할때 window가 두개 뜰때가 있다.  
App.xmal
```cs hl_lines=5
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

xaml과 xaml.cs 모두 MainWindow를 사용한다고 명시했으니까 두개가 뜨는거다.  
둘 중 하나를 없애주면 된다

[Problem With WPF C# App Spawning Two Main Windows](https://stackoverflow.com/questions/2923431/problem-with-wpf-c-sharp-app-spawning-two-main-windows)