---
title: Event queue
author: Dmytro Ivanov
date: Nov, 2021
---

# Let's dive straight into it

## Events vs Messages

The basic implementation primitive of pull mode/message driven code. More proper name is "message queue".

Events vs Messages are often confused and used interchangeably:

- Generally when we say "we listen to events" we mean getting our callback called instantaneously when event actually happens.
- Generally when we say "we process messages" we imply pulling some sort of message queue.

More broadly:

- Events are a snapshot of history (e.g. "button was pressed"). They can be instantaneous/delayed/replayed/stored/etc (e.g. log file). Producers of events don't have any idea about consumers of events.
- Messages are like mail, they typically have a concrete destination.

Nowadays in game development both terms are kinda used interchangeably.
In cloud you might see them differ somewhat, e.g. Publish–subscribe pattern.

## Event queue

```{.csharp .number-lines}
public class Event {
	...
}

public class Example {
	private Queue<Event> EventQueue; // First-In-First-Out data structure (FIFO)

	// Event Producer, could be more than one, could be multiple threads
	public void SomethingHappenedCallback(...) {
		var newEvent = new Event {
			...
		};

		// Could grow event queue infinitely, or start dropping old events if we have too many already.
		EventQueue.Enqueue(newEvent);
	}

	// Event Consumer, could be more than one, could be multiple threads
	public void OnUpdate() {

		// Could process all events at once, or limit about events per frame
		while (EventQueue.TryDequeue(out var currentEvent)) {
			... process event ...

			// We could also generate new events here, need to be able to exit from the loop though.
		}
	}
}
```

## More practical example

We want to start playing some audio but callback is called on a thread.

```{.csharp .number-lines}
public class MyApp {
	private Queue<MyEvent> EventQueue;

	// Called on some random thread
	private void OnInStorePurchaseComplete(...) {
		// can't call Audio.Play(...) here
		EventQueue.Enqueue(new Event {
			Type = EventType.OnInStorePurchaseComplete, // some enum
			Payload = ...
		});
	}

	// Called on main thread
	private void OnUpdate() {
		while (EventQueue.TryDequeue(out var currentEvent)) {

			switch (currentEvent.Type) {
				case EventType.OnInStorePurchaseComplete:
					Audio.Play(...);
					break;
				...
			}

		}
	}
}
```

## Input event queue

```{.csharp .number-lines}
public enum EventType {
	KeyDown,
	KeyUp,
	MouseMove,
	...
}

public class Event {
	public EventType Type;
	public KeyCode keyCode;
	public Vector2 mousePosition;
	...
}

public class MyApp {
	private Queue<Event> EventQueue;

	// This could arrive at any point, even during our rendering
	private void Windows_On_Key_Down(KeyCode keyCode) {
		var newEvent = new Event {
			Type = EventType.KeyDown,
			keyCode = keyCode
		};
		EventQueue.Enqueue(newEvent);
	}

	private void Windows_On_Key_Up(KeyCode keyCode) {...}

	private void Windows_On_Mouse_Move(Vector2 newPosition) {...}

	private void OnUpdate() {
		while (EventQueue.TryDequeue(out var currentEvent)) {
			switch (currentEvent.Type) {
				case EventType.KeyDown: ...; break;
				...
			}
		}
	}
}
```

## Infinite loops

You could find some system enqueuing events in response to events.
If events are handled instantaneously this might create infinite loops, potentially with stack overflows.

Placing events on a queue makes it such that, one system will only start processing events from another system in "next update",
where in game development that usually means next frame.

We still have an infinite loop, just spaced out in time.

## Queuing theory

::: columns
:::: column

Queues are like buckets, with inflow and outflow of events.

Outflow rate might be limited, if it takes 1 millisecond to process an event, you might only be able to process 16 events per frame before you start dropping your FPS.

Inflow rate might change a lot. E.g. when moving 8 KHz moving mouse you get 8000 events/second, or 133 events/frame at 60 Hz. But when not moving it you get 0 events/second.

If your outflow rate is smaller than inflow rate you might find yourself in a death spiral:

- It takes longer to process current queue
- In meantime more events arrive
- It takes even longer to process new events
- In meantime even more events arrive
- ...

We can't escape queuing theory, so it's a good idea to be able to do something more radical when you need it: drop events in case of overflow, coalesce them, etc.

::::
:::: column

![](markdown/e3_event_queue_bucket.png)

(From [here](https://www.chegg.com/homework-help/questions-and-answers/1-filling-damaged-bucket-tank-cross-sectional-area-si-hole-bottom-cross-sectional-area--ta-q47015584))

::::
:::

## Event coalescing

Coalescing is destructively-ish merging two or more things together into one thing.
Usually term is used in memory management, and sometimes in timers.

In input events often we coalesce consecutive mouse movements together.

Before:

- event 1, mouse move, pos (100, 100) delta (50, 50)
- event 2, mouse move, pos (200, 200) delta (100, 100)
- event 3, mouse move, pos (250, 250) delta (150, 150)

After:

- event 1, mouse move, pos (250, 250) delta (300, 300)

## Ring buffer

Ring buffer is an excellent way to implement fixed size FIFO.

![](markdown/e3_event_queue_ring_buffer.gif)

(From [here](http://www.mathcs.emory.edu/~cheung/Courses/171/Syllabus/8-List/array-queue2.html))

# Self study

## Self study

- What are event queues and why they are used?
- What are the common data structures used to implement them?
- [Game Programming Patterns: Event Queue](http://gameprogrammingpatterns.com/event-queue.html)

Optionally:

- Read up and implement ring buffer yourself.
