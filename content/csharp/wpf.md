+++
title="WPF"
date=2025-05-21

[taxonomies]
categories = ["post"]
tags = ["C#","WPF"]
+++

### namespace  
wpf 예제를 보면 x:Type 이런게 보이는데 도대체 x는 뭔가?  
파일 제일 위쪽을 보면 이런게 보일텐데  
```
xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"  
``` 
C# 코드에서 `using` namespace를 적는 것과 비슷하게, XAML에서는 `xmlns`로 XML namespace prefix를 선언한다. 다만 완전히 같은 개념은 아니다. `xmlns:x`는 `x:Class`, `x:Key`, `x:Name`, `x:Type` 같은 XAML language feature를 쓰기 위한 prefix다. WPF XAML은 case-sensitive라서 element와 property 이름의 대소문자도 맞아야 한다.

<br>

### ContentControl
ContentControl은 기본이 되는 클래스  
app.xaml.cs 이 프로그램의 시작점인데 MainWindow를 돌린다.  
MainWindow는 Window를 상속하고 Window는 ContentControl를 상속한다.  
사실 우리는 ContentControl을 알게모르게 사용중인 것이다.  
```cs
<ContentControl Content="{Binding CurrentViewModel}" />
```
```cs
<ContentControl Content="{StaticResource person}" />
```
UIViewModel을 보여주는 뷰를 <views:UIView> 로 세팅
DataTemplate 안에 넣는 것을 보여준다  
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

### Display data
wpf를 공부하다 보면 데이터를 테이블이나 리스트 같은 걸로 표현해야 할때가 있는데 예제들에서 자주 보이는 컨트롤러들은 다음과 같다  

[DataGrid](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.datagrid) 
vs [GridView](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/controls/gridview-overview) 
vs [ItemsControl](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.itemscontrol) 
vs [ListBox](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.listbox) 
vs [ListView](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.listview)
<br>
상속의 순서는
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

### Textbox getting userinput
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
