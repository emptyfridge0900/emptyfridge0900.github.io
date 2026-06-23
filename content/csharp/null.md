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
// device가 null이면 device?.DeviceId는 null이고, 전체 비교 결과는 false가 된다
var device = GetDevice();
if(device?.DeviceId > 0){
  //do something
}
```
정확히는 `device?.DeviceId`까지는 `int?` 같은 nullable value가 되고, 그 다음 `> 0` 비교 결과가 `bool`이 된다. `device`가 null이면 비교 결과는 false가 된다. null일 때 대체값을 명시하고 싶으면 `device?.DeviceId > 0` 대신 `(device?.DeviceId ?? 0) > 0`처럼 쓸 수 있다.

```cs
if(PropertyChanged != null){
  PropertyChanged();
}
...
PropertyChanged?.Invoke();
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
a ??= 10;
// now a is 10
```
## [Null-forgiving operator (T!)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/null-forgiving)
사용하도록 설정된 null 허용 주석 컨텍스트에서 null 허용 연산자를 사용하여 이전 식에 대한 모든 null 허용 경고를 억제합니다. 
런타임에서 x! 식은 기본식 x의 결과로 계산됩니다.
```cs
Person? person = FindPerson();

// nullable warning: person might be null
Console.WriteLine(person.Name);

// no nullable warning, but runtime behavior is unchanged
Console.WriteLine(person!.Name);
```

`!`는 "내가 보기에는 null이 아니니 nullable warning을 억제해줘"라고 compiler에게 알려주는 연산자다. runtime null check를 추가하지 않는다. 실제로 `person`이 null이면 `person!.Name`도 `NullReferenceException`이 날 수 있다. warning은 기본적으로 compile을 막지 않지만, project가 warnings-as-errors로 설정되어 있으면 build 실패로 이어질 수 있다.

### Ref

- Null-conditional operators: <https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/member-access-operators#null-conditional-operators--and->
- Null-coalescing operators: <https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/null-coalescing-operator>
- Null-forgiving operator: <https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/null-forgiving>
- Nullable reference types: <https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/null-safety/nullable-reference-types>
