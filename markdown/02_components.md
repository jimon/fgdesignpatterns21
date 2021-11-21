---
title: Composition, Composite, Components
author: Dmytro Ivanov
date: Nov, 2021
---

# Composition

## Composition

Broadly: combining objects or data into more complex objects.

Introduced in 1959 in COBOL as records, where record is a collection of related data, "Personal Record".

Originally meant just data composition:

```{.csharp .number-lines}
struct Player
{
	Vector2 position;
	Vector2 velocity;
	Sprite character;
}
```

Just a few years later in 1962 with Simula the concept developed into object/behavioral composition:

```{.csharp .number-lines}
struct Player
{
	Vector2 position;
	Vector2 velocity;
	Sprite character;
	Sprite weapon;

	void Jump()
	{
		if (IsTouching(character, Ground))
			velocity.y += 10;
	}

	void Render()
	{
		character.Render();
		weapon.Render();
	}
}
```

## Composition

Note composition is generally different from polymorphism:

```{.csharp .number-lines}
public class Shape
{
	Vector2 position;
	Vector2 size;

	public virtual void Draw() { }
}

public class Circle : Shape
{
	public override void Draw() { }
}
```

`Circle` is a subtype of `Shape` rather than a composition of `Shape` with something.

## Aggregation

Same as composition, but lifetime is detached now:

```{.cpp .number-lines}
struct Player
{
	Vector2 position;
	Vector2 velocity;
	Collider* collider;
	Sprite* character;
	Sprite* weapon;
	Animator* animatorController;
}
```

Because fields are pointers, lifetime of `Player` struct has no influence on lifetime of `Collider`, `Sprite`, etc.

## Composite pattern

Generally a composition of a group of objects that can be treated the same way as a single instance.

```{.csharp .number-lines}
interface IStoreItem
{
	void Buy();
	int GetPrice();
}

class CoinPack1000 : IStoreItem
{
	public void Buy() {  }
	public int GetPrice() { return 1; }
}

class GemPack10 : IStoreItem
{
	public void Buy() {  }
	public int GetPrice() { return 2; }
}

class CompositeStoreItem : IStoreItem
{
	private List<IStoreItem> _items;

	public void Add(IStoreItem item) { _items.Add(item); }

	public void Buy()
	{
		foreach (var item in _items)
			item.Buy();
	}

	public int GetPrice()
	{
		return _items.Select(x => x.GetPrice()).Sum();
	}
}
```

# Components

## Components

