---
layout: post
title:  "Replacing PlayerPrefs"
date:   2020-03-11 15:08:00 +0100
tags: unity, gamedev, c#
summary: "You might’ve been using Unity for quite some time before you first encountered the problem of saving your game. Unity comes well equipped for a variety of tasks, but saving (and loading) is not one of them..."
---

You might’ve been using Unity for quite some time before you first encountered the problem of saving your game. Unity comes well equipped for a variety of tasks, but saving (and loading) is not one of them. In your effort to tackle the problem, you did a quick google search and ran into answers mentioning `PlayerPrefs` as the solution.

While using PlayerPrefs for storing player authored data may seem simple, there are multiple reasons it’s actually a bad idea the biggest of which is that PlayerPrefs stores data in the registry (on Windows).

The registry is a low-level database used for storing application metadata like the executable location, version data, startup settings, etc. Storing a huge save file among those seems a bit ridiculous.

In a perfect scenario, we’d love to have a way to easily store saves and preferences in files that can be visible to the users, easily transferred between installs of our games and allow us, the developers, to have control over the writing and reading processes.

In this post, I'll mainly focus on making a direct replacement for `PlayerPrefs`, in a later post I'll go into building a system that's more production ready.
`PlayerPrefs` is a static class, globally accessible, so our `StorageSystem` class will be the same. Here is the class structure without implementation:

{% highlight cs %}
public static class StorageSystem
{
  private static void Initialize() 
  {
    throw new System.NotImplementedException();
  }

  public static void Save() 
  {
    throw new System.NotImplementedException();
  }

  public static void DeleteKey(string key) 
  {
    throw new System.NotImplementedException();
  }

  public static void DeleteAll() 
  {
    throw new System.NotImplementedException();
  }

  public static bool HasKey(string key) 
  {
    throw new System.NotImplementedException();
  }

  public static string GetString(string key)
  {
    throw new System.NotImplementedException();
  }

  public static void SetString(string key, string value) 
  {
    throw new System.NotImplementedException();
  }
}
{% endhighlight %}

In the `Initialize` method we'll ensure that our save directory exists and initialize a dictionary to store the data in RAM. We'll leave loading the save file for later.

{% highlight cs %}
using System.IO;

public static class StorageSystem
{
  public static readonly string SAVE_PATH = Path.Combine(Application.persistentDataPath, "Saves/Save.save");

  private static Dictionary<string, string> _data;

  private static void Initialize() 
  {
    // make sure all directories exist
    Directory.CreateDirectory(Path.GetDirectoryName(SAVE_PATH));

    _data = new Dictionary<string, string>();

    throw new System.NotImplementedException("TODO: Load the save file into _data!");
  }

  ...
}
{% endhighlight %}

