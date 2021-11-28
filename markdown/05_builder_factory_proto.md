---
title: Factory, Builder, Prototype
author: Dmytro Ivanov
date: Nov, 2021
---

# Let's start with the problem

## Classic in-game item store, problem one

```{.csharp .number-lines}
interface IPotion { ... }

class HealthPotion  : IPotion { ... }
class ManaPotion    : IPotion { ... }
class DefBuffPotion : IPotion { ... }

class GameStoreAndPotionManager
{
	public IPotion BuyPotion(??? %some_sort_of_store_id_from_ui% ???)
	{
		return ???
	}
}
```

We need some way to separate instance of a potion and the potion class itself

## Classic in-game item store, problem two

```{.csharp .number-lines}
class ManaPotion : IPotion
{
	public void Apply(PlayerProfile player)
	{
		player.Mana += HowMuchManaToAddWhenUsed * PotionLevel;
	}

	// This applies to all mana potion instances.
	// So it's a "property" of mana potion as a "type of potion".
	public const int HowMuchManaToAddWhenUsed = 100;

	// This applies to this concrete instance of mana potion.
	// So it's a "property" of particular instance of mana potion.
	public int PotionLevel;
}
```

- `HowMuchManaToAddWhenUsed` is a constant in code, how can we load it from a json or game editor?

# Modern solutions

## Create an enum that represents the type

```{.csharp .number-lines}
enum PotionType
{
	Invalid, // Could be useful if you do any sort of UI work

	Health,
	Mana
}

interface IPotion { ... }
class HealthPotion : IPotion { ... }
class ManaPotion : IPotion { ... }

class GameStoreAndPotionManager
{
	public IPotion BuyPotion(PotionType type)
	{
		switch (type)
		{
			case PotionType.Health:
				return new HealthPotion(???);
			case PotionType.Mana:
				return new ManaPotion(???);
			default:
				// or make it return null and be fail safe
				throw new InvalidEnumArgumentException();
		}
	}
}
```

## Create config static fields or struct

::: columns
:::: column

Move config to statics with-in the class:

```{.csharp .number-lines}
public class HealthPotion ... {
	// not cool that it's not-readonly here
	public static int HowMuchHealthToAddWhenUsed;

	public readonly int Level;

	public HealthPotion(int level) {
		Level = level;
	}

	public void Apply() {
		Player.Instance.Health += HowMuchHealthToAddWhenUsed * Level;
	}
}

public class GameStoreAndPotionManager {
	public IPotion BuyPotion(PotionType type, int level) {
		switch (type) {
			case PotionType.Health:
				return new HealthPotion(level);
			...
		}
	}

	public void LoadConfig() {
		HealthPotion.HowMuchHealthToAddWhenUsed = ...;
	}
}
```

::::
:::: column

Or move it out to a separate object:

```{.csharp .number-lines}
struct HealthPotionsConfig {
	public readonly int HowMuchHealthToAddWhenUsed;

	public HealthPotionsConfig(...) { ... }
}

public struct HealthPotion ... {
	public readonly int Level;

	public HealthPotion(int level) {
		Level = level;
	}

	public void Apply() {
		Player.Instance.Health +=
			GameStoreAndPotionManager
				.HealthPotionsConfig
				.HowMuchHealthToAddWhenUsed * Level;
	}
}

public class GameStoreAndPotionManager {
	public IPotion BuyPotion(PotionType type, int level) {
		...
	}

	public static HealthPotionsConfig HealthPotionsConfig { get; private set; }

	public void LoadConfig() {
		HealthPotionsConfig = new HealthPotionsConfig(...);
	}
}
```

::::
:::

## Types? Where we’re going, we don’t need types.

::: columns
:::: column

```{.csharp .number-lines}
enum PotionType {
	Invalid,
	Health,
	Mana
}

// Sum type / Tagged union
struct Potion {

	public readonly PotionType PotionType;

	public Potion(PotionType potionType) { ... }

	...

	public static void Apply(...) {
		switch(PotionType) {
		case PotionType.Health: ...; break;
		case PotionType.Mana:   ...; break;
		}
	}
}

struct GameStoreAndPotionManager {
	public static Potion BuyPotion(PotionType type) {
		return new Potion(type, ...);
	}
}
```

