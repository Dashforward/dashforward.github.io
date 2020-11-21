---
layout: post
title: Writing android plugins for Unity
subtitle: 
comments: false
---

{% assign imgurl = "{{site.baseurl}}/assets/img/2020-11-20-writing-android-plugins/" %}

# Part 1: Hello World

## Writing a library

Create a new empty android project with no activity within Android Studio

<img src="{{imgurl}}writing-android-plugin-001.png">

Note that you can select project or package view with the above top left project view panel as well as rick-clicking to get project settings.

1) Right click on the project and create a new module, select android library

<img src="{{imgurl}}writing-android-plugin-002.png">

2) Remove the old app plugin by applying the following steps

    * Right click on the Project and select “Open module settings”
    * Select the module you want to delete and press the minus button above.
    * Apply changes

3) Write the plugin in java

```java
package com.example.unityplugin;
import android.util.Log;

public class UnityPluginExample {
  private static String TAG = "SampleUnityPlugin";

  public void _ConsoleLog(String message) {
    Log.d(TAG, message);
  }
}
```

## Copying the library

Build the library and copy it to your Unity project.

To copy the library to your unity project every time your recompile, add the following to the `build.gradle` inside the library folder:
Here we use a path relative to our project.

```gradle
task copyPlugin(type: Copy) {
  dependsOn assemble
  from('build/outputs/aar')
  into('../../../../Assets/Plugins/Android')
  include(project.name + '-release.aar')
}
```

<img src="{{imgurl}}writing-android-plugin-003.png">


## Calling from C#

Create an AndroidJavaObject and call a method on it, this will instantiate an object based on the class you declared in java.

```csharp
public void NativeConsoleLog(string message) {
  var nativePlugin = new AndroidJavaObject("com.example.unitylib.UnityPluginExample");
  nativePlugin.Call("_ConsoleLog", message);
}
```