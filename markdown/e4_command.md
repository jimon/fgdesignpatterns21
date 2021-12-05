---
title: Command pattern
author: Dmytro Ivanov
date: Nov, 2021
---

# Straight into code

## Command pattern is

```C#
public interface ICommand
{
	void Execute();
}

public class SomeCommand : ICommand
{
	public void Execute()
	{
		...
	}
}

public class Invoker
{
	private ICommand command;

	public void DoSomething()
	{
		...
		command.Execute();
		...
	}
}
```

## Undo/Redo

Most common use case in game development is undo/redo system in editors:

::: columns
:::: column

```C#
public interface IUndoRedoCommand
{
	void Perform();
	void Undo();
}

public class MoveGameObject : IUndoRedoCommand
{
	public GameObject Target;
	public Vector3 NewPosition;
	private Vector3 _oldPosition;

	public void Perform()
	{
		_oldPosition = Target.transform.position;
		Target.transform.position = NewPosition;
	}

	public void Undo()
	{
		Target.transform.position = _oldPosition;
	}
}
```
::::
:::: column

```C#
public class UndoRedoSystem
{
	private Stack<IUndoRedoCommand> _undoStack = new();
	private Stack<IUndoRedoCommand> _redoStack = new();

	public void Perform(IUndoRedoCommand command)
	{
		command.Perform();
		_undoStack.Push(command);
	}

	public void Undo()
	{
		if (_undoStack.TryPop(out var command))
		{
			command.Undo();
			_redoStack.Push(command);
		}
	}

	public void Redo()
	{
		if (_redoStack.TryPop(out var command))
		{
			command.Perform();
			_undoStack.Push(command);
		}
	}
}
```

::::
:::

## Historical context

Some language type trivia:

- In C/C++ a function pointer is literally 4/8 byte value pointing to location in memory where the function's code literally is. (Well, actually getting the function pointer is quite a bit more involved, because you might not know where your code will be in memory, so `&foo` might generate lookup to "Global Offset Table" or alike which will be filled by Dynamic Linker in runtime, Dynamic Linker is the thing that actually loads .exe/ELF/etc files in memory)

- In C++ method pointer on some compilers is function pointer to the method code + magic adjustment to `this` ptr. You still need to pass `this` manually. Like:
```C++
void CallMethodPtr(const SomeObject &object /* I am 8 bytes reference to this */, void (SomeObject::*MemberFunction)() const /* I am 16 bytes member pointer on Clang/GCC */) {
	(object.*MemberFunction)();
}
```
Also during your career you might see something called "C++ fast delegates", which is an alternative implementation of being able to use method pointers by only using function pointers. This is achieved by creating wrapper functions for every method you get pointer to.

- In C# a delegate is fairly complex data structure that can accommodate method pointer. Delegate type is a class that inherits from `System.MulticastDelegate`, has plenty of stuff, importantly, can store value of `this` in the instance of delegate.

## Historical context

Command pattern is from 80-x early 90-x, language landscape was very different back than.

One way to look at it is a workaround of C++ limitations of not storing `this` in member pointer.

Another way to look at it is workaround of C++ limitations of not having higher-order functions transparently accepting method pointers.

This was later on fixed with `std::function` and lambdas, but was not really a problem in C# to begin with.

## Higher-order functions

From [Wiki](https://en.wikipedia.org/wiki/Higher-order_function):

> a higher-order function is a function that does at least one of the following:
> - takes one or more functions as arguments (i.e. procedural parameters),
> - returns a function as its result.

In command pattern, "Invoker" then would be a higher-order function.

## C# Lambdas

Let's see how this:

```C#
public class Foo {
	delegate int AddOneDelegate(int number);

	static int DelegateCaller(AddOneDelegate oneDelegate) {
		return oneDelegate(1);
	}

	static int AddOneAsStaticMethod(int x) {
		return x + 1;
	}

	public void Bar() {
		var a = 1;
		var b = DelegateCaller(AddOneAsStaticMethod);
		var d = DelegateCaller((x) => {return x + 1;});
		var e = DelegateCaller((x) => x + a); // same as (x) => {return x + foo;}
	}
}
```

Is [getting compiled in sharplab](https://sharplab.io/#v2:EYLgxg9gTgpgtADwGwBYA0ATEBqAPgAQCYBGAWACh8BmAAiJoDEIIaBvCmzmjGAGxgDmAQwAuMGgEsAdiJoBBDBgDyUmABE+g0TAAU02VICuAW2AwoASgDcFDl3zEkkmTQ39hYgMJDe/KDoVlVTctMRoIYM0PGAs2CgBILjoAdnDI920dYms7TgBfXJpChyd9eUUVGDkAZwBlEVEJMABZGBEACwgMPRcEWPZyJKT8VIQabBpiG0GuApnOYtp8FBoAISF/fsKkgDcNmiEaAF5J6aGuPagaYGPXKO1vX3MAitUa+saWts6MHPmhy7cW4haKPPw6HR9Y4APjYIxoYwmUzyf3OnEB4hOIIePnBkNiR1hiIO1hoAHoyTRqkJjOIhNUaPiYXDRuMaAAzZhWOZJOZ5IA===).

## Final remarks

- Might have some use in game logic, for example around generating commands in one place and executing them in another place "in the right time".
- If your command in command pattern has only one method, consider using delegates instead. In C++ that would be "fast delegates".
- Undo/Redo often is implemented via command patterns, but command doesn't necessarily have to have a "undo" method.

# Self study

## Self study

- What are the common use cases for command pattern?
- [Game Programming Patterns: Command](http://gameprogrammingpatterns.com/command.html)

Optional:
- How C# lambdas capture outside variables?
- How C# lambdas are implemented? (tip: it's a class)