To make sure our `Initialize` method gets called when Unity loads, we can use the [`RuntimeInitializeOnLoadMethod`](https://docs.unity3d.com/ScriptReference/RuntimeInitializeOnLoadMethodAttribute.html) attribute. This attribute will run the method it's attached to when the game loads for the first time. By passing in the `RuntimeInitializeLoadType.BeforeSceneLoad` load type to the attribute, we can make sure our storage system is initialized before any `Awake` method executes:

{% highlight cs %}
using System.IO;
using UnityEngine;

public static class StorageSystem
{
  public static readonly string SAVE_PATH = Path.Combine(Application.persistentDataPath, "Saves/Save.save");

  private Dictionary<string, string> _data;

  [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
  private static void Initialize() 
  {
    // make sure all directories exist
    Directory.Create(Path.GetDirectoryName(SAVE_PATH));

    _data = new Dictionary<string, string>();

    throw new System.NotImplementedException("TODO: Load the save file into _data!");
  }

  ...
}
{% endhighlight %}

Before we can load the save file, or save it for that matter, we have to decide how we'll turn our string dictionary into a file. For this example, we'll use Json.NET to serialize our dictionary into JSON text and write it to a file. You can download Json.NET [here](https://www.newtonsoft.com/json).
After you download and extract the archive, go to the `Bin/net45` folder and copy the all of the files in it into a folder named `Plugins` in your project's assets folder (if the plugins folder doesn't exist, create one).

![Json.NET imported](/assets/Json_dot_net_imported.png)

Now we can use the `JsonConvert.DeserializeObject<T>()` method from the `Newtonsoft.Json` namespace to convert from a JSON string to an object.
Let's load the current save file into our data dictionary in the `Initialize` method if the file exists:

{% highlight cs %}
using System.IO;
using UnityEngine;

public static class StorageSystem
{
  public static readonly string SAVE_PATH = Path.Combine(Application.persistentDataPath, "Saves/Save.save");

  private Dictionary<string, string> _data;

  [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
  private static void Initialize() 
  {
    // make sure all directories exist
    Directory.CreateDirectory(Path.GetDirectoryName(SAVE_PATH));

    if (File.Exists(SAVE_PATH))
    {
      string fileContents = File.ReadAllText(SAVE_PATH);
      _data = JsonConvert.DeserializeObject<Dictionary<string, string>>(fileContents);
    }
    else
    {
      // no save file exists, create a new dictionary
      _data = new Dictionary<string, string>();
    }
  }

  ...
}
{% endhighlight %}

Similiarly, let's also implement the save method by storing the contents of the `_data` dictionary to the file:

{% highlight cs %}
using System.IO;
using UnityEngine;

public static class StorageSystem
{
  ...

  public static void Save() 
  {
    // make sure all directories exist
    Directory.CreateDirectory(Path.GetDirectoryName(SAVE_PATH));

    string fileContents = JsonConvert.SerializeObject(_data);
    File.WriteAllText(SAVE_PATH, fileContents);
  }

  ...
}
{% endhighlight %}

Now the `_data` dictionary gets stored and loaded from our permanent storage device whenever we need it to.
We can proceed to implement the rest of the methods of our `StorageSystem` by just manipulating the contents of the `_data` dictionary.

The `DeleteKey` and `HasKey` methods only need to call the `Remove` and `ContainsKey` method on the dictionary:

{% highlight cs %}
public static class StorageSystem
{
  ...

  public static void DeleteKey(string key) 
  {
    _data.Remove(key);
  }

  public static void DeleteAll() 
  {
    _data.Clear();
  }

  public static bool HasKey(string key) 
  {
    return _data.ContainsKey(key);
  }

  ...
}
{% endhighlight %}

For the `GetString` method we simply return the dictionary entry under the provided key and for the `SetString` we set the value in the dictionary:

{% highlight cs %}
public static class StorageSystem
{
  ...

  public static string GetString(string key)
  {
    return _data[key];
  }

  public static void SetString(string key, string value) 
  {
    _data[key] = value;
  }
}
{% endhighlight %}

To completely replace `PlayerPrefs` we'd need to implement the `int` and `float` methods as well. Instead of replicating the limited model found in `PlayerPrefs`, let's rather turn the `Get` and `Set` methods generic:

{% highlight cs %}
public static class StorageSystem
{
  ...

  public static T Get<T>(string key)
  {
    return JsonConvert.DeserializeObject<T>(_data[key]);
  }

  public static void Set<T>(string key, T value) 
  {
    _data[key] = JsonConvert.SerializeObject(value);
  }
}
{% endhighlight %}

Simple, right? Now our `PlayerPrefs` replacement is done, we can test it using a `MonoBehavior` script.

{% highlight cs %}
using System.Diagnostics;
using UnityEngine;
using Debug = UnityEngine.Debug;

public class StorageSystemTest : MonoBehaviour
{
    private void Awake()
    {
        Debug.Log("Starting test!");
        
        // should print false if this is the first time running, otherwise true
        Debug.Log($"HasKey(\"TEST\") = {StorageSystem.HasKey("TEST")}");
        
        Debug.Log($"SetString(\"TEST\", \"Hello world!\")");
        StorageSystem.Set("TEST", "Hello world!");
        
        // should print true
        Debug.Log($"HasKey(\"TEST\") = {StorageSystem.HasKey("TEST")}");
        Debug.Assert(StorageSystem.HasKey("TEST") == true);
        
        // should print Hello world!
        Debug.Log($"GetString(\"TEST\") = {StorageSystem.Get<string>("TEST")}");
        Debug.Assert(StorageSystem.Get<string>("TEST") == "Hello world!");
        
        Debug.Log($"SetString(\"TEST2\", \"Hello world 2!\")");
        StorageSystem.Set("TEST2", "Hello world 2!");
        
        // should print true
        Debug.Log($"HasKey(\"TEST2\") = {StorageSystem.HasKey("TEST2")}");
        Debug.Assert(StorageSystem.HasKey("TEST2") == true);
        Debug.Log($"DeleteKey(\"TEST2\")");
        StorageSystem.DeleteKey("TEST2");
        
        // should print false
        Debug.Log($"HasKey(\"TEST2\") = {StorageSystem.HasKey("TEST2")}");
        Debug.Assert(StorageSystem.HasKey("TEST2") == false);
        
        Debug.Log($"Save() -- look into your project folder, there should be a Saves directory there with a save file inside");
        StorageSystem.Save();
        
        Process.Start("explorer.exe", @"Saves\");
        Process.Start("notepad.exe", @"Saves\Save.save");
    }
}
{% endhighlight %}

The complete `StorageSystem` script:

{% highlight cs %}
using Newtonsoft.Json;
using System.Collections.Generic;
using System.IO;
using UnityEngine;

public static class StorageSystem
{
    public static readonly string SAVE_PATH = Path.Combine(Application.persistentDataPath, "Saves/Save.save");

    private static Dictionary<string, string> _data;

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    private static void Initialize() 
    {
        // make sure all directories exist
        Directory.CreateDirectory(Path.GetDirectoryName(SAVE_PATH));

        if (File.Exists(SAVE_PATH))
        {
            string fileContents = File.ReadAllText(SAVE_PATH);
            _data = JsonConvert.DeserializeObject<Dictionary<string, string>>(fileContents);
        }
        else
        {
            // no save file exists, create a new dictionary
            _data = new Dictionary<string, string>();
        }
    }

    public static void Save() 
    {
        // make sure all directories exist
        Directory.CreateDirectory(Path.GetDirectoryName(SAVE_PATH));

        string fileContents = JsonConvert.SerializeObject(_data);
        File.WriteAllText(SAVE_PATH, fileContents);
    }

    public static void DeleteKey(string key) 
    {
        _data.Remove(key);
    }

    public static void DeleteAll() 
    {
        _data.Clear();
    }

    public static bool HasKey(string key) 
    {
        return _data.ContainsKey(key);
    }

    public static T Get<T>(string key)
    {
        return JsonConvert.DeserializeObject<T>(_data[key]);
    }

    public static void Set<T>(string key, T value) 
    {
        _data[key] = JsonConvert.SerializeObject(value);
    }
}

{% endhighlight %}