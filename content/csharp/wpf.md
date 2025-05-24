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
C# 코드에서 파일 첫줄에 네임스페이스 적는 것이랑 똑같다  

<br>

### ContentControl
ContentControl은 기본이 되는 클래스  
app.xaml.cs 이 프로그램의 시작점인데 MainWindow를 돌린다.  
MainWindow는 Window를 상속하고 Window는 ContentControl를 상속한다.  
사실 우리는 ContentControl을 알게모르게 사용중인 것이다.  
```cs
<ContentControl content={Binding CurrentViewModel}/>
```
```cs
<ContentControl content={StaticResource person}/>
```
UIViewModel을 보여주는 뷰를 <views:UIView> 로 세팅
DataTemplate 안에 넣는 것을 보여준다  
```cs
<ContentControl content={Binding CurrentViewModel}>
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

[DataGrid](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.datagrid?view=windowsdesktop-9.0) 
vs [GridView](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/controls/gridview-overview) 
vs [ItemsControl](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.itemscontrol?view=windowsdesktop-9.0) 
vs [ListBox](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.listbox?view=windowsdesktop-9.0) 
vs [ListView](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.listview?view=windowsdesktop-9.0)
<br>
상속의 순서는
<br>
ItemControl->ListBox->ListView  
그리고 GridView view mode는 ListView Control의 view mode이다.  
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