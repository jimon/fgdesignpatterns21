---
title: Double buffering
author: Dmytro Ivanov
date: Nov 2021
---

# Let's create a problem

## Two systems

```{.csharp .number-lines}
class SystemA {
	// Slow
	public int[] ProduceData() { ... }
}

class SystemB {
	// Slow
	public void ConsumeData(int[] data) { ... }
}

static class Game {
	static void Main() {
		var a = new SystemA();
		var b = new SystemB();
		while(true) {
			// This might take many seconds
			// But we also need to render some graphics, play audio, etc and keep a good FPS
			b.ConsumeData(a.ProduceData());
			RenderGame();
		}
	}
}
```


## Do a bit of work each frame

```{.csharp .number-lines}
class SystemA {
	// Fast, but takes multiple calls to finish the work. Returns true when done.
	public bool WorkForABit() { ... }
	// Returns null if not ready. If returns non-null, will also start new work batch.
	public int[] GetProducedDataAndStartNewWork() { ... }
}

class SystemB {
	// Returns true when done. Will keep returning true if no new data is set.
	public bool WorkForABit() { ... }
	public void SetDataToConsume(int[] data) { ... }
}

static class Game {
	static void Main() {
		var a = new SystemA();
		var b = new SystemB();
		while(run) {
			a.WorkForABit(); // Always run SystemA

			if(b.WorkForABit()) { // SystemB is done
				// Check if System A is ready
				var data = a.GetProducedDataAndStartNewWork();
				if (data != null) // Kick-off processing in SystemB
					b.SetDataToConsume(data);
			}

			RenderGame();
		}
	}
}
```

## Lets make a functioning prototype


```{.csharp .number-lines}
class SystemA {
	private int[] _data = new int[Count];
	private int _step, _generation;
	private const int Count = 3;

	public bool WorkForABit() {
		if (_step >= Count)
			return _step >= Count;
		Console.WriteLine($"SystemA generation {_generation} step {_step}");
		// _data[_step] = ...;
		_step++;
		return _step >= Count;
	}

	public (int[] data, int generation) GetProducedDataAndStartNewWork() {
		if (_step < Count)
			return (null, _generation);
		_step = 0;
		_generation++;
		// Make a copy here, otherwise both systems will be sharing same array
		var copy = new int[Count];
		Array.Copy(_data, copy, Count);
		return (copy, _generation - 1);
	}
}

class SystemB {
	private int[] _data;
	private int _step, _generation;

	public bool WorkForABit() {
		if (_data == null)
			return true;
		if (_step < _data.Length) {
			Console.WriteLine($"SystemB generation {_generation} step {_step}");
			// ... = _data[_step];
			_step++;
		}
		return _step >= _data.Length;
	}

	public void SetDataToConsume(int[] setData, int generation) {
		_data = setData;
		_step = 0;
		_generation = generation;
	}
}

static class Game {
	static void Main() {
		var a = new SystemA();
		var b = new SystemB();
		for(var i = 0; i < 10; ++i) {
			Console.WriteLine($"Frame {i}");
			a.WorkForABit(); // Always run SystemA

			if(b.WorkForABit()) { // SystemB is done
				// Check if System A is ready
				var data = a.GetProducedDataAndStartNewWork();
				if (data.data != null) // Kick-off processing in SystemB
					b.SetDataToConsume(data.data, data.generation);
			}
		}
	}
}
```

## And let's run it

```{.number-lines}
------------------------------Frame 0
SystemA generation 0 step 0
------------------------------Frame 1
SystemA generation 0 step 1
------------------------------Frame 2
SystemA generation 0 step 2
------------------------------Frame 3
SystemA generation 1 step 0 // <-- wait a sec, SystemA starts next batch in parallel?
SystemB generation 0 step 0 
------------------------------Frame 4
SystemA generation 1 step 1
SystemB generation 0 step 1
------------------------------Frame 5
SystemA generation 1 step 2
SystemB generation 0 step 2
------------------------------Frame 6
SystemA generation 2 step 0
SystemB generation 1 step 0
------------------------------Frame 7
SystemA generation 2 step 1
SystemB generation 1 step 1
------------------------------Frame 8
SystemA generation 2 step 2
SystemB generation 1 step 2
------------------------------Frame 9
SystemA generation 3 step 0
SystemB generation 2 step 0
```

