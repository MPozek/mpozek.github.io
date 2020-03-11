This unfortunately won't work. The method `File.WriteAllText` expects we provide it with text (string) and not with an unknown type `T`.
We need some way to convert any type `T` into a string. That process is called serialization.

Although there are many good serialization tools available for C# and Unity, we'll use the built-in `JsonUtility` class.
Using the method `JsonUtility.ToJson`, we can convert a lot of types (not every type, there are restrictions) to a string. Let's do that.

{% highlight cs %}
using System.IO;

public class StandaloneStorageSystem : IStorageSystem {
  public const string SAVE_FOLDER = "Saves";

  public StandaloneStorageSystem() {
    // create the saves folder if it doesn't exist
    Directory.Create(SAVE_FOLDER);
  }

  public void Set<T>(string key, T value) {
    string filePath = Path.Combine(SAVE_FOLDER, key);
    
    string serializedValue = JsonUtility.ToJson(value);

    File.WriteAllText(filePath, serializedValue);
  }
}
{% endhighlight %}