---
title: Dirty flag
author: Dmytro Ivanov
date: Nov, 2021
---

# Let's start with the problem

## Problem definition

```{.csharp .number-lines}
class Barrel {
	Vector3 pos;
}

class Ship {
	Barrel[] barrels;
}

void MoveSomeBarrels(Ship ship) { // happens rarely
	ship.barrels[10].pos = ...;
}

void MoveSomeOtherBarrels(Ship ship) { // happens rarely
	ship.barrels[20].pos = ...;
}
```

## Problem definition

```{.csharp .number-lines}
class Barrel {
	Vector3 pos;
}

class Ship {
	Barrel[] barrels;
	float momentOfInertia;
}

void MoveSomeBarrels(Ship ship) {
	ship.barrels[10].pos = ...;
	ship.momentOfInertia = CalculateMomentOfInertia(ship); // <---
}

void MoveSomeOtherBarrels(Ship ship) {
	ship.barrels[20].pos = ...;
	ship.momentOfInertia = CalculateMomentOfInertia(ship); // <---
}

float CalculateMomentOfInertia(Ship ship) { ... }
```

## Problem definition

```{.csharp .number-lines}
class Barrel {
	Vector3 pos;
}

class Ship {
	Barrel[] barrels;
	float momentOfInertia;
}

void MoveSomeBarrels(Ship ship) {
	ship.barrels[10].pos = ...;
}

void MoveSomeOtherBarrels(Ship ship) {
	ship.barrels[20].pos = ...;
}

void PhysicsUpdate(Ship ship) {
	if(???)
		ship.momentOfInertia = CalculateMomentOfInertia(ship);

	...
}

float CalculateMomentOfInertia(Ship ship) { ... }
```

# "Is dirty" flag

## Just add the flag

```{.csharp .number-lines}
class Barrel {
	Vector3 pos;
}

class Ship {
	bool anyBarrelMoved; // dirty flag
	Barrel[] barrels;
	float momentOfInertia;
}

void MoveSomeBarrels(Ship ship) {
	ship.barrels[10].pos = ...;
	// Notice, this doesn't say "recalculate moment of inertia"
	// MoveSomeBarrels doesn't need any knowledge of inertia calculation,
	// which leads to a more cleaner separation of concerns and modular code
	ship.anyBarrelMoved = true;
}

void MoveSomeOtherBarrels(Ship ship) {
	ship.barrels[20].pos = ...;
	ship.anyBarrelMoved = true;
}

void PhysicsUpdate(Ship ship) {
	if(ship.anyBarrelMoved) { // <---
		ship.momentOfInertia = CalculateMomentOfInertia(ship);
		ship.anyBarrelMoved = false;
	}

	// This requires that either:
	// - moment of inertia only used here
	// - sequences of who is changing and who is reading is well defined
	// More on that later

	...
}

float CalculateMomentOfInertia(Ship ship) { ... }
```

## What if we have two object types

```{.csharp .number-lines}
class Barrel {
	Vector3 pos;
}

class Cannon {
	Vector3 pos;
}

class Ship {
	bool anyBarrelMoved; // <---
	bool anyCannonMoved; // <---
	Barrel[] barrels;
	Cannon[] cannons;
	float momentOfInertia;
}

void MoveSomeBarrels(Ship ship) {
	ship.barrels[10].pos = ...;
	ship.anyBarrelMoved = true;
}

void MoveSomeCannons(Ship ship) {
	ship.barrels[20].pos = ...;
	ship.anyCannonMoved = true;
}

void PhysicsUpdate(Ship ship) {
	if(ship.anyBarrelMoved || ship.anyCannonMoved) {
		ship.momentOfInertia = CalculateMomentOfInertia(ship);
		ship.anyBarrelMoved = false;
		ship.anyCannonMoved = false;
	}

	...
}

float CalculateMomentOfInertia(Ship ship) { ... }
```

## What if we have two object types