::::
:::: column

Games are on a spectrum from "every game element type is codified and present in the type system" to "everything is data driven and code is just a shell for the data".

Former allows for leveraging type system to do compile time computations, e.g. "can a health potion be used on an immortal player?" very well could be a compile time check, e.g. if you made a mistake in scripting your boss fight sequence it wouldn't compile due to type system safety.

Latter allows to separate concerns of making a potion game system, store game system, etc from concerns of designing and balancing the actual items, which in part enables a path to create tools for non-coding folks to do the game making directly (inventory editors, connections to game balancing tooling like [Machinations](https://machinations.io), etc).

::::
:::

# How did we got here

## Factory

Factory or Factory class is an act of encapsulating object creation and all related logic in one concrete location:

```{.csharp .number-lines}
class Potion ... { ... }

class PotionFactory
{
	public Potion Create(...)
	{
		return new Potion(...);
	}
}
```

Useful when construction of an object is bulky/unwieldy, e.g. tons of dependency injection, configuration, etc.

## Factory Method

Factory method, also called virtual constructor, is an act of decoupling interface and implementation of a creational method:

```{.csharp .number-lines}
class IItem ... { ... }

// Notice this is no longer a factory, rather it's a generic subsystem interface
// It might also be in a generic "inventory system" framework on asset store
abstract class IGenericGameSystem
{
	...

	// Factory method, e.g. a method that will create an item
	// but the invoking code doesn't know which method is called nor which item is created
	public abstract IItem CreateItem(...);
}

// ----------------------------------------------------------------------------

// This is specific to your particular game
class Potion : IItem { ... }
class PotionGameSystem : IGenericGameSystem
{
	public override IItem CreateItem(...)
	{
		return new Potion(...);
	}
}
```

## Abstract Factory

Abstract factory is an act of moving all factory methods into a separate interface:

```{.csharp .number-lines}
class IItem ... { ... }
class INpc  ... { ... }

// Notice concern of creating objects is fully encapsulated here
abstract class IGameFactory {
	public abstract IItem CreateItem(...);
	public abstract INpc  CreateNpc(...);
}

// Notice our generic system no longer contains knowledge how to create things
abstract class IGenericGameSystem {
	public IGameFactory Factory { get; private set; }
}

// ----------------------------------------------------------------------------

class BattleRoyalSpecificPotion : IItem { ... }
class BattleRoyalSpecificNpc    : INpc  { ... }
class BattleRoyalFactory : IGameFactory {
	public override IItem CreateItem(...) { ... }
	public override INpc  CreateNpc(...)  { ... }
}
class BattleRoyalGameSystem : IGenericGameSystem {
	...
}

// ----------------------------------------------------------------------------

class SandboxSpecificPotion : IItem { ... }
class SandboxSpecificNpc    : INpc  { ... }
class SandboxFactory : IGameFactory {
	public override IItem CreateItem(...) { ... }
	public override INpc  CreateNpc(...)  { ... }
}
class SandboxGameSystem : IGenericGameSystem {
	...
}
```

# Philosophical intermission

## Some reactions in the industry

![](markdown/05_builder_factory_proto_factory_meme_1.jpg)

## Some reactions in the industry

![](markdown/05_builder_factory_proto_factory_meme_2.jpg)

## Some reactions in the industry

![](markdown/05_builder_factory_proto_factory_meme_3.png)

## Occam's razor in 14th century

- "Plurality must never be posited without necessity" 14th century, probably the original
- "entities should not be multiplied beyond necessity" 16th century interpretation by John Punch
- "the simplest explanation is usually the best one"

![](markdown/05_builder_factory_proto_factory_occams_razor.jpg)

# Back to the code

## Factory can be in the object class itself

We can mend factory pattern in many different ways.

```{.csharp .number-lines}
public class BossNpcEffectManager : MonoBehaviour
{
	// as there is no constructor, we roll a generic Setup method instead
	internal void Setup(AnotherGameSystem depInjection, BossEffectType type, ...) { ... }

	// in-place/in-class factory
	public static BossNpcEffectManager AddBossNpcEffectManager(GameObject go, AnotherGameSystem depInjection, BossEffectType type, ...)
	{
		var manager = go.AddComponent<BossNpcEffectManager>();
		manager.Setup(depInjection, type, ...);
		return manager;
	}

	void Update() { ... }
}

public class GameManager : MonoBehaviour
{
	void SpawnNewBoss()
	{
		// or can be Instantiate a prefab
		var go = new GameObject(...);

		// meh
		BossNpcEffectManager.AddBossNpcEffectManager(go);

		...
	}
}
```

## Extension methods

Allows for advanced mending.

```{.csharp .number-lines}
public class BossNpcEffectManager : MonoBehaviour
{
	internal static void Setup(this BossNpcEffectManager manager, ...) { ... }

	void Update() { ... }
}

// I Can't Believe It's Not Factory!
public static class BossNpcEffectManagerExtensionMethods
{
	public static BossNpcEffectManager AddBossNpcEffectManager(this GameObject go, ...)
	{
		var manager = go.AddComponent<BossNpcEffectManager>();
		manager.Setup(...);
		return manager;
	}
}

public class GameManager : MonoBehaviour
{
	void SpawnNewBoss()
	{
		var go = new GameObject(...);

		// Uniform Function Call Syntax (UFCS)
		// also Rider will give you auto-suggestions when you type "go."
		go.AddBossNpcEffectManager(...);
	}
}
```

## Factory remarks

- Factory nowadays is anything that encapsulates object creation, e.g. typing `foo.CreateBar(...)` instead of `new Bar(...)`
- Use whatever is more natural / ergonomic in a given situation.
- Think a bit ahead, how the code would evolve in the future, e.g. new item types added, types removed, morphology changes, etc.

# Builder

## Modern Builder

Factory moves object construction to another place, but we still need to create the object somehow.

Builder provides a more flexible way to encapsulate different objects creation that have similar looking construction mechanisms.

Production examples from Unity input system:

::: columns
:::: column

```{.csharp .number-lines}
var ctrlMouseposition = new UnityEngine.InputSystem.Controls.Vector2Control();
ctrlMouseposition.Setup()
    .At(this, 0)
    .WithParent(parent)
    .WithChildren(13, 2)
    .WithName("position")
    .WithDisplayName("Position")
    .WithLayout(kVector2Layout)
    .WithUsages(0, 1)
    .DontReset(true)
    .WithStateBlock(new InputStateBlock
    {
        format = new FourCC(1447379762),
        byteOffset = 0,
        bitOffset = 0,
        sizeInBits = 64
    })
    #if UNITY_EDITOR
    .WithProcessor<InputProcessor<UnityEngine.Vector2>, UnityEngine.Vector2>(new UnityEngine.InputSystem.Processors.EditorWindowSpaceProcessor())
    #endif
    .Finish();
```

::::
:::: column

```{.csharp .number-lines}
var ctrlMousemiddleButton = new UnityEngine.InputSystem.Controls.ButtonControl();
ctrlMousemiddleButton.Setup()
    .At(this, 6)
    .WithParent(parent)
    .WithName("middleButton")
    .WithDisplayName("Middle Button")
    .WithShortDisplayName("MMB")
    .WithLayout(kButtonLayout)
    .IsButton(true)
    .WithStateBlock(new InputStateBlock
    {
        format = new FourCC(1112101920),
        byteOffset = 24,
        bitOffset = 2,
        sizeInBits = 1
    })
    .WithMinAndMax(0, 1)
    .Finish();
```

::::
:::

## Modern Builder via C# named and optional arguments

Named and optional arguments can be leveraged to achieve the subset of properties covered by builder pattern.

```{.csharp .number-lines}
class Item
{
	public Item(int requiredProperty, int optionalProperty1 = 0, int optionalProperty2 = 1)
	{
		...
	}
}

var item = new Item(
	requiredProperty: 1,
	// using default for optionalProperty1
	optionalProperty2: 2
);
```

# How did we got here

## Builder pattern accords, part 1

Builder generally is a container that can hold data necessary for construction.
And allows to split construction up into multiple steps.

```{.csharp .number-lines}
class Item {
	public Item(int property1, int property2, ...) { ... }
}

class ItemBuilder {
	private int _property1;
	private int _property2;

	public void SetProperty1(int property1) {_property1 = property1;}
	public void SetProperty2(int property2) {_property2 = property2;}

	public Item Build() {
		return new Item(_property1, _property2, ...);
	}
}

class SomeSystem {
	public Item CreateItem() {
		var builder = new ItemBuilder();
		builder.SetProperty1(...);
		builder.SetProperty2(...);
		...
		return builder.Build();
	}
}
```

## Builder pattern accords, part 2

Generalize the interface to achieve ability to construct different objects, while sharing setup code for generic properties.

::: columns
:::: column

```{.csharp .number-lines}
class GenericItem {
	public int GenericProperty { get; private set; }
}
abstract class GenericItemBuilder {
	private int _genericProperty;
	public void SetGenericProperty(int genericProperty) {_genericProperty = genericProperty;}
	public abstract GenericItem Build();
}

// --------------------------------------------------------------------------

class ConcreteItem1 : GenericItem {
	public int ConcreteProperty1 { get; private set; }
	public ConcreteItem1(int genericProperty, int concreteProperty1, ...) { ... }
}
class ConcreteItem1Builder : GenericItemBuilder {
	private int _concreteProperty1;
	public void SetConcreteProperty1(int concreteProperty1) {...}
	public override GenericItem Build() {
		return new ConcreteItem1(_genericProperty, _concreteProperty1, ...);
	}
}

// --------------------------------------------------------------------------

class ConcreteItem2 : GenericItem {
	public int ConcreteProperty2 { get; private set; }
	public ConcreteItem2(int genericProperty, int concreteProperty2, ...) { ... }
}
class ConcreteItem2Builder : GenericItemBuilder {
	private int _concreteProperty2;
	public void SetConcreteProperty2(int concreteProperty2) { ... }
	public override GenericItem Build() {
		return new ConcreteItem2(_genericProperty, _concreteProperty2, ...);
	}
}
```

::::
:::: column

```{.csharp .number-lines}
class SomeSystem {
	public GenericItem CreateItem1() {
		var builder = new ConcreteItem1Builder();
		builder.SetGenericProperty(...);
		builder.SetConcreteProperty1(...);
		...
		return builder.Build();
	}

	public GenericItem CreateItem2() {
		var builder = new ConcreteItem2Builder();
		// alternatively you can move common setup code into a separate method
		SetupGenericBuilder(builder);
		builder.SetConcreteProperty2(...);
		...
		return builder.Build();
	}

	private void SetupGenericBuilder(GenericItemBuilder builder) {
		builder.SetGenericProperty(...);
	}
}
```

::::
:::

## Builder pattern accords, part 3

[Wiki: Method chaining idiom](https://en.wikipedia.org/wiki/Method_chaining).

```{.csharp .number-lines}
class Item {
	public Item(int property1, int property2, ...) { ... }
}

class ItemBuilder {
	private int _property1;
	private int _property2;

	public ItemBuilder WithProperty1(int property1) {
		_property1 = property1;
		return this;
	}

	public ItemBuilder WithProperty2(int property2) {
		_property2 = property2;
		return this;
	}

	public Item Build() {
		return new Item(_property1, _property2, ...);
	}
}

class SomeSystem {
	public Item CreateItem() {
		return new ItemBuilder()
			.WithProperty1(...);
			.WithProperty2(...);
			...
			.Build();
	}
}
```

C++ STL folks love `<<` operator, some of STL is using method chaining:
```{.cpp .number-lines}
std::cout << "Some text " << 1234 << "Some other text";
```

## Builder remarks

- Builder is generally anything that allows you to split up object creation into multiple steps followed by `.Build()`, `.Finalize()`, `.Create()`, etc.
- Can also create and hold to sub-parts, e.g. when creating complicated composites, like attaching multiple sprites to one GameObject.
- Generally builders have limited lifespan, e.g. only for duration of creation.
- Hence it could be a zero cost abstraction in some cases if compiler/VM can inline everything.

# Prototype pattern

## Prototype pattern

Solves similar problem to a builder, but focuses on objects that mostly don't vary much during creation:

```{.csharp .number-lines}
public enum PotionType {
	Health,
	Mana
}

public class Potion {
	public PotionType PotionType;
	...
	public int Level;

	public Potion Clone() {
		// Note, if Potion is struct than MemberwiseClone will do a boxing allocation
		return (Potion)MemberwiseClone();
	}
}

class GameStoreAndPotionManager {
	private Potion HealthPotionStandard;
	private Potion ManaPotionStandard;
	...

	public GameStoreAndPotionManager() {
		HealthPotionStandard = new Potion ...;
		ManaPotionStandard = new Potion ...;
	}

	public Potion BuyPotion(PotionType type, int level)
	{
		switch (type)
		{
			case PotionType.Health:
				var potion = HealthPotionStandard.Clone();
				potion.Level = level;
				return potion;
			...
		}
	}
}

```

## Clone method

You might need to type it out by hand in a case of:

- You want a deep copy (remember, `MemberwiseClone` is shallow copy)
- You're programming in a language with less developed reflection such as C/C++
- You want to be more in control over performance

::: columns
:::: column

```{.csharp .number-lines}
using System.Runtime.CompilerServices;

public struct Foo {
	public int a;
	public int b;

	public Foo CloneSlow() {
		return (Foo)MemberwiseClone();
	}

	[MethodImpl(MethodImplOptions.AggressiveInlining)]
	public Foo CloneFast() {
		return new Foo {
			a = a,
			b = b
		};
	}
}
```

::::
:::: column

```{.asm .number-lines}
; Core CLR 6.0.21.52210 on amd64

Foo.CloneSlow()
    L0000: push rsi
    L0001: sub rsp, 0x20
    L0005: mov rsi, rcx
    L0008: mov rcx, 0x7ffcbc95ccc0
    L0012: call 0x00007ffd108cabf0
    L0017: mov rcx, [rsi]
    L001a: mov [rax+8], rcx
    L001e: mov rcx, rax
    L0021: call 0x00007ffcb0d40020
    L0026: mov rax, [rax+8]
    L002a: add rsp, 0x20
    L002e: pop rsi
    L002f: ret

Foo.CloneFast()
    L0000: push rax
    L0001: xor eax, eax
    L0003: mov [rsp], rax
    L0007: mov eax, [rcx]
    L0009: mov [rsp], eax
    L000c: mov eax, [rcx+4]
    L000f: mov [rsp+4], eax
    L0013: mov rax, [rsp]
    L0017: add rsp, 8
    L001b: ret
```

::::
:::

## Prototype remarks

- Prototype is generally anything that stores some "reference"/"standard" instance of an object and then clones/copies it to create a new instance.
- Some overlap with factory method + builder.
- Could be useful in scenarios where you just want to make a new copy with a slight change to it. For example in particle systems making dozen of clones of same particle emitter.

# Self study

## Self study

- Why we use creational patterns?
- What are the general properties of Factory?
- What is a Builder pattern and it's main properties?
- What are the main gotchas with Prototype pattern?

Optionally:

- What are the differences between Factory / Factory Method / Abstract Factory?
- How creational patterns can help in unit testing?
- [Wiki: Factory Method](https://en.wikipedia.org/wiki/Factory_method_pattern)
- [Wiki: Abstract Factory](https://en.wikipedia.org/wiki/Abstract_factory_pattern)
- [Wiki: Builder](https://en.wikipedia.org/wiki/Builder_pattern)
- [Wiki: Prototype](https://en.wikipedia.org/wiki/Prototype_pattern)
- [Wiki: Method chaining](https://en.wikipedia.org/wiki/Method_chaining)
- [Game Programming patterns: Prototype](http://gameprogrammingpatterns.com/prototype.html)
- "Design Patterns: Elements of Reusable Object-Oriented Software" (1994)

Exercise:
```
Add potions from UI, would be great if you do it in pairs/groups:

- Create a factory that can create potions based on an enum (e.g. public Potion Create(PotionType type) or similar)
- Implement it via switch-case
- Implement it via builder (you might still use switch-case), try method chaining
- Implement it via prototype, try MemberwiseClone and manual hand-written cloning

Optionally:
- Add a few UI buttons
- Connect button click to factory create via some callback/event/etc
- Add newly created potions to player profile
- Discuss how did it worked out for you, any gotchas on the way, and overall experience of implementing factory/builder/prototype.
```