---
title: Flyweight, Pool
author: Dmytro Ivanov
date: Nov, 2021
---

# Flyweight pattern origins, true story

## Flyweight pattern origins, true story

![](markdown/07_flyweight_pool_pointers_meme.jpg)

# Once upon a time making a game with a custom 2d engine

## Once upon a time making a game with a custom 2d engine, exhibit A

::: columns
:::: column

```{.csharp .number-lines}
public struct Color
{
	public float Red;
	public float Green;
	public float Blue;
	public float Alpha;

	public static Color RandomColor() { ... }
}

public class Texture
{
	public int Width;
	public int Height;
	public Color[] Pixels;
}

public class OpenGLRender
{
	public static void Render2DSprite(
		int positionX,
		int positionY,
		Texture texture,
		Color color)
	{
		...
	}

	public static Texture LoadPNG(string filename) { ... }
}
```

::::
:::: column

```{.csharp .number-lines}
public class GridCell
{
	public Texture CurrentBackground;
	public Color CurrentColor;
}

public class Grid
{
	public int Width;
	public int Height;
	public GridCell[] World;

	public Grid(int width, int height)
	{
		Width = width;
		Height = height;
		World = new GridCell[Width * Height];
		...
	}

	public void Render()
	{
		...
	}
}
```

::::
:::

## 2D array recap

![](markdown/07_flyweight_pool_2d_array_recap.jpg)

