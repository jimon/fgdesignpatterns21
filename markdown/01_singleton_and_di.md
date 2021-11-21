---
title: Singleton and Dependency Injection
author: Dmytro Ivanov
date: Nov, 2021
---

# Inventing the problem

## Let's start with the basics

In C# 10 you could write:

```{.csharp .number-lines}
using System;

var playerPosition = 0;

while (true)
{
	var key = Console.ReadKey().Key;

	if (key == ConsoleKey.LeftArrow)  playerPosition--;
	if (key == ConsoleKey.RightArrow) playerPosition++;

	Console.WriteLine($"position = {playerPosition}");
}
```

## Let's start with the basics

Moving out character controller to a static class

```{.csharp .number-lines}
using System;

while (true)
{
	var key = Console.ReadKey().Key;

	if (key == ConsoleKey.LeftArrow)  Player.MoveLeft();
	if (key == ConsoleKey.RightArrow) Player.MoveRight();

	Console.WriteLine($"position = {Player.Position}");
}

public static class Player
{
	public static int Position = 0;

	public static void MoveLeft()  {Position--;}
	public static void MoveRight() {Position++;}
}
```

## Let's start with the basics

Adding one more field

```{.csharp .number-lines}
using System;

while (true)
{
	var key = Console.ReadKey().Key;

	if (key == ConsoleKey.LeftArrow)  Player.MoveLeft();
	if (key == ConsoleKey.RightArrow) Player.MoveRight();

	Console.WriteLine($"position = {Player.Position} weapon = {Player.Weapon}");
}

public static class Player
{
	public static int Position = 0;
	public static int Weapon = 0;

	public static void MoveLeft()   {Position--;}
	public static void MoveRight()  {Position++;}
	public static void NextWeapon() {Weapon = (Weapon + 1) % 4;}
}
```

## Let's start with the basics

Moving out to normal class with single instance.
Render knows about character here.

```{.csharp .number-lines}
var player = new Character();

while (true)
{
	var key = Console.ReadKey().Key;

	if (key == ConsoleKey.LeftArrow)  player.MoveLeft();
	if (key == ConsoleKey.RightArrow) player.MoveRight();
	if (key == ConsoleKey.Spacebar)   player.NextWeapon();

	Render.RenderCharacter(player);
}

public static class Render
{
	public static void RenderCharacter(Character character)
	{
		Console.WriteLine($"position = {character.Position} weapon = {character.Weapon}");
	}
}

public class Character
{
	public int Position = 0;
	public int Weapon = 0;

	public void MoveLeft()   {Position--;}
	public void MoveRight()  {Position++;}
	public void NextWeapon() {Weapon = (Weapon + 1) % 4;}
}
```

## The root of the problem: Cross systems dependency

Let's focus on render <-> character systems dependency.
Render generally is a different subsystem to characters/NPC/AI/etc, and it shouldn't really know about them much.
Character should contain the knowledge via code, components, etc how it should be rendered.
But now character doesn't know the current instance of renderer.

```{.csharp .number-lines}
public class Render
{
	// Usually renderers need some sort of graphics context initialization.
	// Which needs to be executed before we call any graphics functions.
	public GfxContext Context = ...;
	public Render() { ... }

	public void RenderSprite(Sprite sprite, int position)
	{
		...
	}
}

public class Character
{
	...

	public Sprite Sprite = ...;

	...

	public void OnRender()
	{
		???.RenderSprite(Sprite, ...);
	}
}
```

What are the options here?

# Getting to the solution

## Zero one infinity rule

