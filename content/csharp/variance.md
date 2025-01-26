+++
title="Variance"
date=2025-01-25

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++


In C#, covariance and contravariance enable implicit reference conversion for **array types**, **delegate types**, and **generic type arguments**

Covariance는 한국어로 공변성이라고 하고 자식(파생,하위) 타입이 부모 타입의 변수로 할당되는 형변환\
Contravariance는 한국어로 반공변성이라고 하고 부모 타입이 자식 타입 변수로 할당되는 형변환

## Covariance for arrays
이것은 우리가 흔히 아는 개념이다. 자식객체가 부모객체에 할당 
```cs
object[] array = new String[10];  
```

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
It is also important to remember that classes that implement variant interfaces are still invariant. 
```cs
//this generates a compiler error
List<Dog> = new List<GermanShepherd>();
//this compiles
IEnumerable<Dog> = new List<GermanShepherd>();
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
//less derived class instance assigned to more derived class instance. so it's contravariance
IEqualityComparer<GermanShepherd> comparer = new BaseComparer();
//compile fail
IEqualityComparer<Animal> comparer = new BaseComparer();
```


## [Creating Variant Generic Interfaces](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/covariance-contravariance/creating-variant-generic-interfaces)
A generic interface that has covariant or contravariant generic type parameters is called **variant**.
```cs
interface IDogHospital<in T,out R>
{
    R DoSomething(T t);
	
	//compile error, type with out keyword cannot be an argument of a method
	//void DoSomething(R r);
	
	// compile error, type with in keyword cannot be a return type
	//T DoSomething();
	
	void DoSomething(Action<R> callback);
	R DoSomethingElse<TT>() where TT:T;
	//compile error
	//R DoSomethingElse<TT>() where TT:R;
}
```
Implementation
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
