---
layout: post
title:  "Replacing PlayerPrefs"
date:   2020-03-11 15:08:00 +0100
tags: unity, gamedev, c#
---

You might’ve been using Unity for quite some time before you first encountered the problem of saving your game. Unity comes well equipped for a variety of tasks, but saving (and loading) is not one of them. In your effort to tackle the problem, you did a quick google search and ran into answers mentioning `PlayerPrefs` as the solution.

While using PlayerPrefs for storing player authored data may seem simple, there are multiple reasons it’s actually a bad idea the biggest of which is that PlayerPrefs stores data in the registry (on Windows).

The registry is a low-level database used for storing application metadata like the executable location, version data, startup settings, etc. Storing a huge save file among those seems a bit ridiculous.

In a perfect scenario, we’d love to have a way to easily store saves and preferences in files that can be visible to the users, easily transferred between installs of our games and allow us, the developers, to have control over the writing and reading processes.

Let’s start with usability. Our first goal is to define how we’d like to interact with our new storage system. We can define this behavior upfront by defining an interface.
An interface is a definition of a contract that a class implementing that interface needs to fulfill. It’s a way of promising that a certain class has certain methods. For example, we can define an interface `ICountable` that represents a contract that a class has some count:

{% highlight cs %}
public interface ICountable {
  int GetCount();
}
{% endhighlight %}

As you can see, interfaces don't provide implementation details, they just tell us what methods have to exist. Also notice that we didn't say if the method `GetCount` is private or public. That's because all interface methods are always public.
A class can then “sign the contract” of the interface like so:

{% highlight cs %}
public class ExampleCountable : ICountable {
}
{% endhighlight %}

It then has to deliver on what was promised by providing the methods declared in the interface:

{% highlight cs %}
public class ExampleCountable : ICountable {
  public void GetCount() {
    return 16;
  }
}
{% endhighlight %}

By defining an interface, we can think about what methods a class should have to fulfill a certain role. In our case, we want to define what’s common to all storage system implementations. We’ll call the interface we want to implement `IStorageSystem`.

{% highlight cs %}
public interface IStorageSystem {
}
{% endhighlight %}

In `PlayerPrefs` we usually use two main method types, Get and Set. So for example, to store a string using `PlayerPrefs` we'd call `PlayerPrefs.SetString(name, value)`. When we want to retrieve an string we'd use `PlayerPrefs.GetString(name)`.
We can take a similar approach to constructing our storage system:

{% highlight cs %}
public interface IStorageSystem {
  void SetString(string key, string value);
  string GetString(string key);
}
{% endhighlight %}

We also need to cover the case when the user wants to delete a save and when they want to check if a save exists:

{% highlight cs %}
public interface IStorageSystem {
  void SetString(string key, string value);
  string GetString(string key);

  void Delete(string key);
  bool Exists(string key);
}
{% endhighlight %}

That's it! Now we just need to implement the system itself.
We start implementing a storage system for standalone platforms (windows, mac, linux) by declaring another class:

{% highlight cs %}
public class StandaloneStorageSystem : IStorageSystem {
}
{% endhighlight %}

This will produce some compile errors because our `StandaloneStorageSystem` doesn't yet implement the interface, it doesn't fulfil the contract. So let's implement it right away.

We'll store the different save files (the different keys) in separate files on the user's storage device.
To accomplish this, we'll be using classes from the `System.IO` namespace. I'll give a quick overview of the methods we'll use:

- `Path.Combine` - this method joins two paths in a way that makes sense for the operating system - for example `Path.Combine("SomeFolder", "SomeFile")` will return `"SomeFolder\SomeFile"` on Windows
- `File.ReadAllText` - this method takes a file path and returns a string containing all the text in the file located on that path
- `File.WriteAllText` - this method takes a file path and a string and writes the contents of the string to a file specified by the path
- `File.Delete` - this method deletes a file on a given file path
- `File.Exists` - this method takes a file path and checks if a file exists at the provided path
- `Directory.Create` - this method takes a directory path and creates all directories that don't already exists in the provided path

