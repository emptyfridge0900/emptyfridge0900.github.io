+++
title="Variance"
date=2025-01-25

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++


C#에서 covariance와 contravariance는 **array type**, **delegate type**, **generic type argument**에 대해 implicit reference conversion을 가능하게 한다.

Covariance는 한국어로 공변성이라고 하고 자식(파생,하위) 타입이 부모 타입의 변수로 할당되는 형변환\
Contravariance는 한국어로 반공변성이라고 하고 부모 타입이 자식 타입 변수로 할당되는 형변환

## Array covariance
이것은 우리가 흔히 아는 개념이다. 자식 객체 배열을 부모 객체 배열 변수에 할당할 수 있다.
```cs
object[] array = new String[10];
```
다만 array covariance는 type-safe하지 않다. 위 배열에 `array[0] = 10;` 같은 값을 넣으면 compile은 될 수 있지만 runtime에 `ArrayTypeMismatchException`이 난다.

## [Variance in Generic Interfaces](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/covariance-contravariance/variance-in-generic-interfaces)
```cs
public class Animal{
	public string Name{get;set;}
}

public class Cat:Animal{

}
public class Dog:Animal{

}
public class Schnauzer:Dog{

}
public class GermanShepherd : Dog{

}
```
Covariance
```cs
IEnumerable<GermanShepherd> germanShepherds = new List<GermanShepherd>();
IEnumerable<Dog> dogs = germanShepherds;
dogs = new List<Schnauzer>();
```
주의할 점은 variant interface를 구현한 class 자체는 여전히 invariant라는 것이다.
```cs
// compiler error
List<Dog> dogList = new List<GermanShepherd>();
// compile 성공
IEnumerable<Dog> dogEnumerable = new List<GermanShepherd>();
```
Contravariance
```cs
class BaseComparer : IEqualityComparer<Dog>
{
    public int GetHashCode(Dog baseInstance)
    {
        return baseInstance.GetHashCode();
    }
    public bool Equals(Dog x,Dog y)
    {
        return x == y;
    }
}
// 덜 파생된 타입용 comparer를 더 구체적인 타입에 사용할 수 있다. 이것이 contravariance다.
IEqualityComparer<GermanShepherd> comparer = new BaseComparer();
// compile fail
IEqualityComparer<Animal> comparer = new BaseComparer();
```


## [Creating Variant Generic Interfaces](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/covariance-contravariance/creating-variant-generic-interfaces)
covariant 또는 contravariant generic type parameter를 가진 generic interface를 **variant** interface라고 부른다.
```cs
interface IDogHospital<in T,out R>
{
    R DoSomething(T t);

	// compile error: out keyword가 붙은 type은 method argument로 사용할 수 없다.
	//void DoSomething(R r);

	// compile error: in keyword가 붙은 type은 return type으로 사용할 수 없다.
	//T DoSomething();

	void DoSomething(Action<R> callback);
	R DoSomethingElse<TT>() where TT:T;
	// compile error
	//R DoSomethingElse<TT>() where TT:R;
}
```
구현 예시:
```cs
public class DogHospital<T,R> : IDogHospital<T, R>
{
    public R DoSomething(T s)
    {
        throw new NotImplementedException();
    }

    public void DoSomething(Action<R> callback)
    {
        throw new NotImplementedException();
    }

    public R DoSomethingElse<TT>() where TT : T
    {
        throw new NotImplementedException();
    }
}
var hospital = new DogHospital<Animal,Dog>();
Animal animal = hospital.DoSomething(new Schnauzer());
Dog dog = hospital.DoSomethingElse<Cat>();
```

## Ref

- Covariance and contravariance: <https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/covariance-contravariance/>
- Variance in generic interfaces: <https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/covariance-contravariance/variance-in-generic-interfaces>
- Creating variant generic interfaces: <https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/covariance-contravariance/creating-variant-generic-interfaces>
