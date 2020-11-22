---
layout: post
title: Unity - Search suggestion window using UIElements
subtitle: Search and filter popup for textfields
comments: false
---

{% assign imgurl = "/assets/img/posts/2020-11-22-search-suggestion-window/" %}

## Design

We want to have text suggestion for our text field, so we can pre-fill the content.
We will use `UIElements` for that.
We will display an extra window with a search bar and a list of items to select:
This article was made using `Unity 2020.2` which is in beta at the time of writing.

<img src="{{site.baseurl}}{{imgurl}}search-suggestion-window-001.png">

## Creating the test window

Create a new C# script for our editor window.
Using UI elements, we can easily create a new text field:

```csharp
using UnityEditor;
using UnityEngine;
using UnityEngine.UIElements;

public class TestWindow : EditorWindow {

	[MenuItem("Tools/Test Window")]
	public static void GetWindow() => GetWindow<TestWindow>();

	public void OnEnable() {
		var root = rootVisualElement;

		var textField = new TextField("Platform:");
		root.Add(textField);
	}
}
```

<img src="{{site.baseurl}}{{imgurl}}search-suggestion-window-002.png">


## Creating the search suggestion window

<img src="{{site.baseurl}}{{imgurl}}search-suggestion-window-003.png">

Right-click on your project and create a new editor window (under UI toolkit), or create a `C# script`, a `stylesheet` (`uss` styling) and a `UIDocument` (`uxml` structure) for our new window.

Create the 3 assets:
<img src="{{site.baseurl}}{{imgurl}}search-suggestion-window-004.png">

To make sure it displays properly, you'll also make a `GetWindow` method. (It might be prefilled by unity if you created a new editor window, feel free to replace what was prefilled).

Our C# class:

```csharp
using UnityEditor;
using UnityEngine;
using UnityEngine.UIElements;

public class SearchAndSelectWindow : EditorWindow {

	[MenuItem("Tools/Search And Select Window")]
	public static void GetWindow() => GetWindow<SearchAndSelectWindow>();

	public void OnEnable() {
		var root = rootVisualElement;

		//load and add the uxml file
		var visualTree = AssetDatabase.LoadAssetAtPath<VisualTreeAsset>("Assets/Scripts/Editor/SearchAndSelectWindow.uxml");
		VisualElement uxmlFileRoot = visualTree.Instantiate();
		root.Add(uxmlFileRoot);

		//load and add the uss stylesheet
		var styleSheet = AssetDatabase.LoadAssetAtPath<StyleSheet>("Assets/Scripts/Editor/SearchAndSelectWindow.uss");
		root.styleSheets.Add(styleSheet);
	}

}
```

In the uxml file, we use `ToolbarSearchField`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<engine:UXML
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:engine="UnityEngine.UIElements"
    xmlns:editor="UnityEditor.UIElements"
    xsi:noNamespaceSchemaLocation="../../../../UIElementsSchema/UIElements.xsd">
	<editor:ToolbarSearchField text="SearchField" class="search-field" />
</engine:UXML>
```

You can keep the uss style document empty for now.
Open the window to make sure things display properly:

<img src="{{site.baseurl}}{{imgurl}}search-suggestion-window-005.png">