Using these methods, we have enough control over the filesystem to begin implementing our storage solution.

We'll store all our saves in a subdirectory named "Saves" of the directory where our executable file is.
To define this path, we'll declare a constant string in our storage implementation:

{% highlight cs %}
using System.IO;

public class StandaloneStorageSystem : IStorageSystem {
  public const string SAVE_FOLDER = "Saves";
}
{% endhighlight %}

When we initialize the storage system for the first time, we have to make sure that our save folder exists. For that we'll use the constructor:

{% highlight cs %}
using System.IO;

public class StandaloneStorageSystem : IStorageSystem {
  public const string SAVE_FOLDER = "Saves";

  public StandaloneStorageSystem() {
    // create the saves folder if it doesn't exist
    Directory.Create(SAVE_FOLDER);
  }
}
{% endhighlight %}

Let's start implementing the `SetString` method.
For every "key" that our storage system stores, we will create a new file named the same as the key and store the value inside.

{% highlight cs %}
using System.IO;

public class StandaloneStorageSystem : IStorageSystem {
 
  ...

  public void SetString(string key, string value) {
    string filePath = Path.Combine(SAVE_FOLDER, key);
    File.WriteAllText(filePath, value);
  }
}
{% endhighlight %}

Our `GetString` method is similar, but instead of writing to the file, we'll read from it:

{% highlight cs %}
using System.IO;

public class StandaloneStorageSystem : IStorageSystem {
 
  ...

  public void SetString(string key, string value) {
    string filePath = Path.Combine(SAVE_FOLDER, key);
    File.WriteAllText(filePath, value);
  }

  public string GetString(string key) {
    string filePath = Path.Combine(SAVE_FOLDER, key);
    return File.ReadAllText(filePath);
  }
}
{% endhighlight %}

We'll implement the delete and exists methods using the similar `System.IO` calls:

{% highlight cs %}
using System.IO;

public class StandaloneStorageSystem : IStorageSystem {
 
  ...

  public void Delete(string key) {
    string filePath = Path.Combine(SAVE_FOLDER, key);
    File.Delete(filePath);
  }

  public bool Exists(string key) {
    string filePath = Path.Combine(SAVE_FOLDER, key);
    return File.Exists(filePath);
  }
}
{% endhighlight %}

We repeated a single line, `string filePath = Path.Combine(SAVE_FOLDER, key);`, in all our methods. To keep things a bit cleaner, let's separate that line into a method:

{% highlight cs %}
using System.IO;

public class StandaloneStorageSystem : IStorageSystem {
  public const string SAVE_FOLDER = "Saves";

  public StandaloneStorageSystem() {
    // create the saves folder if it doesn't exist
    Directory.Create(SAVE_FOLDER);
  }

  private string GetFilePathForKey(string key) {
    return Path.Combine(SAVE_FOLDER, key);
  }

  public void SetString(string key, string value) {
    string filePath = GetFilePathForKey(key);
    File.WriteAllText(filePath, value);
  }

  public string GetString(string key) {
    string filePath = GetFilePathForKey(key);
    return File.ReadAllText(filePath);
  }

  public void Delete(string key) {
    string filePath = GetFilePathForKey(key);
    File.Delete(filePath);
  }

  public bool Exists(string key) {
    string filePath = GetFilePathForKey(key);
    return File.Exists(filePath);
  }
}
{% endhighlight %}

While this might seem like we didn't accomplish much, now we can easily change how our storage files are formatted. Let's add a '.save' extension to all our files by changing the new method:

{% highlight cs %}
using System.IO;

public class StandaloneStorageSystem : IStorageSystem {
 
  ...

  private string GetFilePathForKey(string key) {
    return Path.Combine(SAVE_FOLDER, key + ".save");
  }

  ...
}
{% endhighlight %}

`PlayerPrefs` supports getting and setting ints and floats in addition to strings, let's add the 4 additional methods for ints and floats to our interface:

