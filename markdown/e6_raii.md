---
title: Resource acquisition is initialization (RAII)
author: Dmytro Ivanov
date: Nov, 2021
---

# Let's start with the problem

## Problem definition

```{.csharp .number-lines}
void Function() {
	EnableSomethingExternal();

	if (DoSomething1() == Error) {
		DisablingSomethingExternal();
		return;
	}

	if (DoSomething2() == Error) {
		DisablingSomethingExternal();
		return;
	}

	if (DoSomething3() == Error) {
		DisablingSomethingExternal();
		return;
	}

	DoSomethingFinally();
	DisablingSomethingExternal();
}
```

This code is tedious.

## Solution: Structured programming

One attempt could be to use "structured programming" and avoid early exits:

```{.csharp .number-lines}
void Function() {
	EnableSomethingExternal();

	if (DoSomething1() != Error) {
		if (DoSomething2() != Error) {
			if (DoSomething3() != Error) {
				DoSomethingFinally();
			}
		}
	}

	DisablingSomethingExternal();
}
```

Early programming started with what is now known as [Non-structured programming](https://en.wikipedia.org/wiki/Non-structured_programming),
which was generally programming without procedures/functions, and instead relying on GOTO for control flow.

[Structured programming](https://en.wikipedia.org/wiki/Structured_programming) emerged in ALGOL 58 created in 1958 as antipode to non-structured programming.
Famous developments include [Böhm–Jacopini theorem](https://en.wikipedia.org/wiki/Structured_program_theorem) in 1966 (also known as "Structured program theorem")
and [Nassi–Shneiderman diagrams](https://en.wikipedia.org/wiki/Nassi%E2%80%93Shneiderman_diagram) in 1972.
One of the most influential works was the "Go To Statement Considered Harmful" open letter by Edsger W. Dijkstra in 1968 which coined the term.

While most non-structured programming was gradually phased out in 60-70-x, elements of it are still in use today: early `return`, `goto`, exceptions, `break`, `continue`, etc (there is no clear consensus if last two are non-structured).

## Solution: Exceptions as control flow

Another attempt could be to use exceptions as control flow:

```{.csharp .number-lines}
void Function() {
	try {
		EnableSomethingExternal();

		if (DoSomething1() == Error)
			throw;

		if (DoSomething2() == Error)
			throw;

		if (DoSomething3() == Error)
			throw;
	}
	catch() {}
	finally {
		DisablingSomethingExternal();
	}
}
```

Java is famous for using exceptions as control flow, generally it's considered an anti-pattern.

# RAII

## What is RAII?

From [Wiki](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization):

> Resource acquisition is initialization (RAII) is a programming idiom ... In RAII, holding a resource ... is tied to object lifetime: resource allocation (or acquisition) is done during object creation (specifically initialization), by the constructor, while resource deallocation (release) is done during object destruction (specifically finalization), by the destructor.

RAII is part of resource management problem.

Remember how aggregation separates lifetimes of nested objects?
RAII is inversion of that, and instead connects a lifetime of disconnected object/pointer.

## RAII in C++

Embedding external lifetime managing into the scope based lifetime of our class instance:

```{.cpp .number-lines}
class RAIIContainer {
public:
	RAIIContainer(){
		EnableSomethingExternal();
	}
	~RAIIContainer(){
		DisablingSomethingExternal();
	}
}

void Function() {
	RAIIContainer container;

	if (DoSomething1() == Error)
		return;

	if (DoSomething2() == Error)
		return;

	if (DoSomething3() == Error)
		return;

	DoSomethingFinally();
}
```

## RAII in C\#

C# doesn't have direct support for RAII idiom. But we can achieve the same properties nevertheless.

```{.csharp .number-lines}
class RAIIContainer : IDisposable {
	public RAIIContainer() {
		EnableSomethingExternal();
	}

	// There are so many gotchas here, more on that later
	public void Dispose() {
		if (!disposed) {
			DisablingSomethingExternal();
			disposed = true;
		}
	}

	private bool disposed = false;
}

void Function() {
	using (var container = new RAIIContainer()) {
		if (DoSomething1() == Error)
			return;

		if (DoSomething2() == Error)
			return;

		if (DoSomething3() == Error)
			return;

		DoSomethingFinally();
	}
}
```

## RAII in C\#

With C# 8.0 you can also write:

```{.csharp .number-lines}
class RAIIContainer : IDisposable {
	...
}

void Function() {
	using var container = new RAIIContainer();

	if (DoSomething1() == Error)
		return;

	if (DoSomething2() == Error)
		return;

	if (DoSomething3() == Error)
		return;

	DoSomethingFinally();
}
```

# Deep dive into IDisposable

## Deep dive into IDisposable: Implementation

All IDisposable is really is is just this:

```{.csharp .number-lines}
namespace System {
	public interface IDisposable {
		void Dispose();
	}
}

class RAIIContainer : IDisposable {
	// On this level, it's exactly same as any other method in any other interface
	public void Dispose() {
		...
	}
}

void Function() {
	using (var container = new RAIIContainer()) {
		...
	}
}
```

## Deep dive into IDisposable: Syntax sugar

As per the [language specification](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/statements#the-using-statement) `using` unrolls to:

```{.csharp .number-lines}
namespace System {
	public interface IDisposable {
		void Dispose();
	}
}

class RAIIContainer : IDisposable {
	public void Dispose() {
		...
	}
}

void Function() {
	var container = new RAIIContainer();
	try {
		...
	} finally {
		if (container != null) { // check only present if value is nullable value type
			((IDisposable)resource).Dispose(); // cast here if value is non-dynamic, cast before try statement if value is dynamic
		}
	}
}
```

There is no magic, it's literally just try-finally block with calling `Dispose` method in the end.

[Sharplab exercise](https://sharplab.io/#v2:EYLgxg9gTgpgtADwGwBYA0ATEBqAPgAQCYBGAWACgiACAJQEEBJBgYQgDsAXAQwEs2YoVEFQYARHgGcADhAldgAGxhUA3hSoaq+AMy1GLdtz4CAFAEpV6zQF8KVjQHoHVACoALAcq6wqEiFQBbLjYATyoAcwgOMDcuCSoPWDRA6GV2Kg5YjioFLg4Bey1dfBQqcWlZGHNLck06qh4AMyoTAEIMSRkJGAwLNVr6wY6K7owqAF4MqABXGABuQrrbAY1lwqkoHgA3POVgCAgFKmGunomqRq4FboXyNcpCKgAxA4p+upKtAFYOHnZq96DLQABhaO0EkE4vH4gkm/AA7nomKwocYoOY+osbIVltYgA===).

## Deep dive into IDisposable: Finalizers

[Finalizer](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/finalizers) is a method that is called during object collection by the garbage collector. Note object collection is not tied to object lifetime scope.

It's the most useful for handling external non-managed resources. Just relying on `Dispose` is not enough, because if `Dispose` is not called then you will be leaking external non-managed resources. If you're not dealing with non-managed handles, generally finalizers are not needed.

```{.csharp .number-lines}
class MyObject {
	~MyObject() { // Finalizer
		// Cleanup statements
	}
	// Note you can't call finalizer manually, it can only be called by GC.
}
```

Which is really a syntax sugar for:
```{.csharp .number-lines}
class MyObject {
	// This method is called by GC right before it will destroy objects memory.
	// You will never see Finalize unless you write C++/CLI.
	protected override void Finalize() {
		try {
			// Cleanup statements
		} finally {
			base.Finalize();
		}
	}
}
```

GC has a special "finalizer queue" where it puts objects that require a call to finalizer.
This is mostly to ensure finalizer code runs in a sensible point in time.
As most objects don't have any finalizer, GC is more free to deal with them at any point of execution.

During finalizer call, all managed children are already freed.

You can talk to GC to tell it not to call the finalizer of your object, just call `GC.SuppressFinalize(obj);` at any point.

Note finalizers might not be called during application shutdown.

## Deep dive into IDisposable: Combining them together

Don't worry, Rider will generate a template for you. Or same code can be copy-pasted from Microsoft docs.

```{.csharp .number-lines}
class RAIIContainer : IDisposable {
	private IntPtr externalHandle;
	private Foo managedHandle; // Foo is IDisposable also
	private bool disposed = false; // Dispose can be called multiple times, that's why Unity has DisposeSentinel

	public RAIIContainer() {
		externalHandle = CreateSomethingExternal();
		managedHandle = new Foo();
	}
	public void Dispose() {
		Dispose(iAmBeingCalledFromDisposeAndNotFinalize: true);
		GC.SuppressFinalize(this);
	}
	~RAIIContainer() { // finalizer
		Dispose(iAmBeingCalledFromDisposeAndNotFinalize: false);
	}

	protected void Dispose(bool iAmBeingCalledFromDisposeAndNotFinalize) {
		if(disposed) // already disposed
			return;

		// Dealing with non-managed handles
		ReleaseSomethingExternal(externalHandle);
		externalHandle = null;

		// Dealing with managed handles
		// Note if we're called from finalizer, managed handle is already cleaned up
		if(iAmBeingCalledFromDisposeAndNotFinalize) {
			managedHandle.Dispose();
			managedHandle = null;
		}

		disposed = true;
	}
}
```

# Extra

## External non-managed handles: SafeHandle

> The point of a SafeHandle is to ensure that your class object doesn't get destroyed while you passed the handle to native code.

```{.csharp .number-lines}
class MySafeHandle : SafeHandleZeroOrMinusOneIsInvalid
{
	public MySafeHandle(IntPtr handle)
		:base(true)
	{
		SetHandle(handle);
	}

	[ReliabilityContract(Consistency.WillNotCorruptState, Cer.Success)]
	protected override bool ReleaseHandle() {
		ReleaseSomethingExternal(handle);
		return true;
	}
}

class MyClass : IDisposable
{
	private MySafeHandle handle;

	public MyClass() {
		handle = new SomeSafeHandle(CreateSomethingExternal());
	}

	public void Dispose() {
		handle.Dispose();
	}
}
```

## RAII in Unity

- Generally `using` and `IDisposable` works for low level stuff (files, network, etc).
- Generally `MonoBehaviour` used for higher level stuff.
- For resources like meshes/textures/etc: Asset references (via Asset Database) and Addressables. E.g. `Resources.UnloadUnusedAssets`.

```{.csharp .number-lines}
public class Foo : MonoBehaviour
{
	// Don't really need to care when Mesh is loaded and when it's unloaded
	public Mesh Mesh;
	...
}
```

# Self study

## Self study

- In which situations RAII is beneficial?
- How does `using` work in C#?

Optionally:

- [Wiki: RAII](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization)
- [SO: Proper use of the IDisposable interface](https://stackoverflow.com/questions/538060/proper-use-of-the-idisposable-interface)
- [.NET: IDisposable Interface](https://docs.microsoft.com/en-us/dotnet/api/system.idisposable?view=net-5.0)
