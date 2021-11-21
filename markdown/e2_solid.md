---
title: SOLID and other principles
author: Dmytro Ivanov
date: Nov, 2021
---

# Historical context

## Historical context

Transition from Waterfall model to Agile software development brought many different things:

- Backlogs
- Pair Programming
- Code Refactoring
- Extreme programming
- Kanban
- Scrum

# Common principles

## SOLID

From [Wiki](https://en.wikipedia.org/wiki/SOLID):

- The Single-responsibility principle: "There should never be more than one reason for a class to change."
- The Open–closed principle: "Software entities ... should be open for extension, but closed for modification."
- The Liskov substitution principle: "Functions that use pointers or references to base classes must be able to use objects of derived classes without knowing it."
- The Interface segregation principle: "Many client-specific interfaces are better than one general-purpose interface."
- The Dependency inversion principle: "Depend upon abstractions, [not] concretions."

## DRY / WET / AHA

- Don't Repeat Yourself.
- Write Everything Twice.
- Avoid Hasty Abstractions.

## KISS

- Keep it simple, stupid.

## YAGNI

- You aren't gonna need it.

Inside joke: YAGNI programming principles.

## Defensive programming

- Making the software behave in a predictable manner despite unexpected inputs or user actions.

## Fail-fast

- Fail-fast systems are usually designed to stop normal operation rather than attempt to continue a possibly flawed process.

Example: Switch SDK.

## Separation of concerns

- Split program into parts/modules.
- Every part covers a specific concern.

# Principles can be an anti-pattern

## Reality check

- Ivory tower development.
- Does it solve users problem?
- Does it create users value?
- Is it an effective solution?

# Closing remarks

## Zen Of Python

- Beautiful is better than ugly.
- Explicit is better than implicit.
- Simple is better than complex.
- Complex is better than complicated.
- Flat is better than nested.
- Sparse is better than dense.
- Readability counts.
- Special cases aren't special enough to break the rules.
- Although practicality beats purity.

## Zen Of Python

- Errors should never pass silently.
- Unless explicitly silenced.
- In the face of ambiguity, refuse the temptation to guess.
- There should be one - and preferably only one - obvious way to do it.
- Although that way may not be obvious at first unless you're Dutch.
- Now is better than never.
- Although never is often better than right now.
- If the implementation is hard to explain, it's a bad idea.
- If the implementation is easy to explain, it may be a good idea.
- Namespaces are one honking great idea – let's do more of those!

# Self study

## Self study

- What each SOLID concept mean on day-to-day level in terms of writing/refactoring code?

- [Wiki: Inversion of control](https://en.wikipedia.org/wiki/Inversion_of_control)
- [Wiki: Pareto principle](https://en.wikipedia.org/wiki/Pareto_principle)
- [Wiki: Ninety–ninety rule](https://en.wikipedia.org/wiki/Ninety%E2%80%93ninety_rule)