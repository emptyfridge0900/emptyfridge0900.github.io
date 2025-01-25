+++
title="Null Operator"
date=2025-01-24

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++


## [Null-conditional operators (?.) (?[])](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/member-access-operators#null-conditional-operators--and-)
null 조건부 연산자는 피연산자가 null이 아닌 것으로 평가되었을 때만 멤버 액세스, ?. 또는 요소 액세스, ?[], 연산을 피연산자에게 적용하며, 그렇지 않으면 null을 반환합니다.
```cs
var device = GetDevice();
if(device !=null && device.DeviceId > 0){
  //do something
}
...
// device가 null이면 null을 반환하는데, boolean 값이 아님에도 에러가 뜨지 않는다
var device = GetDevice();
if(device?.DeviceId > 0){
  //do something
}
```
```cs
if(delegate != null){
  delegate();
}
...
delegate?.Invoke();
```

## [Null-coalescing operators (??)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/null-coalescing-operator)
null 병합 연산자 ??는 null이 아닌 경우 왼쪽 피연산자의 값을 반환합니다. 
```cs
int? a = null;  
int b = a ?? 10;
//b is 10
```

## [Null-coalescing assignment (??=)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/null-coalescing-operator)
null 병합 대입 연산자 ??=(은)는 왼쪽 피연산자가 null(으)로 계산되는 경우에만 오른쪽 피연산자의 값을 왼쪽 피연산자에 할당합니다. 
```cs
int? a = null;
a ?? = 10;
// now a is 10
```
## [Null-forgiving operator (T!)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/null-forgiving)
사용하도록 설정된 null 허용 주석 컨텍스트에서 null 허용 연산자를 사용하여 이전 식에 대한 모든 null 허용 경고를 억제합니다. 
런타임에서 x! 식은 기본식 x의 결과로 계산됩니다.
```cs
//compile time warning, cannot compile the code
var doorWithIndex = Doors.Select((door,idx) => (door,idx)).FirstOrDefault();
var door = doorWithIndex.Value.door;
...
//no compile time warning
var doorWithIndex = Doors.Select((door,idx) => (door,idx)).FirstOrDefault();
var door = doorWithIndex!.Value.door;
```
