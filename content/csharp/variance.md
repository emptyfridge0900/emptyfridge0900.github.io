+++
title="Variance"
date=2025-01-25

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

In C#, covariance and contravariance enable implicit reference conversion for **array types**, **delegate types**, and **generic type arguments**.

Covariance means a value of a child, derived, or more specific type can be assigned to a variable of a parent type.
Contravariance means a value typed for a parent type can be assigned where a child type is expected, in specific input-position scenarios.

## Covariance for arrays

This is the familiar direction: a child object can be assigned to a parent object reference.

```cs
object[] array = new String[10];
```

However, array covariance is not type-safe. If you insert a value like `array[0] = 10;`, it may compile, but it throws `ArrayTypeMismatchException` at runtime.

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

Covariance:

```cs
IEnumerable<GermanShepherd> germanShepherds = new List<GermanShepherd>();
IEnumerable<Dog> dogs = germanShepherds;
dogs = new List<Schnauzer>();
```

It is also important to remember that classes that implement variant interfaces are still invariant.

```cs
//this generates a compiler error
List<Dog> dogList = new List<GermanShepherd>();
//this compiles
IEnumerable<Dog> dogEnumerable = new List<GermanShepherd>();
```

Contravariance:

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

Implementation:

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
