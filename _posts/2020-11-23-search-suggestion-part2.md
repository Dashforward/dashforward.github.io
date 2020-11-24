---
layout: post
title: Unity - Search suggestion window (part 2)
subtitle: Search and filter popup for list view using UIElements
comments: false
---

{% assign imgurl = "/assets/img/posts/2020-11-23-search-suggestion-part2/" %}

Building on top of the previous article covering a dropdown window to search suggested text, we will now add the selected option to a list. Removing the current elements out of the suggestion list.
For this we will need to cover more `ListView` setup with custom rows as well as the basics of setting our own `VisualElement`.
This article was made using `Unity 2020.2` which is in beta at the time of writing.

<img src="{{site.baseurl}}{{imgurl}}search-suggestion-part2.gif">

## Creating a custom VisualElement

For now we will start with a very simple `VisualElement` with only one button. We need a parameterless constructor so that the class can be used in uxml.

```csharp
public class SearchAndSelectList : VisualElement { 

	public new class UxmlFactory : UxmlFactory<SearchAndSelectList, UxmlTraits> { }

	public SearchAndSelectList() {
		var button = new Button();
		button.text = "+";
		button.style.width = 20;
		Add(button);
	}

}
```

The following code is what makes us override the default factor so we can declare the `VisualElement` in xml:

```csharp
public new class UxmlFactory : UxmlFactory<SearchAndSelectList, UxmlTraits> { }
```

A basic `uxml` file would like this:

```xml
<?xml version="1.0" encoding="utf-8"?>
<engine:UXML
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:engine="UnityEngine.UIElements"
    xmlns:editor="UnityEditor.UIElements"
    xsi:noNamespaceSchemaLocation="../../../../UIElementsSchema/UIElements.xsd">
	<SearchAndSelectList/>
</engine:UXML>
```

Call the above code `TestWindow.uxml`.

As usual in an `EditorWindow`, we will test our code:

```csharp
using System;
using System.Collections.Generic;
using UnityEditor;
using UnityEngine;
using UnityEngine.UIElements;

public class TestWindow : EditorWindow {

	[MenuItem("Tools/Test Window")]
	public static void GetWindow() => GetWindow<TestWindow>();

	public void OnEnable() {

		VisualElement root = rootVisualElement;

		var visualTree = AssetDatabase.LoadAssetAtPath<VisualTreeAsset>("Assets/Scripts/Editor/TestWindow.uxml");
		VisualElement rootFromUxml = visualTree.Instantiate();
		root.Add(rootFromUxml);

		var searchAndSelectList = root.Q<SearchAndSelectList>();
		searchAndSelectList.style.marginLeft = 5;
		searchAndSelectList.style.marginRight = 5;

	}

}
```

Alternatively, we could have instantiated a `SearchAndSelectList` and added it to our window as follow:

```csharp
var searchAndSelectList = new SearchAndSelectList();
root.Add(searchAndSelectList);
```

<img src="{{site.baseurl}}{{imgurl}}search-suggestion-part2-001.png">


## Listview with customized rows

We go back to our `SearchAndSelectList`, we will keep our parameterless constructor, but add an override to instantiate with a list:

```csharp
public class SearchAndSelectList : VisualElement { 
	private List<string> m_SelectedValues; //what we display in our listview

	private List<string> m_OptionsSource; //list of suggestions we can search form
	public List<string> OptionsSource {
		get {
			return m_OptionsSource;
		}
		set {
			m_OptionsSource = value;
			if (m_SelectedValues != null) {
				m_SelectedValues.Clear();
			}
		}
	}

	public new class UxmlFactory : UxmlFactory<SearchAndSelectList, UxmlTraits> { }

	public SearchAndSelectList() : this(new List<string>()) {}

	public SearchAndSelectList(List<string> values) {
		m_OptionsSource = values;
		m_SelectedValues = new List<string>();

		var button = new Button();
		button.text = "+";
		button.style.width = 20;
		Add(button);
	}

}
```

