---
title: Advanced Magic or Introduction to SMT Solvers / Theorem Provers
author: Dmytro Ivanov
date: Nov, 2021
---

# Whaaaaaa?

## What this even is?

TL;DR:

- Advanced Magic Super Powers.
- Tell computer to do work for you.
- Tell computer to prove program correctness for you.
- Low usage in game development, we need more!
- Even using tiny bits might allow for great things.

On a more serious note:

- Part of formal computer science: formal verification.
- General equation solvers(-ish). Or rather finding one solution to satisfy any given formula.
- Can also prove that things do work.
- A one step closer to the future where "code compiles" means "code works".

## How they work: Boolean satisfiability problem (SAT)

For any boolean equation of `x0 & x1 & !x2 ... | x3 & !x4 & x5 | ... = True`

Find a set of values (x0 ... xN) such as equation still holds true.

SAT is NP-complete problem .. so fairly hard.

## How they work: Satisfiability modulo theories (SMT) solver

For any mathematical equation `f(x0, x1, ..., xN) = True`.

Find a set of values (x0 ... xN) such as equation still holds true.

SMT is NP-hard or undecidable, meaning impossible to construct an algorithm to solve it.

## Z3

[Z3 Theorem Prover](https://en.wikipedia.org/wiki/Z3_Theorem_Prover) from Microsoft Research.

![](markdown/bonus_z3_overview.jpg)

## Example

```{.csharp .number-lines}
var c = new Context();
var s = c.MkSolver();

var x = c.MkBoolConst("x");
var y = c.MkBoolConst("y");

s.Assert( c.MkAnd(x, c.MkNot(y)) ); // x & !y == true

var result = s.Check();
if (result != Status.SATISFIABLE)
	return;

Console.WriteLine($"x = {s.Model.Eval(x, true)}"); // true
Console.WriteLine($"y = {s.Model.Eval(y, true)}"); // false
```

# Why?

## Sudoku solver

::: columns
:::: {.column width="60%"}

```{.csharp .number-lines}
var sudoku = new[,] {
	{ 5, 3, 0,  0, 7, 0,  0, 0, 0 }, // Zero means empty cell here.
	{ 6, 0, 0,  1, 9, 5,  0, 0, 0 },
	{ 0, 9, 8,  0, 0, 0,  0, 6, 0 },

	{ 8, 0, 0,  0, 6, 0,  0, 0, 3 },
	{ 4, 0, 0,  8, 0, 3,  0, 0, 1 },
	{ 7, 0, 0,  0, 2, 0,  0, 0, 6 },

	{ 0, 6, 0,  0, 0, 0,  2, 8, 0 },
	{ 0, 0, 0,  4, 1, 9,  0, 0, 5 },
	{ 0, 0, 0,  0, 8, 0,  0, 7, 9 },
};

public static class ArrayExtensions { // just helper methods for arrays
	public static Expr[] Column(this Expr[,] matrix, int columnNumber) {
		return Enumerable.Range(0, matrix.GetLength(0))
			.Select(x => matrix[x, columnNumber])
			.ToArray();
	}

	public static Expr[] Row(this Expr[,] matrix, int rowNumber) {
		return Enumerable.Range(0, matrix.GetLength(1))
			.Select(x => matrix[rowNumber, x])
			.ToArray();
	}

	public static Expr[] Square33(this Expr[,] matrix, int squareX, int squareY) {
		var r = new Expr[9];
		for (var y = 0; y < 3; ++y)
			for (var x = 0; x < 3; ++x)
				r[y * 3 + x] = matrix[y + squareY * 3, x + squareX * 3];
		return r;
	}
}
```

::::
:::: {.column width="40%"}

Let's solve this [Sudoku from Wiki](https://en.wikipedia.org/wiki/Sudoku_solving_algorithms#/media/File:Sudoku_Puzzle_by_L2G-20050714_standardized_layout.svg):

![](markdown/bonus_z3_sudoku.png)

::::
:::


## Sudoku solver

::: columns
:::: column

```{.csharp .number-lines}
var c = new Context();
var s = c.MkSolver();

var vars = new Expr[9, 9];

var k1 = c.MkInt(1);
var k9 = c.MkInt(9);

for (var y = 0; y < 9; ++y)
for (var x = 0; x < 9; ++x)
	if (sudoku[y, x] > 0)
		// Just load as constant if already known
		vars[y, x] = c.MkInt(sudoku[y, x]);
	else {
		var v = c.MkIntConst($"var_{x}_{y}");
		// Assert unknown value must be in [1, 9] range
		s.Assert(c.MkAnd(c.MkGe(v, k1), c.MkLe(v, k9))); // (v >= 1) && (v <= 9) == true
		vars[y, x] = v;
	}


// Assert every row has only distinct values
for (var y = 0; y < 9; ++y)
	s.Assert(c.MkDistinct(vars.Row(y)));

// Assert every column has only distinct values
for (var x = 0; x < 9; ++x)
	s.Assert(c.MkDistinct(vars.Column(x)));

// Assert every 3x3 square has only distinct values
for (var y = 0; y < 3; ++y)
	for (var x = 0; x < 3; ++x)
		s.Assert(c.MkDistinct(vars.Square33(x, y)));
```

::::
:::: column

```{.csharp .number-lines}
// Advanced Magic
var result = s.Check();
if (result != Status.SATISFIABLE)
	return;

for (var y = 0; y < 9; ++y)
{
	for (var x = 0; x < 9; ++x)
	{
		var v = s.Model.Eval(vars[y, x], true);
		Console.Write((x > 0 ? ", " : "") + v);
	}
	Console.WriteLine();
}
```

::::
:::

## Sudoku solver

The program prints:

```
5, 3, 4,  6, 7, 8,  9, 1, 2
6, 7, 2,  1, 9, 5,  3, 4, 8
1, 9, 8,  3, 4, 2,  5, 6, 7

8, 5, 9,  7, 6, 1,  4, 2, 3
4, 2, 6,  8, 5, 3,  7, 9, 1
7, 1, 3,  9, 2, 4,  8, 5, 6

9, 6, 1,  5, 3, 7,  2, 8, 4
2, 8, 7,  4, 1, 9,  6, 3, 5
3, 4, 5,  2, 8, 6,  1, 7, 9
```

What the heck even happened?

- Z3 has no idea how to solve Sudoku, or even what Sudoku is.
- Neither do we have any idea how to solve Sudoku.
- Here is a Sudoku solver purely based on rules of Sudoku itself.
- _Advanced_ Magic.

## Potential Gamedev usages

To start with:

- You can check if your generated puzzles are actually solvable without creating a solver.
- Implement suggestion system for puzzles.
- Verify that your puzzle-ish mechanics have solutions.

More advanced:

- Prove your card game doesn't have a dead lock situation. Would likely need temporal analysis also.
- Prove your puzzle mechanic is always solvable and doesn't have dead ends.
- Generate new puzzles
- Sanity check game economics

Engine usages:

- Tired of buggy multithreaded code even if you wrote hundreds of tests? Try TLA+.

# Takeaways

## Takeaways

- You might not find an immediate need for these tools, because game development industry hasn't embraced them _yet_.
- Remember about them next time you work on any sort of puzzle-ish game and have non-trivial questions.
- Science is cool! :)

Tools:

- [Z3 Theorem Prover](https://en.wikipedia.org/wiki/Z3_Theorem_Prover)
- [Coq](https://en.wikipedia.org/wiki/Coq)
- [TLA+](https://en.wikipedia.org/wiki/TLA%2B)

If you decide to try it out in your career, do poke me!