```{.csharp .number-lines}
class Barrel {
	Vector3 pos;
}

class Cannon {
	Vector3 pos;
}

class Ship {
	bool transformsDirty; // <---
	Barrel[] barrels;
	Cannon[] cannons;
	float momentOfInertia;
}

void MoveSomeBarrels(Ship ship) {
	ship.barrels[10].pos = ...;
	ship.transformsDirty = true;
}

void MoveSomeCannons(Ship ship) {
	ship.barrels[20].pos = ...;
	ship.transformsDirty = true;
}

void PhysicsUpdate(Ship ship) {
	if(ship.transformsDirty) {
		ship.momentOfInertia = CalculateMomentOfInertia(ship);
		ship.transformsDirty = false;
	}

	...
}

float CalculateMomentOfInertia(Ship ship) { ... }
```

## What if objects are grouped

::: columns

:::: column

```{.csharp .number-lines}
class Barrel {
	Vector3 pos;
}

class Cannon {
	Vector3 pos;
}

class ShipDeck {
	bool deckTransformsDirty;
	Barrel[] barrels;
	Cannon[] cannons;
	float weight;
}

class Ship {
	bool shipTransformsDirty;
	ShipDeck[] deck;
	float momentOfInertia;
}

void MoveSomeBarrels(Ship ship) {
	ship.deck[2].barrels[10].pos = ...;
	ship.deck[2].deckTransformsDirty = true;
	ship.shipTransformsDirty = true;
}

void MoveSomeCannons(Ship ship) {
	ship.deck[3].barrels[20].pos = ...;
	ship.deck[3].deckTransformsDirty = true;
	ship.shipTransformsDirty = true;
}
```

::::
:::: column

```{.csharp .number-lines}
void PhysicsUpdate(Ship ship) {
	if(ship.shipTransformsDirty) {
		ship.momentOfInertia = CalculateMomentOfInertia(ship);
		ship.shipTransformsDirty = false;
	}

	foreach(var deck in ship.decks)
		if (deck.deckTransformsDirty) {
			deck.weight = ...expensive_computation...;
			deck.deckTransformsDirty = false;
		}

	...
}

float CalculateMomentOfInertia(Ship ship) { ... }
```

::::
:::

## What if we need to compute something per object

::: columns

:::: column

```{.csharp .number-lines}
class Barrel {
	bool transformsDirty;
	Vector3 pos;
}

class Cannon {
	bool transformsDirty;
	Vector3 pos;
}

class ShipDeck {
	bool deckTransformsDirty;
	Barrel[] barrels;
	Cannon[] cannons;
	float weight;
}

class Ship {
	bool shipTransformsDirty;
	ShipDeck[] deck;
	float momentOfInertia;
}

void MoveSomeBarrels(Ship ship) {
	ship.deck[2].barrels[10].pos = ...;
	ship.deck[2].barrels[10].transformsDirty = true;
	ship.deck[2].deckTransformsDirty = true;
	ship.shipTransformsDirty = true;
}

void MoveSomeCannons(Ship ship) {
	ship.deck[3].barrels[20].pos = ...;
	ship.deck[3].barrels[20].transformsDirty = true;
	ship.deck[3].deckTransformsDirty = true;
	ship.shipTransformsDirty = true;
}
```

::::
:::: column

```{.csharp .number-lines}
void PhysicsUpdate(Ship ship) {
	if(ship.shipTransformsDirty) {
		ship.momentOfInertia = CalculateMomentOfInertia(ship);
		ship.shipTransformsDirty = false;
	}

	foreach(var deck in ship.decks) {
		if (deck.deckTransformsDirty) {
			deck.deckTransformsDirty = false;
			deck.weight = ...;
		}

		foreach(var barrel in deck.barrels) {
			if (barrel.transformsDirty) {
				...
				barrel.transformsDirty = false;
			}
		}

		foreach(var cannon in deck.cannons) {
			if (cannon.transformsDirty) {
				...
				cannon.transformsDirty = false;
			}
		}
	}


	...
}

float CalculateMomentOfInertia(Ship ship) { ... }
```