[Game Programming Patterns](http://gameprogrammingpatterns.com/component.html) describes components as:

> A single entity spans multiple domains. To keep the domains isolated, the code for each is placed in its own component class.
> The entity is reduced to a simple container of components.
>
> A restaurant menu is a good analogy. If each entity is a monolithic class, it’s like you can only order combos. We need to have a separate class for each possible combination of features. To satisfy every customer, we would need dozens of combos.
>
> Components are à la carte dining - each customer can select just the dishes they want, and the menu is a list of the dishes they can choose from.

Where an entity is any singular, identifiable, separate object.

## Without components

```{.csharp .number-lines}
class Player
{
	Vector2 position;
	Vector2 velocity;
	Sprite character;

	void Update(Physics physics, Renderer renderer, Input input)
	{
		if (input.GetDirection() == Input.Left)
			velocity = new Vector2(-1, 0);
		else if (input.GetDirection() == Input.Right)
			velocity = new Vector2(1, 0);
		else
			velocity = new Vector2(0, 0);

		position = physics.Move(position, velocity * Time.deltaTime);

		if (velocity.x >= 0)
			renderer.Draw(character, position);
		else
			renderer.Draw(character, position, flip: FlipAxis.X);
	}
}
```

## With components

::: columns

:::: column

```{.csharp .number-lines}
class PhysicsBodyComponent
{
	Vector2 position;
	Vector2 velocity;
	void Update(Physics physicsWorld)
	{
		position = physicsWorld.Move(position, velocity * Time.deltaTime);
	}
}

class CharacterInputComponent
{
	void Update(PhysicsBodyComponent physics, Input input)
	{
		if (input.GetDirection() == Input.Left)
			physics.velocity = new Vector2(-1, 0);
		else if (input.GetDirection() == Input.Right)
			physics.velocity = new Vector2(1, 0);
		else
			physics.velocity = new Vector2(0, 0);
	}
}
```

::::

:::: column


```{.csharp .number-lines}
class CharacterSpriteComponent
{
	Sprite character;
	void Update(PhysicsBodyComponent physics, Renderer renderer)
	{
		if (physics.velocity.x >= 0)
			renderer.Draw(character, physics.position);
		else
			renderer.Draw(character, physics.position, flip: FlipAxis.X);
	}
}

class Player
{
	PhysicsBodyComponent physicsComponent;
	CharacterInputComponent inputComponent;
	CharacterSpriteComponent spriteComponent;

	void Update(Physics physics, Renderer renderer, Input input)
	{
		inputComponent.Update(physicsComponent, input);
		physicsComponent.Update(physics);
		spriteComponent.Update(physicsComponent, renderer);
	}
}
```

::::

:::

This is almost exactly what Unity does:

- Instead of entity-class it uses `GameObject`, which is sealed so you can't extend it, hence generally there is no `class Player : GameObject`, rather it's just a generic component bag.
- Most scripting components are derived from `MonoBehaviour`
- Managed in runtime with `AddComponent` / `GetComponent` / `Destroy`
- Heavily relies on serialization of `GameObject` and `MonoBehaviour` to enable editor scene workflows.

## Component bag

Every component bag needs to answer a crucial, architecture deciding property: is it possible to have more than one component of same type?

```{.csharp .number-lines}
interface IComponent
{
	void Update();
}

class Entity
{
	IComponent[] components;                 // option 1
	Dictionary<Type, IComponent> components; // option 2

	void Update()
	{
		foreach (var component in components)
			component.Update();
	}
}
```

Every Entity-Component system will fall into one of this two options.
Unity is option 1 but most API's makes you feel like option 2 like `GetComponent<T>()`.

## Coupling between components

Now everything is decoupled, but input component needs to know about physics component, how is that done?

- In some engines entity is implemented as just a struct with pointers, like prior aggregation example. Then every component can know about the parent entity.
- In Unity every `MonoBehaviour` knows parent `GameObject` and can get any component via `GetComponent<T>()`.
  There are a few gotchas, how do we know that input is updated before physics? What if there is no physics component?
- Send message. Even more gotchas as this shifts problem from static typing to dynamic typing, and maybe to runtime if messages can be delayed.

```{.csharp .number-lines}
public class Example1 : MonoBehaviour
{
	void Start()
	{
		gameObject.SendMessage("ApplyDamage", 5.0);
	}
}

public class Example2 : MonoBehaviour
{
	public void ApplyDamage(float damage)
	{
		...
	}
}
```

Generally some level of coupling is unavoidable simply because in the end of the day we need to get the work done.

Choose coupling level based on separation of concerns and code reuse.

Separation of concerns might let you split your code into many classes, but it's hard to work with many of them, just adding them in Unity editor to the game object might be tedious. Facades might be a good idea here. E.g. some generic `GameStoreManager` that handles working with the rest of classes.

# ECS

## ECS?

Ok, entities, components, what is ECS then? ECS adds one more: Entity-Component-System.

::: columns

:::: column

```{.csharp .number-lines}
interface IComponent { }

struct PhysicsBodyComponent : IComponent {
	Vector2 position;
	Vector2 velocity;
}

// tag component
struct CharacterInputComponent : IComponent {}

struct CharacterSpriteComponent : IComponent {
	Sprite character;
}

struct Entity {
	int physicsBodyIndex;
	int characterInputIndex;
	int characterSpriteIndex;
}

struct World {
	PhysicsBodyComponent[] physicsBodies;
	CharacterInputComponent[] characterInputs;
	CharacterSpriteComponent[] characterSprites;
}
```

Systems are important because processors are Single Instruction Multiple Data (SIMD).
Tightly iterating over permutation of components and executing same operation over them enables more performant code.
Versus in normal E-C approach where you need to pay many costs to invoke update for every component (virtual call, cache miss, etc).

Another aspect is how do you store data of your components: AoS vs SoA vs AoSoA.
While this particular design pattern is independent from storage architecture,
most ECS frameworks focus primarily on efficient memory layouts.

::::

:::: column

```{.csharp .number-lines}
void PhysicsBodySystem(Physics physicsWorld) {
	foreach(PhysicsBodyComponent component in
		world.GetAllEntitiesSuchAs<PhysicsBodyComponent>())
		component.position = physicsWorld.Move(component.position,
			component.velocity * Time.deltaTime);
}

void CharacterInputSystem(Input input) {
	foreach((PhysicsBodyComponent physics, CharacterInputComponent _) in
		// Note this query is non-trivial
		world.GetAllEntitiesSuchAs<PhysicsBodyComponent, CharacterInputComponent>()) {
		if (input.GetDirection() == Input.Left)
			physics.velocity = new Vector2(-1, 0);
		else if (input.GetDirection() == Input.Right)
			physics.velocity = new Vector2(1, 0);
		else
			physics.velocity = new Vector2(0, 0);
	}
}

void CharacterSpriteSystem(Renderer renderer) {
	foreach((PhysicsBodyComponent physics, CharacterSpriteComponent sprite) in
		world.GetAllEntitiesSuchAs<PhysicsBodyComponent, CharacterSpriteComponent>())
		if (physics.velocity.x >= 0)
			renderer.Draw(sprite.character, physics.position);
		else
			renderer.Draw(sprite.character, physics.position, flip: FlipAxis.X);
}

void UpdateAll(...) {
	PhysicsBodySystem(...);
	CharacterInputSystem(...);
	CharacterSpriteSystem(...);
}
```

::::

:::

## AoS vs SoA vs AoSoA


```{.csharp .number-lines}
// ------------------------------------------------------
// Array of Structs
// In memory you will have one stream of everything
struct Player
{
	Vector3 position;
};

struct WorldAoS
{
	Player[] players;
};

// ------------------------------------------------------
// Struct of Arrays
// In memory you will have one stream per component
struct WorldSoA
{
	float[] PlayerX;
	float[] PlayerY;
	float[] PlayerZ;
}

// ------------------------------------------------------
// Arrays of Structs of Arrays
// In memory you will have one stream per component
struct WorldSoA
{
	unsafe fixed float PlayerX[64];
	unsafe fixed float PlayerY[64];
	unsafe fixed float PlayerZ[64];
}

struct WorldAoSoA
{
	WorldSoA[] arrays;
}
```

## No magic

There is no fundamental complexity or magic to components, as a pattern it's very straightforward.

If you want you can even build your own EC/ECS framework and inject it to Unity via `UnityEngine.LowLevel.PlayerLoop.SetPlayerLoop(...)`.

But the rest of ecosystem needs to use the same API's, so to make editor, tools, etc to work nicely together.

Hence having only one framework makes a lot of sense.

# Self study

## Self study

- What is composition?
- What is aggregation? Hows it's different from composition?
- What are main properties of composite pattern?
- What are components? And what are their main properties?
- What are the main complexities in EC and ECS?

- [Wiki: Composite pattern](https://en.wikipedia.org/wiki/Composite_pattern)
- [Game Programming Patterns: Component](http://gameprogrammingpatterns.com/component.html)

Extra:

- [https://en.wikipedia.org/wiki/Object_composition](https://en.wikipedia.org/wiki/Object_composition)
- [Sum types](https://chadaustin.me/2015/07/sum-types/) / [https://en.wikipedia.org/wiki/Tagged_union](https://en.wikipedia.org/wiki/Tagged_union)

Exercise:
```
Implement a character game object where:

- Movement is controlled by it's own separate component script
- Rendering (setting of armor/weapon/etc models) is controlled by separate script
- Sound and other effects are controlled by component script

Optionally:
- Add an extra GameObject responsible for "game character editor", e.g. something global
- Add some generic UI buttons to it (OnGUI is ok) to change armor/weapon/etc or trigger effects, all by invoking methods on your own script components, rather than setting models from UI code.
- Try also triggering sounds from movement (calling one component from another component)
- Try doing the same but via Unity messages
```