## If SystemB is faster

If we make SystemB process two items in one step by changing to `_step += 2;`

```{.number-lines}
------------------------------Frame 0
SystemA generation 0 step 0
------------------------------Frame 1
SystemA generation 0 step 1
------------------------------Frame 2
SystemA generation 0 step 2
------------------------------Frame 3
SystemA generation 1 step 0
SystemB generation 0 step 0
------------------------------Frame 4
SystemA generation 1 step 1
SystemB generation 0 step 2 // SystemB is already one item ahead
------------------------------Frame 5
SystemA generation 1 step 2 // SystemB does nothing here, SystemB is "starving"
------------------------------Frame 6
SystemA generation 2 step 0
SystemB generation 1 step 0
------------------------------Frame 7
SystemA generation 2 step 1
SystemB generation 1 step 2
------------------------------Frame 8
SystemA generation 2 step 2
------------------------------Frame 9
SystemA generation 3 step 0
SystemB generation 2 step 0
```

## If SystemA is faster

If we make SystemA process two items in one step by changing to `_step += 2;`

```{.number-lines}
------------------------------Frame 0
SystemA generation 0 step 0
------------------------------Frame 1
SystemA generation 0 step 2
------------------------------Frame 2
SystemA generation 1 step 0
SystemB generation 0 step 0
------------------------------Frame 3
SystemA generation 1 step 2
SystemB generation 0 step 1
------------------------------Frame 4
SystemB generation 0 step 2 // SystemA does nothing here, SystemA is "starving"
------------------------------Frame 5
SystemA generation 2 step 0
SystemB generation 1 step 0
------------------------------Frame 6
SystemA generation 2 step 2
SystemB generation 1 step 1
------------------------------Frame 7
SystemB generation 1 step 2
------------------------------Frame 8
SystemA generation 3 step 0
SystemB generation 2 step 0
------------------------------Frame 9
SystemA generation 3 step 2
SystemB generation 2 step 1
```

## It's all threads

![](markdown/e0_double_buffer_threads.png)