{% highlight cs %}
public interface IStorageSystem {

  void SetString(string key, string value);
  string GetString(string key);
 
  void SetFloat(string key, float value);
  float GetFloat(string key);
 
  void SetInt(string key, int value);
  int GetInt(string key);

  void Delete(string key);
  bool Exists(string key);

}
{% endhighlight %}

We changed the contract, so we have to implement the new methods in our class as well. To store an `int` or a `float` in a file we have to convert them to a string:

{% highlight cs %}
using System.IO;

public class StandaloneStorageSystem : IStorageSystem {
 
  ...

  public void SetInt(string key, int value) {
    string stringifiedValue = value.ToString();
    SetString(key, stringifiedValue);
  }

  public void SetFloat(string key, float value) {
    string stringifiedValue = value.ToString();
    SetString(key, stringifiedValue);
  }
}
{% endhighlight %}

When we get the values from the file, they're represented as text (string), so we have to convert them to `float` or `int`. To do that we can use parsing methods:

- `int.Parse(someString)`
- `float.Parse(someString)`

Let's implement the get methods using `Parse`:

{% highlight cs %}
using System.IO;

public class StandaloneStorageSystem : IStorageSystem {
 
  ...

  public int GetInt(string key) {
    string stringifiedValue = GetString(key);
    return int.Parse(stringifiedValue);
  }

  public float GetFloat(string key) {
    string stringifiedValue = GetString(key);
    return float.Parse(stringifiedValue);
  }
}
{% endhighlight %}

And that's it! Now we have a replacement for PlayerPrefs that uses files instead of the registry!

Here's an example of how we might use the system:

{% highlight cs %}
using UnityEngine;

public class StorageSystemExample : MonoBehaviour {
 
  IStorageSystem storageSystem;

  public string TextToStore = "test";

  void Start() {
    storageSystem = new StandaloneStorageSystem();

    if (storageSystem.Exists("SAVE1")) {
      string previousText = storageSystem.GetString("SAVE1");
      Debug.Log("Found previous save with text: " + previousText);
    }

    storageSystem.SetString("SAVE1", TextToStore);

    Debug.Log("Saved " + TextToStore + " in a new save");
  }

}
{% endhighlight %}

If you attach the component to a game object and run the game the system will save a text to a file. You can go to your project folder and see that a new folder has been made named "Saves" and inside of it is a file named "SAVE1.save".
You can even open the file with notepad and see the contents!

This leaves us with a simple system that replaces `PlayerPrefs` with a storage system that saves data to files the players can easily manage. The final contents of `StorageSystem.cs` can be found below:

{% highlight cs %}
using System.IO;

public class StandaloneStorageSystem : IStorageSystem {

  public const string SAVE_FOLDER = "Saves";

  public StandaloneStorageSystem() {
    // create the saves folder if it doesn't exist
    Directory.Create(SAVE_FOLDER);
  }

  private string GetFilePathForKey(string key) {
    return Path.Combine(SAVE_FOLDER, key);
  }

  public void SetString(string key, string value) {
    string filePath = GetFilePathForKey(key);
    File.WriteAllText(filePath, value);
  }

  public string GetString(string key) {
    string filePath = GetFilePathForKey(key);
    return File.ReadAllText(filePath);
  }

  public void Delete(string key) {
    string filePath = GetFilePathForKey(key);
    File.Delete(filePath);
  }

  public bool Exists(string key) {
    string filePath = GetFilePathForKey(key);
    return File.Exists(filePath);
  }

  public int GetInt(string key) {
    string stringifiedValue = GetString(key);
    return int.Parse(stringifiedValue);
  }

  public float GetFloat(string key) {
    string stringifiedValue = GetString(key);
    return float.Parse(stringifiedValue);
  }
}
{% endhighlight %}

In the next post, I'll go into extending the storage system to support more types (using serialization) and optimization steps you should consider if you're writing and reading from storage more than once per run.