As per [Zero one infinity rule](https://en.wikipedia.org/wiki/Zero_one_infinity_rule):

> The Zero one infinity (ZOI) rule is a rule of thumb in software design proposed by early computing pioneer Willem van der Poel.
>
> Allow none of foo, one of foo, or any number of foo.
>
> The only reasonable numbers are zero, one and infinity.
>
> — Bruce J. MacLennan

In practice allowing infinity of things is generally not useful, because you can quickly run out of memory, then what?
In that case stream-processing is usually used: Kafka, Hadoop, etc,
but it has non-zero overhead and is not practical for cases where you do fit in memory.

What we care here, is zero<->one part.

## The Singleton

> The singleton pattern is a software design pattern that restricts the instantiation of a class to one "single" instance.

```{.csharp .number-lines}
public class Render
{
	public GfxContext Context = ...;
	private Render() { ... } // constructor is private

	public static Render Instance { get; } = new Render();

	public void RenderSprite(Sprite sprite, int position)
	{
		...
	}
}

public class Character
{
	public Sprite Sprite = ...;

	public void OnRender()
	{
		Render.Instance.RenderSprite(Sprite, ...);
	}
}
```

Generally there is only one possible instance of renderer in the running application.
Hence a singleton fits nicely here.

Singleton can be considered an anti-pattern in some cases.

## Dependency injection

A different way is to use [Inversion of control](https://en.wikipedia.org/wiki/Inversion_of_control), with dependency injection being one of them:

```{.csharp .number-lines}
public class Render
{
	public GfxContext Context = ...;
	public Render() { ... }

	public void RenderSprite(Sprite sprite, int position)
	{
		...
	}
}

public class Character
{
	public Render CurrentRender = ...;
	public Sprite Sprite = ...;

	public Character(Render render)
	{
		CurrentRender = render;
	}

	public void OnRender()
	{
		CurrentRender.RenderSprite(Sprite, ...);
	}
}
```

Hard to manage if you need many systems.

What if current renderer changes?

Dependency injection can be considered an anti-pattern in some cases.
Wait, what? Both of them are bad!?

## One or two?

Sometimes one needs a test fixture.
But when shipping the application, one would only see `RealRender`.

```{.csharp .number-lines}
public abstract class IRender
{
	public static IRender Instance { get; } = ...; // How to chose what to create?

	public virtual void RenderSprite(Sprite sprite, int position) {}
}

public class RealRender : IRender
{
	public override void RenderSprite(Sprite sprite, int position)
	{
		...
	}
}

public class TestFixtureRender : IRender
{
	public override void RenderSprite(Sprite sprite, int position)
	{
		...
	}
}
```

Can be extended to `D3D12Render`, `VulkanRender`, etc.

## Dependency Injection and tests

Let's say we have a player profile and tavern store:

```{.csharp .number-lines}
// Can be an abstract class, or class with virtual methods, etc
public interface IPlayerProfile
{
	bool SpendGems(int gems);
}

public class PlayerProfile : IPlayerProfile
{
	...
}

public class TavernStoreManager
{
	private IPlayerProfile m_PlayerProfile;

	public TavernStoreManager(IPlayerProfile playerProfile)
	{
		m_PlayerProfile = playerProfile;
	}

	public void BuyItem(...)
	{
		m_PlayerProfile.SpendGems(...);
	}
}
```

## Make a test with a test fixture and dependency injection

```{.csharp .number-lines}
public interface IPlayerProfile
{
	bool SpendGems(...);
}

public class PlayerProfileMock : IPlayerProfile
{
	public bool SpendGemsWasCalled;
	public int TotalGemsSpent;
	bool SpendGems(int gems)
	{
		SpendGemsWasCalled = true;
		TotalGemsSpent += gems;
		return true;
	}
}

public class TavernStoreManager
{
	...
	public void BuyItem(...)
	{
		m_PlayerProfile.SpendGems(...);
	}
}

public class TavernStoreManagerTests
{
	[Test]
	public void BuyItem_CorrectlySpendsGems()
	{
		var mock = PlayerProfileMock();
		var tavern = new TavernStoreManager(mock);
		tavern.BuyItem(...);

		Assert.That(mock.SpendGemsWasCalled, Is.True); // think about order here
		Assert.That(mock.TotalGemsSpent,     Is.EqualTo(3));
	}
}
```

# Performance

## Sidetrack: hot path / cold path

- Hot codepath - code execution paths in which most of the execution time is spent.
- Cold codepath - everything else.

Sometimes one can hear "optimize the hot path".

## Performance

```{.csharp .number-lines}
public class IRender
{
	// One pointer indirection here
	public static Render Instance { get; } = ...;

	// A virtual call here, which is 1-2 pointer indirections
	public virtual void RenderSprite(Sprite sprite, int position) {}
}

public class RealRender : IRender
{
	public override void RenderSprite(Sprite sprite, int position)
	{
		...
	}
}

public class Character
{
	public Sprite Sprite = ...;

	public void OnRender() // Hot path
	{
		Render.Instance.RenderSprite(Sprite, ...);
	}
}
```

Now let's say we have 1 million of characters to render, we just added 2-3 pointer indirections in runtime for no good reason.
JIT could optimize them out though. Emphasis on _could_.

## Performance

```{.csharp .number-lines}
public static class Render
{
	private struct InstanceData
	{
		public GfxContext context; // should be value-type
	}

	// notice, this is a value-type
	// notice we group all static fields into one,
	// so we have one entry point to initialize/shutdown the subsystem
	private static InstanceData data = ...;

	// sometimes useful to ask to inline the method
	[MethodImpl(MethodImplOptions.AggressiveInlining)]
	public static void RenderSprite(Sprite sprite, int position)
	{
		// if data is null we're certain the system is not initialized
		// in contrast to just having bunch of static member fields
		// where it's hard to tell in which state the class is
	}
}

public class Character
{
	public Sprite Sprite;

	public void OnRender()
	{
		Render.RenderSprite(Sprite, 0);
	}
}
```

This is not a singleton as per the classical definition.
This is the same as original static renderer we discussed, the only difference is now all state is encapsulated in single static variable.
This way we can easily do reset/restart/etc.

# Gotchas

## Order of initialization, lazy initialization, cycling dependency

```{.csharp .number-lines}
public class SystemA
{
	private SystemA()
	{
		SystemB.Instance.Bar();
	}

	// lazy initialization
	public static SystemA Instance { get; } = new SystemA();

	public void Foo() { }
}

public class SystemB
{
	private SystemB()
	{
		// Oooops, cycling dependency
		SystemA.Instance.Foo();
	}

	public static SystemB Instance { get; } = new SystemB();

	public void Bar() { }
}
```

In which order do systems start here?

What if we have some external expectation about systems starting in specific order (audio, graphics, etc).

## Order of initialization, lazy initialization, cycling dependency

Generally production code tend to gravitate something like this:

```{.csharp .number-lines}
internal class SystemsManagerStartup {
	internal void Startup() {
		SystemA.Startup(); // Somewhere in code explicit initialization sequence is codified
		SystemB.Startup();
	}
}

public class SystemA
{
	public static SystemA Instance { get; private set; }
	internal static void Startup() {
		Instance = new SystemA();
	}

	private SystemA() {
		Debug.Assert(SystemB.Instance != null);
		SystemB.Instance.Bar();
	}

	public void Foo() { }
}

public class SystemB
{
	public static SystemB Instance { get; private set; }
	internal static void Startup() {
		Instance = new SystemB();
	}

	private SystemB() {
		Debug.Assert(SystemA.Instance != null);
		SystemA.Instance.Foo(); // Check with startup manager
	}

	public void Bar() { }
}
```

# Unity

## Singleton in Unity: via GameObject in the scene

```{.csharp .number-lines}
class Singleton : MonoBehaviour
{
	public static Singleton Instance { get; private set; }

	private void Awake()
	{
		if (Instance != null && Instance != this)
			Destroy(this.gameObject); // Mmm, but why we have two instances to begin with?
		else
			Instance = this;
	}
}
```

Means you need to create a GameObject in the scene manually and attach the script to it.

Beware of prefabs: if you have singletons inside prefabs, more than one prefab instance can be spawned.

The most famous problem is two instances of `EventSystem` historically rising the warning.

One way to tackle this is to have a prefab of game systems, and then manually control that you only have one instance of it in the scene.

## Singleton in Unity: self initializing

```{.csharp .number-lines}
#if UNITY_EDITOR
[UnityEditor.InitializeOnLoad]
#endif
class Singleton
{
	public static Singleton Instance { get; private set; }

	#if UNITY_EDITOR
	static AlwaysInitializedBehaviour() {
		Initialize();
	}
	#endif

	#if UNITY_STANDALONE
	// Gotcha, this is called after Awake!
	[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
	#endif
	static void Initialize()
	{
		Initialize()
		Instance = new Singleton();
	}
}
```

See [Unity docs on Execution Order](https://docs.unity3d.com/Manual/ExecutionOrder.html)

In worst case you could create a bootstrap scene where you create everything, and then jump to a real scene.

## Dependency injection in Unity: via serialization / editor

```{.csharp .number-lines}
public class Foo : MonoBehaviour
{
	public void DoFoo() { }
}

public class Bar : MonoBehaviour
{
	// serialized property, can select instance of Foo in the editor
	public Foo Foo;

	void Update()
	{
		Foo.DoFoo();
	}
}
```

## Dependency injection in Unity: via Zenject

[Zenject](https://github.com/modesttree/Zenject)

```{.csharp .number-lines}
public class Foo
{
	IBar _bar;

	public Foo(IBar bar)
	{
		_bar = bar;
	}
}

public interface IBar {}
public class Bar : IBar {}

public class TestInstaller : MonoInstaller
{
	public override void InstallBindings()
	{
		Container.Bind<Foo>().AsSingle();
		Container.Bind<IBar>().To<Bar>().AsSingle();
	}
}
```

# Self study

## Self study

- What are the main properties or a singleton?
- What are the main properties of dependency injection?
- What are pros/cons of a singleton?
- What are pros/cons of dependency injection?
- Why singletons and dependency injection might be slow?
- Is singleton a thread safe pattern?
- [Game Programming Patterns: Singleton](http://gameprogrammingpatterns.com/singleton.html)
- [Wiki: Singleton pattern](https://en.wikipedia.org/wiki/Singleton_pattern)
- [C2: Singleton pattern](http://wiki.c2.com/?SingletonPattern)

Exercise:
```
Player profile, UI and Potion

Implement as singleton and also as dependency injection:

- A player profile class that contains health value (0-100)
- A UI rendering GameObject with `void OnGUI()` and render a text label showing current amount of health
- A potion class that adds some health to the profile

Optional, try to implement a test that ensures potion actually does increment health value with com.unity.test-framework

public class PotionTests {
	[Test]
	public void Potion_CanIncrementHealth() {
		var profile = ...;
		var potion = ...;
		potion.Use();
		Assert.That(profile.health, Is. ...);
	}
}

```
