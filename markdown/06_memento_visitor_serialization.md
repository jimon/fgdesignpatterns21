---
title: Memento, Visitor, Serialization
author: Dmytro Ivanov
date: Nov, 2021
---

# Memento pattern

## Memento pattern

Memento pattern is about ability to save/restore the originator via opaque objects.

- Originator is the object with internal state.
- Memento is the opaque object storing the internal state of originator.
- Caretaker is something that invokes save/restore.

## Memento C# example

::: columns
:::: column

```{.csharp .number-lines}
public class Originator {
	public Color Color { get; set; }

	public Memento SaveToMemento() {
		return new Memento(Color);
	}

	public void RestoreFromMemento(Memento memento) {
		Color = memento.ToColor();
	}
}

// Usually a "passive" data structure, e.g. with no behavior,
// also usually without references.
// Could also be blittable/unmanaged.
public class Memento {
	private readonly string colorString;

	internal Memento(Color color) {
		colorString = ColorUtility.ToHtmlStringRGBA(color);
	}

	internal Color ToColor() {
		ColorUtility.TryParseHtmlString(colorString, out var color);
		return color;
	}
}
```

::::
:::: column
```{.csharp .number-lines}
public class Caretaker {
	void Foo() {
		var undoRedo = new Stack<Memento>();

		var originator = new Originator();

		originator.Color = Color.cyan;
		undoRedo.Push(originator.SaveToMemento());

		originator.Color = Color.red;
		undoRedo.Push(originator.SaveToMemento());

		originator.Color = Color.magenta;

		originator.RestoreFromMemento(undoRedo.Pop());

		originator.RestoreFromMemento(undoRedo.Pop());
	}
}
```
::::
:::

## Memento pattern

- Memento pattern can be achieved in different ways.
- In practice Memento usually seen as a generalization to serialization.
- Can also be seen as generalization of undo/redo systems.
- Memento is a pattern, serialization is a process.
- In gamedev Memento itself is rarely used, serialization and undo/redo stacks are used instead.

# Visitor pattern

## Visitor pattern overview

Allows for (recursively) traversing an hierarchy of objects and/or their properties.

We implement it by creating a visitor object/interface, that gets callbacks when we "visit" something.

Objects then "accept" a visitor and "dispatches" the requests by calling callbacks on a visitor.

Visitor is a generalization of running an operation over nested objects.

## Visitor pattern example

::: columns
:::: column

```{.csharp .number-lines}
public interface IItemVisitor
{
	void VisitArmor(Armor item);
	void VisitWeapon(Weapon item);
	void VisitItemBag(Item item);
}

public abstract class Item
{
	public abstract void Accept(IItemVisitor visitor);
}

public class Armor : Item
{
	public override void Accept(IItemVisitor visitor) {
		visitor.VisitArmor(this);
	}
}

public class Weapon : Item
{
	public override void Accept(IItemVisitor visitor) {
		visitor.VisitWeapon(this);
	}
}

public class ItemBag : Item
{
	public List<Item> items = new();
	public override void Accept(IItemVisitor visitor) {
		visitor.VisitItemBag(this);

		foreach (var item in items)
			item.Accept(visitor);
	}
}
```
::::
:::: column
```{.csharp .number-lines}
public class IventoryDebugger : IItemVisitor
{
	public void VisitArmor(Armor item) {
		Console.WriteLine("got an armor {...}");
	}

	public void VisitWeapon(Weapon item) {
		Console.WriteLine("got a weapon {...}");
	}

	public void VisitItemBag(Item item) {
		Console.WriteLine("got item bag {...}");
	}
}

class Foo {
	void Bar() {
		var bag = new ItemBag();
		bag.items.Add(new Armor());
		bag.items.Add(new Weapon());
		bag.Accept(new IventoryDebugger());
	}
}

```

::::
:::

## Visitor pattern remarks

- Usually involved in save/load systems.
- Sometimes used for traversing Abstract Syntax Trees (AST) when parsing.
- Could be used to add new operation to already sealed classes.
- Notice that traversal could be Breath-First or Depth-First

# Serialization

## Serialization

