+++
title="WPF"
date=2025-05-21

[taxonomies]
categories = ["post"]
tags = ["C#","WPF"]
+++

### namespace
WPF 예제를 보면 `x:Type` 같은 표현이 보이는데, 여기서 `x`는 도대체 무엇일까?
파일 제일 위쪽을 보면 보통 이런 선언이 있다.
```
xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
```
C# 코드에서 `using` namespace를 적는 것과 비슷하게, XAML에서는 `xmlns`로 XML namespace prefix를 선언한다. 다만 완전히 같은 개념은 아니다. `xmlns:x`는 `x:Class`, `x:Key`, `x:Name`, `x:Type` 같은 XAML language feature를 쓰기 위한 prefix다. WPF XAML은 case-sensitive라서 element와 property 이름의 대소문자도 정확히 맞아야 한다.

<br>

### ContentControl
`ContentControl`은 WPF에서 매우 기본이 되는 control이다.
`App.xaml.cs`는 프로그램의 시작점이고, 여기서 보통 `MainWindow`를 띄운다.
`MainWindow`는 `Window`를 상속하고, `Window`는 다시 `ContentControl`을 상속한다.
즉 우리는 WPF에서 `ContentControl`을 알게 모르게 계속 사용하고 있는 셈이다.
```cs
<ContentControl Content="{Binding CurrentViewModel}" />
```
```cs
<ContentControl Content="{StaticResource person}" />
```
`UIViewModel`을 표시할 view를 `<views:UIView>`로 매핑하려면 `DataTemplate` 안에 넣어 줄 수 있다.
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

Binding="{Binding Name}"
Binding="{Binding ElementName=Name}"

<br>

### 데이터 표시
WPF를 공부하다 보면 데이터를 table이나 list 형태로 표현해야 할 때가 있다. 예제에서 자주 보이는 control들은 다음과 같다.

[DataGrid](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.datagrid)
vs [GridView](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/controls/gridview-overview)
vs [ItemsControl](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.itemscontrol)
vs [ListBox](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.listbox)
vs [ListView](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.listview)
<br>
상속 순서는 다음과 같다.
<br>
ItemsControl -> Selector -> ListBox -> ListView
그리고 `GridView`는 `ListView.View`에 넣는 view mode이다. `DataGrid`처럼 standalone item control이 아니라 `ViewBase`를 상속해서 `ListView`의 데이터를 column 형태로 보여준다.
더 자세한 내용은 [여기서](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/controls/control-library) 보자

<br>

### {Binding PropertyName} and {Binding Path=PropertyName}

https://stackoverflow.com/questions/4306657/difference-between-binding-propertyname-and-binding-path-propertyname

<br>

### SelectedValue vs SelectedItem

<br>

### TextBox에서 user input 받기
<TextBox Width="200" Height="25" FontSize="14"
                 Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}" />

<br>

### Ref

- XAML overview: <https://learn.microsoft.com/en-us/dotnet/desktop/wpf/xaml/>
- ContentControl: <https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.contentcontrol>
- ItemsControl: <https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.itemscontrol>
- ListBox: <https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.listbox>
- ListView: <https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.listview>
- GridView: <https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.gridview>
