---
title: Specification pattern
author: Dmytro Ivanov
date: Nov, 2021
---

# Boolean equation

## Boolean equation

```{.csharp .number-lines}
bool result = (!a) | (b & c) | (d ^ e) | (f && g) | (h || i);
```

Quick recap:

- `!` is logical negation
- `~` is bitwise negation
- `&` is logical and bitwise AND
- `|` is logical and bitwise OR
- `^` is logical and bitwise eXclusive-OR (XOR)
- `&&` is conditional logical AND
- `||` is conditional logical OR
- `<<` and `>>` are bitwise shifts

## Specification pattern

Extended boolean equation encoded as tree of items via composite pattern:

::: columns
:::: column

```{.csharp .number-lines}
public interface ISpecification {
	bool IsSatisfiedBy(object candidate);
}

public class EqualToSpecification : ISpecification {
	public IComparable Value;

	// Expression body definitions
	public bool IsSatisfiedBy(object candidate) => Value.CompareTo(candidate) == 0;
}

public class AndSpecification : ISpecification {
	public ISpecification Left;
	public ISpecification Right;

	public bool IsSatisfiedBy(object candidate) =>
		Left.IsSatisfiedBy(candidate) && Right.IsSatisfiedBy(candidate);
}

public class OrSpecification : ISpecification {
	public ISpecification Left;
	public ISpecification Right;

	public bool IsSatisfiedBy(object candidate) =>
		Left.IsSatisfiedBy(candidate) || Right.IsSatisfiedBy(candidate);
}

public class NotSpecification : ISpecification {
	public ISpecification Other;

	public bool IsSatisfiedBy(object candidate) =>
		!Other.IsSatisfiedBy(candidate);
}
```

::::
:::: column

```{.csharp .number-lines}
var notEqualTwo = new NotSpecification { // oh geez
	Other = new EqualToSpecification {
		Value = 2
	}
};

var a = notEqualTwo.IsSatisfiedBy(1); // True
var b = notEqualTwo.IsSatisfiedBy(2); // False
```

::::
:::

## NUnit example

If you combine specification pattern with fancy builder pattern you can achieve amazing ergonomics:

```{.csharp .number-lines}
Assert.That(..., Is.False);
Assert.That(..., Is.True);
Assert.That(..., Is.Null);
Assert.That(..., Is.Empty);
Assert.That(..., Is.TypeOf<...>());
Assert.That(..., Is.EqualTo(...));
Assert.That(..., Is.EqualTo(...).Within(...)); 
Assert.That(..., Is.Not.Null);
Assert.That(..., Is.Not.EqualTo(...)); // 
```

And most important, it will print human readable messages:

```
Expected: ... But was: ...
Expected: not equal to ... But was: ...
Expected: 1.0f +/- 0.1f But was: 2.0f Off by: -1.0f
```

## Predicate function

From [Microsoft docs](https://docs.microsoft.com/en-us/dotnet/api/system.predicate-1?view=net-6.0):

> Predicate<T> Delegate
> Represents the method that defines a set of criteria and determines whether the specified object meets those criteria.

```{.csharp .number-lines}
public delegate bool Predicate<in T>(T obj);
```

## Predicates and lambdas

```{.csharp .number-lines}
delegate bool IsSatisfiedBy(object obj);
...

IsSatisfiedBy notEqualTwo = (x) => (int)x != 2;

var a = notEqualTwo(1); // False
var b = notEqualTwo(2); // True
```

## Final remarks

- In a way, got obsolete by delegates and lambdas.
- Might be somewhat useful if your language doesn't have generic lambdas (like C#).
- Used a lot in test frameworks to print great readable error messages.
- Often used when you're trying to build an expression in another domain/library: Abstract Syntax Trees, SAT/SMT solvers, theorem provers, etc.
- Or otherwise doing something "on-the-expression" without using something like Roslyn Source Generators or alike.

# Self study

## Self study

- Use specification pattern by writing some NUnit tests with assertions (Easiest way to do it in Unity would be to use Unity Test Framework as it's built on top of NUnit, here is a [quick how to](https://docs.unity3d.com/Packages/com.unity.test-framework@1.1/manual/getting-started.html)).
- Look inside how NUnit `Is.` is implemented (click "Go to definition").