From [wiki](https://en.wikipedia.org/wiki/Serialization).

> Serialization is the process of translating a data structure or object state into a format that can be stored (for example, in a file or memory data buffer) or transmitted (for example, over a computer network) and reconstructed later (possibly in a different computer environment)

- Usually automated.
- Can target human-readable formats.
- Can target schema-driven formats.
- Deals with pointer swizzling, meaning going from pointers to numbers-in-a-file and back.
- Gotcha: Recursive objects.

Marshalling is another closely related concept.

## Manual serialization straight into bytes

```{.csharp .number-lines}
public class MyObject {
	public Color Color { get; set; }

	public byte[] Serialize() {
		// Color32 is like Color, but R/G/B/A values are stored as bytes from 0 to 255
		var t = (Color32)Color;
		return new[]{t.r, t.g, t.b, t.a};
	}

	public void Deserialize(byte[] d) {
		// Color32->Color has implicit conversion
		Color = new Color32(d[0], d[1], d[2], d[3]);
	}
}
```

- Can be improved by adding some intermediate representations.
- Also consider nested objects, e.g. you mind want to call `.Serialize()` recursively. Or use visitor pattern?

## Manual serialization via intermediate representation, part one

::: columns
:::: column

```{.csharp .number-lines}
[StructLayout(LayoutKind.Explicit, Size = 8)]
public struct MyObjectBinaryPacked
{
	[FieldOffset(0)] public uint version;
	[FieldOffset(4)] public unsafe fixed byte ColorBytes[4];

	public byte[] ToBytes() {
		... magic ...
	}

	public static MyObjectBinaryPacked FromBytes(byte[] bytes) {
		... magic ...
	}
}

// This struct in memory will be exactly 8 bytes:
// 00 01 02 03   04 05 06 07
// --VERSION-- |  R  G  B  A
```

::::
:::: column

```{.csharp .number-lines}
public class MyObject {
	public Color Color { get; set; }

	public unsafe MyObjectBinaryPacked Serialize() {
		var t = (Color32)Color;

		var data = new MyObjectBinaryPacked();
		data.version = 1;
		data.ColorBytes[0] = t.r;
		data.ColorBytes[1] = t.g;
		data.ColorBytes[2] = t.b;
		data.ColorBytes[3] = t.a;
		return data;
	}

	public unsafe void Deserialize(MyObjectBinaryPacked data) {
		if (data.version == 1)
		{
			Color = new Color32(
				data.ColorBytes[0],
				data.ColorBytes[1],
				data.ColorBytes[2],
				data.ColorBytes[3]);
		}
		else
		{
			// More on this later
		}
	}
}
```

::::
:::

## Manual serialization via intermediate representation, part two

Usually easier in C/C++, but can be done in C# if you are friends with Marshaling and GC.
This is very advanced low level C#, don't worry about it, but you might see it from time to time.

```{.csharp .number-lines}
[StructLayout(LayoutKind.Explicit, Pack = 1, Size = kSize)]
public struct MyObjectBinaryPacked
{
	private const int kSize = 8;

	[FieldOffset(0)] public uint version;
	[FieldOffset(4)] public unsafe fixed byte ColorBytes[4];

	public byte[] ToBytesViaPointers() {
		var arr = new byte[kSize];

		fixed (MyObjectBinaryPacked* src = &this)
			fixed (byte* dst = arr)
				UnsafeUtility.MemCpy(dst, src, kSize);

		return arr;
	}

	public byte[] ToBytesViaMarshaling() {
		// "LocalAlloc", allocates heap memory bypassing GC
		var ptr = Marshal.AllocHGlobal(kSize);

		// This is the same mechanism as calling C from C# via pinvoke
		// It relies on Application Binary Interface (ABI) somewhat
		Marshal.StructureToPtr(this, ptr, false);

		// Copies "void*" data to managed array
		var arr = new byte[kSize];
		Marshal.Copy(ptr, arr, 0, kSize);

		// Frees heap memory
		Marshal.FreeHGlobal(ptr);

		return arr;
	}
}
```

If you're really into everything low level, go and checkout [System V ABI AMD64](https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf), it describes how everything works on x64 linux on binary level: function calls, function arguments, stack, etc.

Also [C# Type Marshaling](https://docs.microsoft.com/en-us/dotnet/standard/native-interop/type-marshaling).

## Automated serialization

```{.csharp .number-lines}
[Serializable]
public struct MyColor {
	public float r, g, b, a;
}

[Serializable]
public class MyObject {
	public MyColor Color { get; set; }
}

void Foo() {
	var obj = new MyObject();

	var stream = new MemoryStream();
	new BinaryFormatter().Serialize(stream, obj);

	// ...

	stream.Close();
}
```

Usually a magic powered by reflection and low level stuff. Less magical in languages with poor reflection support, e.g. C++.

## Unity serialization

```{.csharp .number-lines}
public class TestComponent : ScriptableObject {
	public int test1;

	[SerializeField]
	private int test2;
}

class Foo {
	void Bar() {
		var instance = ScriptableObject.CreateInstance<TestComponent>();

		var obj = new SerializedObject(instance);
		obj.FindProperty("test1").intValue = 10;
		obj.FindProperty("test2").intValue = 20;
		obj.ApplyModifiedProperties();

		Debug.Log($"value = {instance.test1}");
		Debug.Log($"value = {obj.FindProperty("test2").intValue}");
	}
}
```

## Unity advanced serialization

From [here](https://docs.unity3d.com/2021.2/Documentation/ScriptReference/ISerializationCallbackReceiver.html):

```{.csharp .number-lines}
public class SerializationCallbackScript : MonoBehaviour, ISerializationCallbackReceiver
{
	public List<int> _keys = new List<int> { 3, 4, 5 };
	public List<string> _values = new List<string> { "I", "Love", "Unity" };

	// Unity doesn't know how to serialize a Dictionary
	public Dictionary<int, string>  _myDictionary = new Dictionary<int, string>();

	public void OnBeforeSerialize()
	{
		_keys.Clear();
		_values.Clear();

		foreach (var kvp in _myDictionary)
		{
			_keys.Add(kvp.Key);
			_values.Add(kvp.Value);
		}
	}

	public void OnAfterDeserialize()
	{
		_myDictionary = new Dictionary<int, string>();

		for (int i = 0; i != Math.Min(_keys.Count, _values.Count); i++)
			_myDictionary.Add(_keys[i], _values[i]);
	}
}
```

## Save/load to JSON

```{.csharp .number-lines}
// .NET Core 3.1 / .NET 5+
using System.Text.Json;
using System.Text.Json.Serialization;

public class MyObject
{
	public int Value1 { set; get; }

	[JsonInclude]
	public int Value2;
}

class Foo {
	void Bar() {
		var myObject = new MyObject
		{
			Value1 = 10,
			Value2 = 20
		};

		var options = new JsonSerializerOptions { WriteIndented = true };
		var json = JsonSerializer.Serialize(myObject, options);
		Console.WriteLine(json);
		// Prints:
		// {
		//   "Value1": 10,
		//   "Value2": 20
		// }

		var myObject2 = JsonSerializer.Deserialize<MyObject>(json);
		Console.WriteLine($"Value1={myObject2.Value1} Value2={myObject2.Value2}");
	}
}
```

## Save/load to JSON in Unity

```{.csharp .number-lines}
public class TestComponent : ScriptableObject {
	public int test1;

	[SerializeField]
	private int test2;

	public void SetTest2(int value) { test2 = value; }
}

class Foo {
	void Bar() {
		var foo = new TestComponent();
		foo.test1 = 1;
		foo.SetTest2(2);

		var json = JsonUtility.ToJson(foo);
		Debug.Log(json);

		var bar = new TestComponent();
		JsonUtility.FromJsonOverwrite(json, bar);
		Debug.Log(bar);
	}
}
```


# Schema

## Schema, serialization and upgrading

One day we write:

```{.csharp .number-lines}
public class MyObject // version 1
{
	public int Value1;
}

...

var foo = new MyObject { ... };
SaveToFile("foo.txt", foo);
```

Then some time later we add another field:

```{.csharp .number-lines}
public class MyObject // version 2
{
	public int Value1;
	public int Value2;
}

...

var foo = LoadFromFile<MyObject>("foo.txt");
// What is the value of foo.Value2?
```

Now what?

## Schema, serialization and upgrading

> The database schema is its structure described in a formal language supported by the database management system.

Quick example from MySQL:
```{.sql .number-lines}
CREATE DATABASE example;
USE example;
 
DROP TABLE IF EXISTS customer;
 
CREATE TABLE customer (
 id INT AUTO_INCREMENT PRIMARY KEY,
 postalCode VARCHAR(15) default NULL,
)
```

Another quick example from [Google Protobuf](https://en.wikipedia.org/wiki/Protocol_Buffers)
```{.protobuf .number-lines}
// protobuf
message Person {
  required string name = 1;
  required int32 id = 2;
  optional string email = 3;
}
```

When we serialize, we create object instance representation in another media.

Both original object instance and serialized instance data store the structure of the original object (literally - fields/properties/etc).

Sometimes structure change.

## Schema, serialization and upgrading

The upgrade behavior has to be defined somewhere:

- Usually standard behavior is to set fields to default (`0`, `0.0f`, `null`, etc), write your code to handle this.
- Store struct version in serialized data, and write data upgrade methods in deserializer method, e.g. reader should be able to parse data of version 1 and 2 but writer only writes version 2.
- Schema language usually is able to define upgrade behavior

```{.protobuf .number-lines}
message Person {
  required string name = 1;
}
```
To
```{.protobuf .number-lines}
message Person {
  required string name = 1;
  optional int32  id   = 2 [default = -1];
}
```

What is actually happening: your serialization data stores some part of your API.
Even if internals of serialized data are not public, the whole schema becomes a public contract.
New version of the app should be able to open files from previous version.

Every time you serialize, have an answer to what you gonna do when a need to add/change/delete fields will arise.

## Schema, serialization and upgrading

In Unity `FormerlySerializedAs` can also be used to support renaming of fields.

Before

```{.csharp .number-lines}
public class TestComponent : ScriptableObject
{
	public int test1;

	[SerializeField]
	private int test2;
}
```

To

```{.csharp .number-lines}
public class TestComponent : ScriptableObject
{
	public int test1;

	[SerializeField]
	[FormerlySerializedAs("test2")]
	private int test3;
}
```

# Self study

## Self study

- What is visitor pattern and how it is usually implemented?
- What is serialization and what are the common ways to implement it?
- What are the challenges of dealing with serialized data? (tip: structure change, etc)
- What is data schema and why we use them?

- [Wiki: Memento pattern](https://en.wikipedia.org/wiki/Memento_pattern)
- [Wiki: Visitor pattern](https://en.wikipedia.org/wiki/Visitor_pattern)
- [Wiki: Serialization](https://en.wikipedia.org/wiki/Serialization)
- [Unity: Script Serialization](https://docs.unity3d.com/2021.2/Documentation/Manual/script-Serialization.html), magic has it's own playbook

Optionally:

- [Wiki: Protocol Buffers (protobuf)](https://en.wikipedia.org/wiki/Protocol_Buffers)
- [protobuf](https://developers.google.com/protocol-buffers/docs/overview)
- [Wiki: FlatBuffers](https://en.wikipedia.org/wiki/FlatBuffers)

Exercise:
```
- Implement a visitor pattern for your item/inventory/shop/etc system of choice.
- Write to console names of your items/etc from the visitor.
- Look inside unity .asset/.unity files. Try changing something in the scene (like a color) and see how it changes in the file.

Optionally:
- Try adding properties to MonoBehavior that Unity wouldn't serialize, like a 2d array.
- Try observing "hot reload" where you modify some code in play mode, Unity does domain reload but all your objects are "recovered" via serialization.
- Use SerializedObject / SerializedProperty to edit a private serialized property on some game object.

Advanced:
- Implement a manual serialization of some struct/class.
- Store data as bytes in binary file on disk.
- Open binary file with hex editor and check that it's correct.
```