Now it is time to construct our listview, each row is pretty simple, it just has properly aligned text and a delete button.

<img src="{{site.baseurl}}{{imgurl}}search-suggestion-part2-002.png">

```csharp
public SearchAndSelectList(List<string> values) {
	/* ... */

	//it is necessary to have both an height and and a row height (item height) for the listview to display
	//we will cover changing that height dynamically in another article
	var listView = new ListView();
	listView.itemHeight = 150;
	listView.style.height = 20;

	Func<VisualElement> makeItem = () => {
		//each item has an horizontal container 
		var itemContainer = new VisualElement();
		itemContainer.style.flexDirection = FlexDirection.Row; //horizontal
		itemContainer.style.justifyContent = Justify.SpaceBetween; //expand space between items

		var label = new Label();
		label.style.unityTextAlign = TextAnchor.MiddleCenter; //center text
		itemContainer.Add(label);

		//delete button
		var button = new Button();
		button.text = "-";

		itemContainer.Add(button);

		return itemContainer;
	};

	Action<VisualElement, int> bindItem = (e, i) => {
		e.Q<Label>().text = m_SelectedValues[i];

		//every time we bind an item to some data, we set the proper functionality for our delete button
		var deleteButton = e.Q<Button>();
		if (deleteButton != null) {
			deleteButton.clickable.clicked += () => {
				//moves the value out off our selection list and back to our suggestion list
				m_OptionsSource.Add(m_SelectedValues[i]);
				m_SelectedValues.RemoveAt(i);
				listView.Refresh();
			};
		}
	};

	listView.showAlternatingRowBackgrounds = AlternatingRowBackground.ContentOnly;
	listView.makeItem = makeItem;
	listView.bindItem = bindItem;
	listView.itemsSource = m_SelectedValues;
	listView.selectionType = SelectionType.None;
	Add(listView);

}
```

## Opening our suggestion window

We use `SearchAndSelectWindow.Show` to show a suggestion window using our options:

See the previous article on how to setup this window.
We register a click callback for our add button.
When users select an entry from the  `SearchAndSelectWindow`, we add to our current `listview` data source (the `m_SelectedValues` collection) and refresh the listview.

```csharp
public SearchAndSelectList(List<string> values) {
	/* ... */

	var button = new Button();
	button.text = "+";
	button.style.width = 20;
	button.RegisterCallback<ClickEvent>((evt) => {
		System.Action<string> onComplete = (string text) => {
			m_SelectedValues.Add(text);
			m_OptionsSource.Remove(text);
			listView.Refresh();
		};

		m_OptionsSource.Sort();

		//note: give the full visual element's width, otherwise the suggestion window would be as large as our '+' button
		var rect = new Rect(button.worldBound.x, button.worldBound.y, this.worldBound.width, button.worldBound.height);
		SearchAndSelectWindow.Show(rect, m_OptionsSource, onComplete);
	});

	Add(button);
}
```

## Bind some test values

To finish, we just have to set `SearchAndSelectList.OptionsSource` from the test `EditorWindow` we created earlier:

```csharp
using System;
using System.Collections.Generic;
using UnityEditor;
using UnityEngine;
using UnityEngine.UIElements;

public class TestWindow : EditorWindow {

	[MenuItem("Tools/Test Window")]
	public static void GetWindow() => GetWindow<TestWindow>();

	public void OnEnable() {

		VisualElement root = rootVisualElement;

		var visualTree = AssetDatabase.LoadAssetAtPath<VisualTreeAsset>("Assets/Scripts/Editor/TestWindow.uxml");
		VisualElement rootFromUxml = visualTree.Instantiate();
		root.Add(rootFromUxml);

		var searchAndSelectList = root.Q<SearchAndSelectList>();
		searchAndSelectList.style.marginLeft = 5;
		searchAndSelectList.style.marginRight = 5;

		//create some test values and set them
		var values = new List<string>(Enum.GetNames(typeof(RuntimePlatform)));
		searchAndSelectList.OptionsSource = values;
	}

}
```

