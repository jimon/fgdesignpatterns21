---
title: Introduction to Software Design Patterns
author: Dmytro Ivanov
date: Nov, 2021
---

# Intro

## Little about me

Dmytro

Background:

- BS in Applied mathematics
- C/C++/C#/Python/Lua
- Engine / Systems / Low level

Work:

- Unity
- Spotify
- King

Interests:

- Electronics
- Sports: cycling, snowboarding, kayaking, etc
- A bit of flying

## Course Goals

Go from "We don't know what we don't know" to "We know what we don't know".

Learn new tools to solve problems.

Understand the properties and the context behind the tools.

Have fun :)

## Course Structure

From Monday to Thursday inclusive:

- One or two lectures 10:30 - 12:30, small break in between.
- Mentoring / self study 13:30 - 15:30.
- Allocate some time for graded assignment, when you use a pattern for the first time it's likely to take a few tries.
- On the last day of the course, let's make "show & tell" if you want, poke me if you're interested in presenting.

In general:

- If you have questions, stop me at any time.
- Focus on properties, not names. "I need something with properties X, Y, Z there was a pattern just like that somewhere" is a very good outcome.
- Some patterns are less distinct by nature, don't worry about it.
- Good to discuss in groups to get different perspectives.

All lectures will be available later on [Github pages](https://jimon.github.io/fgdesignpatterns21/) or [Github repo](https://github.com/jimon/fgdesignpatterns21).

## Graded assignment

### What

Make a game, where features are implement with help of patterns.

Feature suggestion:

- Skill tree
- Inventory
- Store
- Crafting
- Power-ups / Buffs / etc
- Combos
- Notifications
- Achievements
- Interactive storytelling
- In-game editor

Optionally, could be interesting to try to implement it as no-coding-required game systems. Meaning someone with no coding skills could use your crafting system to implement new crafting rules in your game.

### Grading

- For Godkänt: Use minimum of 3 patterns in a context of game features.
- For Väl godkänt: Use minimum of 5 patterns in a context of game features.

## Graded assignment details

### How

- Use C# and Unity.
  - Good idea to use 2020.3.x or 2021.2.x.
- Add a readme.txt (or readme.md) in the root folder and write down:
  - Your name, what is the game about
  - List of patterns used in the game, where they used (file name, class or method name), and a short explanation how they were used in the game.

```
Dmytro Ivanov
A simple platform game with potions, item store and inventory.

Patterns:
- Singleton, in `InventoryManager.cs` in `class InventoryManager` as `InventoryManager.instance`
  Used singleton to make the single instance of the class that could be accessed from anywhere.
- ...
```

### Submission

- Upload your project or provide a URL to the repository.
- If you plan to upload, would be nice if you only archive the following folders and their contents: "Assets", "Package", "ProjectSettings", and "readme.txt" file.

### Tips

- Good idea to try refactoring tools in Rider / Visual Studio.
- Good idea to also use version control! :)

## What is a software design pattern

Per [Wiki](https://en.wikipedia.org/wiki/Software_design_pattern):

> A software design pattern is a general, reusable solution to a commonly occurring problem within a given context in software design.

Per [A Pattern Language](https://en.wikipedia.org/wiki/A_Pattern_Language) by Christopher Alexander (1977)

> Each pattern describes a problem which occurs over and over again in our environment
> and then describes the core of the solution to that problem,
> in such a way that you can use this solution a million times over,
> without ever doing it in the same way twice.

Closely related concept: Programming idiom.

Per [c2 wiki](http://wiki.c2.com/?ProgrammingIdiom):

> An idiom is a phrase that doesn't make literal sense, but makes sense once you're acquainted with the culture in which it arose.

## What is a software design pattern

Loosely grouped together:

- Problem statement
- Desired/expected properties
- "The key trick"
- Common implementations

More of an art with happy little accidents.

## Historical context

- [Lightweight genealogical tree](http://rigaux.org/language-study/diagram-light.png)
- [The principal programming paradigms](https://info.ucl.ac.be/~pvr/paradigmsDIAGRAMeng108.pdf)

Remark regarding patterns and DSL's.

## Design Patterns usage today

I think patterns got a secondary role today, and data flows / performance become primary concerns.

We still use patterns to:

- Easier to split larger problem into smaller ones with known/standard solutions.
- Communicate with other devs via a more higher level language (concepts instead of code).

Hence the architecture of modern application is more than just sum of design patterns.

In daily work, most folks don't use the names of patterns but rather talk about the properties (nobody is talking about memento but everyone talks about serialization).

## New code vs old code

When writing new code, patterns can help speed up design phase.

When refactoring old code, patterns can help standardize code.

## Patterns and anti-patterns

Every pattern might have positive properties in one context and negative properties in another.

## Overview

Most patterns can be categorized into 4 types:

- Creational
- Structural
- Behavioral
- Concurrency

## Creational patterns

"Create an object for given data":

- Builder
- Factory method
- Abstract factory
- Prototype

"Manage object lifetime and initialization order for me":

- Singleton
- Multiton
- Dependency injection
- Resource acquisition is initialization (RAII)
- Lazy initialization
- Object pool

## Structural patterns

"I have these two things and I need them to fit together":

- Adapter / Wrapper
- Bridge
- Decorator
- Proxy

"I need a clean nice API to use":

- Facade
- Front controller

Others:

- Composite
- Flyweight
- Marker
- Module

## Behavioral patterns

"I need to couple systems together":

- Chain of responsibility
- Mediator
- Iterator
- Visitor
- Template method
- Observer

"I need to represent state/objects/operations independent of particular instances":

- Command
- Interpreter
- Memento
- Specification
- Strategy
- State

"I need to encode behaviors when there is no object":

- Null object

## Idioms

We will also cover a few gamedev specific idioms:

- Double buffer
- Dirty flag
- RAII
- Event queue

And spend a bit of time talking about [SOLID](https://en.wikipedia.org/wiki/SOLID) and others.

## Concurrency patterns

Not covered in this class :(

If you want to venture there, my advise would be to try bottom-up approach:

What are we even trying to do:

- Context switch, CPU interrupts. There are no threads in hardware, it's all illusion.
- "Memory Barriers: a Hardware View for Software Hackers" by Paul E. McKenney, 2009
- Leslie Lamport works: Sequential consistency, others.
- Parallel computing basics, map-reduce, SIMD/MIMD.
- Amdahl's law and friends.
- Bonus: Virtual memory, page table, translation lookaside buffer (TLB). There is no memory allocation in hardware, it's all illusion.

The tools we have as software programmers:

- Threads.
- Semaphore, spinlock, read-write lock, etc.
- Condition variables, futex.
- Atomics and memory barriers.
- Read-copy-update.

# Self study

## Self study

- What are the main categories of design patterns?
- What are the broader areas of responsibility in each category?
- [Wiki: Software design pattern](https://en.wikipedia.org/wiki/Software_design_pattern)

Bonus:

- Lookup source code of [VVVVVV](https://github.com/TerryCavanagh/VVVVVV)