(From [here](http://www.assignmenthelp.net/questions/memory-layout-of-a-2d-array))

## Once upon a time making a game with a custom 2d engine, exhibit B

::: columns
:::: column

```{.csharp .number-lines}
public class Grid
{
	public int Width;
	public int Height;
	public GridCell[] World;

	public Grid(int width, int height)
	{
		Width = width;
		Height = height;
		World = new GridCell[Width * Height];

		var tileSet = new string[]
		{
			"tileset_earth.png",
			"tileset_water.png",
			"tileset_rocks.png",
			"tileset_forest.png"
		};

		for (var y = 0; y < Height; ++y)
		{
			for (var x = 0; x < Width; ++x)
			{
				var tile = Random.Shared.Next(tileSet.Length);

				var cell = new GridCell
				{
					CurrentBackground = OpenGLRender.LoadPNG(tileSet[tile]),
					CurrentColor = Color.RandomColor()
				};

				World[y * Width + x] = cell;
			}
		}
	}
```

::::
:::: column

```{.csharp .number-lines}
	public void Render()
	{
		for (var y = 0; y < Height; ++y)
		{
			for (var x = 0; x < Width; ++x)
			{
				var cell = World[y * Width + x];
				OpenGLRender.Render2DSprite(
					x,
					y,
					cell.CurrentBackground,
					cell.CurrentColor
				);
			}
		}
	}
}
```

::::
:::

## Once upon a time making a game with a custom 2d engine, exhibit B'

So you make 100x100 grid and start your game.

![](markdown/07_flyweight_pool_1_fps.jpg)

(From [here](https://www.youtube.com/watch?v=Ld1Xl4Fnn-k))

What happened?

# Flyweight pattern

## Flyweight pattern

::: columns
:::: column

```{.csharp .number-lines}
public class GridCell
{
	public int BackgroundTextureIndex;
	public Color CurrentColor;
}

public class Grid
{
	public int Width;
	public int Height;
	public GridCell[] World;
	public Texture[] TileSet; // <--

	public Grid(int width, int height)
	{
		Width = width;
		Height = height;
		World = new GridCell[Width * Height];

		TileSet = new Texture[4];
		var filenames = new[]
		{
			"tileset_earth.png",
			"tileset_water.png",
			"tileset_rocks.png",
			"tileset_forest.png"
		};
		for (var i = 0; i < TileSet.Length; ++i)
			TileSet[i] = OpenGLRender.LoadPNG(filenames[i]);

```

::::
:::: column

```{.csharp .number-lines}
		for (var y = 0; y < Height; ++y)
		{
			for (var x = 0; x < Width; ++x)
			{
				var tile = Random.Shared.Next(TileSet.Length);

				var cell = new GridCell
				{
					BackgroundTextureIndex = tile,
					CurrentColor = Color.RandomColor()
				};

				World[y * Width + x] = cell;
			}
		}
	}

	public void Render()
	{
		for (var y = 0; y < Height; ++y)
		{
			for (var x = 0; x < Width; ++x)
			{
				var cell = World[y * Width + x];
				OpenGLRender.Render2DSprite(
					x,
					y,
					TileSet[cell.BackgroundTextureIndex],
					cell.CurrentColor
				);
			}
		}
	}
}
```

::::
:::

## Intrinsic vs Extrinsic

From [Wiki](https://en.wikipedia.org/wiki/Intrinsic_and_extrinsic_properties):

> ... an intrinsic property is a property of a specified subject that exists itself or within the subject. An extrinsic property is not essential or inherent to the subject that is being characterized.

From grid cell perspective:

- Grid cell color - intrinsic, e.g. every cell might have a different color.
- Grid cell texture - extrinsic, e.g. the actual pixels forming the background picture are not an inherit property of a cell, unless you're making some sort of a game were every cell is unique.

From tileset texture perspective:

- Texture pixel colors - intrinsic
- Sprite color when rendering on a screen - extrinsic, e.g. same sprite can be rendered with many different colors.

Flyweight pattern is a lot about grouping/caching/sharing values with unique intrinsic properties together. Goal is to reduce amount of copies in memory.

## Flyweight pattern in GPU

Instanced rendering / instancing. If we have bunch of instances of same mesh in the scene, for every entity with same mesh:

- Entity transform, like position, etc is intrinsic.
- Entity animation state, e.g. skeletal animation, is intrinsic.
- Entity mesh geometry is extrinsic
- Entity mesh material is extrinsic

So you give the GPU: (oversimplifying!)

- Array of mesh geometries
- Array of mesh materials
- And then array of entities where each entity is transform, animation state, mesh index, material index.

This works because amount of unique props is usually magnitudes lower than amount of props instances. Think particles, 10 particle sprites might have 10+ millions of instances.

![](markdown/07_flyweight_pool_instancing.png)

(From [here](https://www.blend4web.com/doc/en/particles_instancing.html))

## Flyweight pattern remarks

- The term is rarely to almost never used in the industry (just like memento).
- Most game engines use it to some degree, in Unity all assets like materials, meshes, etc.
- C# might make it tricky to understand if multiple instances point to the same reference object or not. So you might want a more sturdy way to point to objects, like:

```{.csharp .number-lines}
public struct TextureHandle
{
	internal uint _internalRenderId;
	internal uint _generationCounter;
};
```

![](markdown/07_flyweight_pool_pointers_meme2.jpg)

A more general meaning of a pointer is an index into some storage.

Generally then it's called a reference, and accessing referring object is called dereferencing the reference.

Hence storing texture index in a cell is "storing texture reference in a cell", but that can be very confusing in C#.

## C# references remark

```{.csharp .number-lines}
public class Texture
{
	... potentially megs of pixels ...
}

public class GridCell
{
	public Texture CurrentBackground;
	public Color CurrentColor;
}

...

for (var i = 0; i < TileSet.Length; ++i)
	TileSet[i] = OpenGLRender.LoadPNG(filenames[i]);

for (var y = 0; y < Height; ++y)
{
	for (var x = 0; x < Width; ++x)
	{
		var tile = Random.Shared.Next(TileSet.Length);

		var cell = new GridCell
		{
			CurrentBackground = TileSet[tile], // works if Texture is reference type
			CurrentColor = Color.RandomColor()
		};
	}
}
```

# Object pool

## Once upon a time making, using GameObjects as particles ...

::: columns
:::: column

```{.csharp .number-lines}
public class ParticleSpawner : MonoBehaviour
{
	public GameObject ParticlePrefab;
	private float timer;

	void Update()
	{
		timer -= Time.deltaTime;
		if (timer < 0.0f) // first frame and every 100-500ms
		{
			for (var i = 0; i < 1000; ++i)
			{
				// YOOOOOOLOOOOOOOOO
				var go = Instantiate(ParticlePrefab);

				// random location
				go.transform.position = new Vector3(
					Random.Range(-10.0f, 10.0f),
					Random.Range(-10.0f, 10.0f),
					Random.Range(-10.0f, 10.0f));
			}

			timer = Random.Range(0.1f, 0.5f);
		}
	}
}
```

::::
:::: column

```{.csharp .number-lines}
public class TestParticle : MonoBehaviour
{
	void Start()
	{
		Destroy(gameObject, 1.0f); // we have a deadline huh
	}

	void Update()
	{
		transform.Rotate(0.0f, 360.0f * Time.deltaTime, 0.0f);
	}
}
```

::::
:::

## Not so YOLO anymore huh

`Instantiate` is really slow. Where is my `InstantiateAsync` huh?

![](markdown/07_flyweight_pool_particle_spawner_slow.jpg)

## Object pool

::: columns
:::: column

```{.csharp .number-lines}
public class ParticleSpawner : MonoBehaviour
{
	public GameObject ParticlePrefab;
	private float timer;

	private Queue<GameObject> freeGameObjectsPool; // can be Stack, or whatever else

	private void ObjectPoolWarmUp()
	{
		freeGameObjectsPool = new Queue<GameObject>();

		// this is optional, but a good idea
		for (var i = 0; i < 1000; ++i)
		{
			var go = Instantiate(ParticlePrefab);
			go.SetActive(false);
			freeGameObjectsPool.Enqueue(go);
		}
	}

	public GameObject ObjectPoolSpawn()
	{
		if (freeGameObjectsPool.TryDequeue(out var result))
		{
			result.SetActive(true);
			return result;
		}

		// of we can return null if our pool is "fixed" size
		return Instantiate(ParticlePrefab);
	}

	public void ObjectPoolReturn(GameObject go)
	{
		go.SetActive(false);
		freeGameObjectsPool.Enqueue(go);
	}
```

::::
:::: column


```{.csharp .number-lines}
	private void Start()
	{
		ObjectPoolWarmUp();
	}

	void Update()
	{
		timer -= Time.deltaTime;
		if (timer < 0.0f)
		{
			for (var i = 0; i < 1000; ++i)
			{
				var go = ObjectPoolSpawn();
				go.GetComponent<TestParticle>().ParticleStart(this);
				go.transform.position = new Vector3(
					Random.Range(-10.0f, 10.0f),
					Random.Range(-10.0f, 10.0f),
					Random.Range(-10.0f, 10.0f));
			}

			timer = Random.Range(0.1f, 0.5f);
		}
	}
}
```

::::
:::

## Particle needs to be adjusted to be "restartable"

```{.csharp .number-lines}
public class TestParticle : MonoBehaviour
{
	private float deadline;
	private ParticleSpawner currentSpawner;

	public void ParticleStart(ParticleSpawner spawner)
	{
		deadline = 1.0f;
		currentSpawner = spawner;
	}

	void Update()
	{
		deadline -= Time.deltaTime;
		if (deadline < 0.0)
			currentSpawner.ObjectPoolReturn(gameObject);

		transform.Rotate(0.0f, 360.0f * Time.deltaTime, 0.0f);
	}
}
```

## Getting there

Notice fewer spikes

![](markdown/07_flyweight_pool_particle_spawner_a_bit_faster.jpg)

## GameObject.SetActive is slow-ish, poke at MeshRenderer instead

```{.csharp .number-lines}
	private void ObjectPoolWarmUp()
	{
		freeGameObjectsPool = new Queue<GameObject>();

		for (var i = 0; i < 1000; ++i)
		{
			var go = Instantiate(ParticlePrefab);
			go.GetComponent<MeshRenderer>().enabled = false; // <---
			freeGameObjectsPool.Enqueue(go);
		}
	}

	public GameObject ObjectPoolSpawn()
	{
		if (freeGameObjectsPool.TryDequeue(out var result))
		{
			result.GetComponent<MeshRenderer>().enabled = true; // <---
			return result;
		}

		return Instantiate(ParticlePrefab);
	}

	public void ObjectPoolReturn(GameObject go)
	{
		go.GetComponent<MeshRenderer>().enabled = false; // <---
		freeGameObjectsPool.Enqueue(go);
	}
```

## Ship it

Almost no spikes

![](markdown/07_flyweight_pool_particle_spawner_fastish.jpg)

## Object pool remarks

- When `new MyObject` takes a lot of time, we amortize by having created, inactive objects in memory ready to be dispatched (used).
- When object is completed it purpose, it returns back to the pool of inactive objects.
- Pools might be grow-able or fixed size.
- [Unity provides object pool since 2021.1+](https://docs.unity3d.com/2021.1/Documentation/ScriptReference/Pool.ObjectPool_1.html)
- A need for an object pool might indicate shortcomings in the approach or in the engine. E.g. particles are usually not `GameObjects`, or loading meshes should be async, etc.

# Self study

## Self study

- How Flyweight works?
- How Object pool works?

- [Game Programming Patterns: Flyweight](http://gameprogrammingpatterns.com/flyweight.html)
- [Game Programming Patterns: Object Pool](http://gameprogrammingpatterns.com/object-pool.html)
- [Wiki: Object Pool](https://en.wikipedia.org/wiki/Object_pool_pattern)

Exercise:
```
In pairs/groups, implement a particle spawner where:

- Particles are GameObjects
- Instantiated via prefab
- Particles get "destroyed" after some time
- Implement naively via Instantiate, profile
- Implement object pool, profile, observe performance boost
```