(from [https://blog.zuru.tech/graphics/2020/04/23/renderinginue4](https://blog.zuru.tech/graphics/2020/04/23/renderinginue4))

## It's all CPU pipelines

![](markdown/e0_double_buffer_cpu.png)

(from [Wiki: Instruction pipelining](https://en.wikipedia.org/wiki/Instruction_pipelining))

# So what is double buffering?

## Double buffering

Because System B always waits for System A, both of them form a pipeline.
In this pipeline there can be only two possible copies of data,
one is "next data" in System A,
and another is "previous data" in System B.

```{.csharp .number-lines}
class SystemA {
	private int[] _data = new int[Count];
	...

	public bool WorkForABit() {
		...
		_data[_step] = ...;
		...
	}

	public int[] GetProducedDataAndStartNewWork() {
		...
		// Make a copy here, otherwise both systems will be sharing same array
		var copy = new int[Count];
		Array.Copy(_data, copy, Count);
		return (copy, _generation - 1);
	}
}

class SystemB {
	private int[] _data;
	...

	public bool WorkForABit() {
		...
		... = _data[_step];
		...
	}

	public void SetDataToConsume(int[] setData) {
		_data = setData;
		...
	}
}
```

Every time we do this, we need to do an allocation, free old data, etc. A lot of work for something that we know ahead of time.

## Double buffering

```{.csharp .number-lines}
class SystemA {
	private int[] _data1 = new int[Count];
	private int[] _data2 = new int[Count];
	// Front buffer holds data that is visible to "outside"
	private int[] _front;
	// Back buffer holds current working set
	private int[] _back;

	public SystemA()
	{
		// in C# arrays are references
		// in C++ this would be pointers
		_back = _data1;
		_front = _data2;
	}

	public bool WorkForABit() {
		...
		_back[_step] = ...;
		...
	}

	public int[] GetProducedDataAndStartNewWork() {
		// Swap _front and _back
		// Note we swap references, not contents
		var temp = _front;
		_front = _back;
		_back = temp;
		// Or can be written as (_front, _back) = (_back, _front);
		return _front;
	}
}
```

We have two buffers and we switch between them.

## Extra

Depending on length of a pipeline, we might need triple buffering or even more. A generalization is called multiple buffering.

In real life, double buffers often occur when we have pipelined (sub)processes running in parallel: threads, hardware, etc.
They are less often used in non-pipelined / single thread code.

# Double buffer origins

## How CRT works

![](markdown/e0_double_buffer_crt.png)

(from [https://www.cnblogs.com/shangdawei/p/4760933.html](https://www.cnblogs.com/shangdawei/p/4760933.html))

![](markdown/e0_double_buffer_scan_pattern.gif)

## How CRT works

![](markdown/e0_double_buffer_vga_connector.png)

(from [https://stimmer.github.io/DueVGA/](https://stimmer.github.io/DueVGA/))

## How would one render over VGA directly

![](markdown/e0_double_buffer_atari.png)

```{.csharp .number-lines}
while(verticalSyncSignal == false)
	DoNothing();
for(var y = 0; y < 37; ++y) // amount of CPU cycles has to be _exact_
	DoNothing();
for(var y = 0; y < 192; ++y) {
	for(var x = 0; x < 68; ++x)
		DoNothing();
	for(var x = 0; x < 160; ++x)
		OutputRegister = ...;
}
for(var y = 0; y < 30; ++y)
	DoNothing();
```

This technique is called [bit banging](https://en.wikipedia.org/wiki/Bit_banging).

(from [https://www.wired.com/2009/03/racing-the-beam/](https://www.wired.com/2009/03/racing-the-beam/))

## Before modern times

Because memory was expensive: Atari 2600 only had 128 bytes of RAM, not enough to store a full image.

Realtime graphics "rendering" was offloaded to hardware, so it can generate output image from a small amount of data.

This usually meant hardware was aware of background, sprites, players, etc.

Overview of [how gameboy works](https://www.gamedeveloper.com/programming/making-a-game-boy-game-in-2017-a-quot-sheep-it-up-quot-post-mortem-part-1-2-).

## Video buffer

As soon as memory chips became more cheap, we started allocating memory for "video buffer".
Finally we can render full picture with whatever we want.

![](markdown/e0_double_buffer_ramdac.png)

(from [http://www.miszalok.de/Textbook/IP03GraphicsCards/GraphicsCards.htm](http://www.miszalok.de/Textbook/IP03GraphicsCards/GraphicsCards.htm))

Process of moving bytes to the wire was encapsulated in "Random Access Memory + Digital To Analog convertor" chip (RAMDAC).

Notice, outputting data to monitor is not instantaneous, but takes approx 16.6ms at 60Hz.
GPU also takes time to render graphics and needs to output data somewhere.
If GPU outputs data to the same video memory as we currently read from, we will get [screen tearing](https://www.reinterpretcast.com/screen-tearing).


## Double buffering in GPU

![](markdown/e0_double_buffer_page_flip.gif)

(from [https://www.iitk.ac.in/esc101/05Aug/tutorial/extra/fullscreen/doublebuf.html](https://www.iitk.ac.in/esc101/05Aug/tutorial/extra/fullscreen/doublebuf.html))

That is also what [flip models](https://docs.microsoft.com/en-us/windows/win32/direct3ddxgi/dxgi-flip-model?redirectedfrom=MSDN) are all about.

# Self study

## Self study

- What problem double buffer is solving?
- What are the main properties of a double buffer.
- [Game Programming Patterns: Double buffer](http://gameprogrammingpatterns.com/double-buffer.html)