::::
:::

## Compute flag in runtime

::: columns

:::: column

```{.csharp .number-lines}
class Barrel {
	bool transformsDirty;
	Vector3 pos;
}

class Cannon {
	bool transformsDirty;
	Vector3 pos;
}

class ShipDeck {
	Barrel[] barrels;
	Cannon[] cannons;
	float weight;
}

class Ship {
	ShipDeck[] deck;
	float momentOfInertia;
}

void MoveSomeBarrels(Ship ship) {
	ship.deck[2].barrels[10].pos = ...;
	ship.deck[2].barrels[10].transformsDirty = true;
}

void MoveSomeCannons(Ship ship) {
	ship.deck[3].barrels[20].pos = ...;
	ship.deck[3].barrels[20].transformsDirty = true;
}
```

::::
:::: column

```{.csharp .number-lines}
void PhysicsUpdate(Ship ship) {
	bool shipTransformsDirty = false;

	foreach(var deck in ship.decks) {
		var deckTransformsDirty = false;

		foreach(var barrel in deck.barrels) {
			if (barrel.transformsDirty) {
				...
				barrel.transformsDirty = false;
				deckTransformsDirty = true;
			}
		}

		foreach(var cannon in deck.cannons) {
			if (cannon.transformsDirty) {
				...
				cannon.transformsDirty = false;
				deckTransformsDirty = true;
			}
		}

		if (deckTransformsDirty) {
			deck.weight = ...;
			shipTransformsDirty = true;
		}
	}

	if(shipTransformsDirty) {
		ship.momentOfInertia = CalculateMomentOfInertia(ship);
	}

	...
}

float CalculateMomentOfInertia(Ship ship) { ... }
```

::::
:::

## Granularity

Dirty flag can be per:

- Instance
- Group
- Group of groups
- etc
- System

## Dirty and properties

```{.csharp .number-lines}
class Barrel {
	Vector3 pos;
}

class Ship {
	bool anyBarrelMoved; // dirty flag
	Barrel[] barrels;

	private float _momentOfInertia;
	// now the property can be accessed at any point
	float momentOfInertia {
		get {
			if (anyBarrelMoved) {
				_momentOfInertia = CalculateMomentOfInertia(ship);
				anyBarrelMoved = false;
			}
			return _momentOfInertia;
		}
	}
}

void MoveSomeBarrels(Ship ship) {
	ship.barrels[10].pos = ...;
	ship.anyBarrelMoved = true;
}

void MoveSomeOtherBarrels(Ship ship) {
	ship.barrels[20].pos = ...;
	ship.anyBarrelMoved = true;
}

void PhysicsUpdate(Ship ship) {
	... ship.momentOfInertia ...;
}

float CalculateMomentOfInertia(Ship ship) { ... }
```

# Self study

## Self study

- What are the properties of dirty flag?
- What are the pros and cons of lazy evaluation?
- When dirty flags and lazy evaluation reduce performance?

- [Game Programming Patterns: Dirty Flag](http://gameprogrammingpatterns.com/dirty-flag.html)
- [Wiki: Lazy evaluation](https://en.wikipedia.org/wiki/Lazy_evaluation)
- [Wiki: Eager evaluation](https://en.wikipedia.org/wiki/Evaluation_strategy#Eager_evaluation)

Exercise:
```
A leader and a follower.

- Create two separate GameObjects (cubes, spheres, etc) and two new separate scripts, assign each game object it's script.
- In first script control GameObject position/rotation/etc with keys or mouse. This will be our leader.
- In second script add a reference to first GameObject (via scene editor or indirectly via `GameObject.Find`).
- In second script use `Transform.hasChanged` to listen when first object changed position, update second object position accordingly. Don't forget to clear the flag.
- Now try changing order of script execution in Unity so follower is executed before leader, observe noticeable one frame delay between follower and leader, while before that follower should be following leader exactly.
- Propose a few ways to solve it. Try using lazy evaluation for computing "current frame Transform of the leader" property.
```
