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

<img src="{{site.baseurl}}{{imgurl}}search-suggestion-window-000.gif">

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
		Init();
	}

	private void Init() {
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

## Adding a list view

Add the list view in our uxml file, after the search field

```xml
<editor:ToolbarSearchField text="SearchField" class="search-field" />
<engine:ListView class="search-listview" />
```

Now we can fill the stylesheet to give a height for our list view rows:

```css
.search-listview {
    --unity-item-height: 16;
}
```

In `SearchAndSelectWindow`, 

First, let's add two fields, we need two lists to keep track of the full list of possible options and our actual filtered result.
We also keep track of the height as it will be used for the window height too:

```csharp
public class SearchAndSelectWindow : EditorWindow {
	private List<string> m_SearchOptions;
	private List<string> m_FilteredSearchOptions;
	private const float m_Height = 320.0f;

	/* ... */
}
```

In `OnEnable`, we will create some dummy data for testing

```csharp
public void OnEnable() {
	//dummy data, this line will be removed in the future
	var searchOptions = new List<string>(Enum.GetNames(typeof(RuntimePlatform)));
	Init(searchOptions);
}
```

Then, after we load the stylesheet, we will setup our list view

```csharp
private void Init(List<string> searchOptions) {
	m_SearchOptions = searchOptions;
	//shallow copy
	m_FilteredSearchOptions = m_SearchOptions.GetRange(0, m_SearchOptions.Count);

	var root = rootVisualElement;
	var visualTree = AssetDatabase.LoadAssetAtPath<VisualTreeAsset>("Assets/Scripts/Editor/SearchAndSelectWindow.uxml");
	VisualElement labelFromUXML = visualTree.Instantiate();
	root.Add(labelFromUXML);

	var styleSheet = AssetDatabase.LoadAssetAtPath<StyleSheet>("Assets/Scripts/Editor/SearchAndSelectWindow.uss");
	root.styleSheets.Add(styleSheet);

	//rows are rendered with a label
	Func<VisualElement> makeItem = () => new Label();

	//bindItem sets the correct value for a given row
	//we use our filtered list for that
	Action<VisualElement, int> bindItem = (e, i) => {
		(e as Label).text = m_FilteredSearchOptions[i];
	};

	//retrieve our list from the uxml file
	var listView = root.Q<ListView>();
	//alternate the background color of our rows so it is more readable
	listView.showAlternatingRowBackgrounds = AlternatingRowBackground.ContentOnly;
	listView.makeItem = makeItem;
	listView.bindItem = bindItem;
	//our filtered list is the source
	listView.itemsSource = m_FilteredSearchOptions;
	//force to select only one row at a time
	listView.selectionType = SelectionType.Single;
	listView.style.height = m_Height;

	m_IsInitialized = true;
}
```

<img src="{{site.baseurl}}{{imgurl}}search-suggestion-window-006.png">

## Filtering content in the listview using the search field

In `SearchAndSelectWindow.cs`, we will add search field interactions at the end of our `Init` method. `ToolbarSearchField` comes from the `UnityEditor.UIElements` namespace:

```csharp
/* ... */
listView.style.height = m_Height;

var searchField = root.Q<ToolbarSearchField>();
searchField.RegisterCallback<ChangeEvent<string>>(evt => {
	UpdateList(listView, evt.newValue);
});
```


Then add the `UpdateList` method, which will update the filtered options based on the current searchfield text:

```csharp
private void UpdateList(ListView listView, string searchText) {
	m_FilteredSearchOptions.Clear();

	listView.selectedIndex = -1;
	m_FilteredSearchOptions.AddRange(m_SearchOptions.Where((string text) => { return text.IndexOf(searchText, StringComparison.CurrentCultureIgnoreCase) >=0; }));

	listView.Refresh();
}
```

Now the content is filtered out based on why we type in:

<img src="{{site.baseurl}}{{imgurl}}search-suggestion-window-006.png">

## Opening the search window as a dropdown

It's now time to wire the window so it can be opened from our initial text field as a way to search for values. Here is our `TextField` again from the first window:

<img src="{{site.baseurl}}{{imgurl}}search-suggestion-window-002.png">

First, at the top of `SearchAndSelectWindow` we will add a way to create and retrieve the current window:
In order to create the editor window without the title and tab data, we use `ScriptableObject.CreateInstance`

```csharp
private static SearchAndSelectWindow _Instance;
private static SearchAndSelectWindow Instance {
	get {
		if (_Instance == null) {
			_Instance = ScriptableObject.CreateInstance<SearchAndSelectWindow>();
		}

		return _Instance;
	}
}
```

We need a method that will help us show the window at a particular position using `Rect`. In addition we know want to wire the possible values from the outside, so you can replace the old `OnEnable` method with this logic, as well as get rid of the `GetWindow` method:
We also need an `onComplete` action that will return the selected text

```csharp
private System.Action<string> m_OnCompleteAction;

public static void Show(Rect rect, List<string> searchOptions, System.Action<string> onComplete) {
	Instance.Init(rect, searchOptions, onComplete);
	Instance.Repaint();
}

private void Init(Rect rect, List<string> searchOptions, System.Action<string> onComplete) {
	Vector2 v2 = GUIUtility.GUIToScreenPoint(new Vector2(rect.x, rect.y));
	rect.x = v2.x;
	rect.y = v2.y;
	m_OnCompleteAction = onComplete;

	Init(searchOptions);

	ShowAsDropDown(rect, new Vector2(rect.width, m_Height));
	Focus();
}
```

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

## Open the suggestion window from our new TestWindow 

Now we are ready to open the search suggestion window from our initial `TestWindow`. We use `textField.worldBound` property to retrieve the position where our window should be shown.
Here is what the complete `TestWindow` looks like:

```csharp
VisualElement root = rootVisualElement;
var textField = new TextField("Platform:");
textField.RegisterCallback<ClickEvent>((evt) => {
	//open the search window here, when onComplete is called we set the textfield data
	System.Action<string> onComplete = (string text) => { textField.SetValueWithoutNotify(text); };
	SearchAndSelectWindow.Show(textField.worldBound, m_Values, onComplete);
});
root.Add(textField);
```

If we open `TestWindow` and click on the textfield, our window is now rendered in it, and clicking outside of it will close it:

<img src="{{site.baseurl}}{{imgurl}}search-suggestion-window-008.png">

We now just need to handle the selection so the text can be returned

## Returning current selection and closing the window

Our `TestWindow` already declares the `onComplete` action and will set the textfield text when called

```csharp
System.Action<string> onComplete = (string text) => { textField.SetValueWithoutNotify(text); };
SearchAndSelectWindow.Show(textField.worldBound, m_Values, onComplete);
```

In `SearchAndSelectWindow`, we need to call the callback:

We can do so at the end of our init method, when the selection of the listview has changed. We also call `Close` on the EditorWindow:

```csharp
listView.onSelectionChange += obj => {
	if (m_OnCompleteAction != null) {
		m_OnCompleteAction((string)obj.FirstOrDefault());
	}
	Close();
};
```

That's it, our initial `TextField` now gets filled with our selection.




