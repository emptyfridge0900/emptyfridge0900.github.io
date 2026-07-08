+++
title="Null Operator"
date=2025-01-24

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

## [Null-conditional operators (?.) (?[])](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/member-access-operators#null-conditional-operators--and-)

The null-conditional operator applies member access `?.` or element access `?[]` only when the operand is not null. Otherwise, it returns null.

```cs
var device = GetDevice();
if(device !=null && device.DeviceId > 0){
  //do something
}
...
// If device is null, device?.DeviceId is null and the whole comparison becomes false
var device = GetDevice();
if(device?.DeviceId > 0){
  //do something
}
```

More precisely, `device?.DeviceId` becomes a nullable value such as `int?`, and the next `> 0` comparison becomes a `bool`. If `device` is null, the comparison result is false. If you want to make the fallback value explicit, use something like `(device?.DeviceId ?? 0) > 0` instead of `device?.DeviceId > 0`.

```cs
if(PropertyChanged != null){
  PropertyChanged();
}
...
PropertyChanged?.Invoke();
```

## [Null-coalescing operators (??)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/null-coalescing-operator)

The null-coalescing operator `??` returns the left operand if it is not null. Otherwise, it returns the right operand.

```cs
int? a = null;
int b = a ?? 10;
//b is 10
```

## [Null-coalescing assignment (??=)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/null-coalescing-operator)

The null-coalescing assignment operator `??=` assigns the right operand to the left operand only when the left operand evaluates to null.

```cs
int? a = null;
a ??= 10;
// now a is 10
```

## [Null-forgiving operator (T!)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/null-forgiving)

In an enabled nullable annotation context, the null-forgiving operator suppresses nullable warnings for the preceding expression.
At runtime, the expression `x!` evaluates to the result of the underlying expression `x`.

```cs
Person? person = FindPerson();

// nullable warning: person might be null
Console.WriteLine(person.Name);

// no nullable warning, but runtime behavior is unchanged
Console.WriteLine(person!.Name);
```

`!` tells the compiler, "I believe this is not null, so suppress the nullable warning." It does not add a runtime null check. If `person` is actually null, `person!.Name` can still throw `NullReferenceException`. Warnings do not block compilation by default, but if the project treats warnings as errors, they can fail the build.

### Ref

- Null-conditional operators: <https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/member-access-operators#null-conditional-operators--and->
- Null-coalescing operators: <https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/null-coalescing-operator>
- Null-forgiving operator: <https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/null-forgiving>
- Nullable reference types: <https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/null-safety/nullable-reference-types>
