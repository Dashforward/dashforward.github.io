---
layout: post
title: Unity - Graph View basics
subtitle: Simple graph view, creating nodes and ports
comments: false
---

{% assign imgurl = "/assets/img/posts/2020-11-24-simple-graph-view/" %}

In this article we will build a simple `GraphView` using `UIElements`. Note that the API is still under an experimental namespace...
This was made using `Unity 2020.2` which is in beta at the time of writing.

<img src="{{site.baseurl}}{{imgurl}}simple-graph-view-000.gif">

## Creating a GraphView

A `GraphView` is a `VisualElement`, the only difference is that it is declared in `UnityEditor.Experimental.GraphView` for now.


```csharp
using UnityEngine;
using UnityEngine.UIElements;
using UnityEditor.Experimental.GraphView;

public class SimpleGraphView : GraphView {

	private int m_NodeCount;
	private readonly Vector2 m_NodeSize = new Vector2(200, 200);

	public SimpleGraphView() {
		//a test node
		CreateNode(Vector2.zero);
	}

	private Node CreateNode(Vector2 position) {
		m_NodeCount++;
		
		var node = new Node() {
			title = $"Node {m_NodeCount}"
		};

		node.RefreshExpandedState();
		node.RefreshPorts();

		node.SetPosition(new Rect(position, m_NodeSize));

		AddElement(node);

		return node;
	}
	
}
```

Now create a test window to see our `GraphView`. `StretchToParentSize` is useful to for the GraphView to fill the entire `EditorWindow`.

```csharp
using UnityEditor;
using UnityEngine;
using UnityEngine.UIElements;

public class TestWindow : EditorWindow {

	[MenuItem("Tools/Test Window")]
	public static void GetWindow() => GetWindow<TestWindow>();

	public void OnEnable() {
		var root = rootVisualElement;

		var graphView = new SimpleGraphView();
		graphView.StretchToParentSize();
		root.Add(graphView);
	}

}
```

Now we have the most simple `GraphView`

<img src="{{site.baseurl}}{{imgurl}}simple-graph-view-001.png">

## Adding manipulators

To add interactions to the `GraphView`, we just need to `manipulators`, they are built-in `UIElements` 

```csharp
public SimpleGraphView() {
	this.AddManipulator(new ContentDragger());
	this.AddManipulator(new SelectionDragger());
	this.AddManipulator(new RectangleSelector());

	//a test node
	CreateNode(Vector2.zero);
}
```

<img src="{{site.baseurl}}{{imgurl}}simple-graph-view-002.gif">

## Right-click menu to create a node

If you right click somewhere on the graph, you'll notice a contextual menu.
In our `SimpleGraphView` class, we will override this menu to add a create node functionality

All we need is override `BuildContextualMenu` and add a "Create/Node" action:

```csharp
public override void BuildContextualMenu(ContextualMenuPopulateEvent evt) {
	base.BuildContextualMenu(evt);
	if (evt.target is GraphView) {
		evt.menu.AppendSeparator();
		System.Action<DropdownMenuAction> createNodeAction = (DropdownMenuAction dropdownMenuAction) => {
			CreateNode(dropdownMenuAction.eventInfo.mousePosition); //use current mouse position for the node position
		}; 
		evt.menu.AppendAction("Create/Node", createNodeAction);
	}
}
```

<img src="{{site.baseurl}}{{imgurl}}simple-graph-view-003.gif">

There is one caveat though, which is the position will be incorrect after we dragged the graph around. I did not find a solution for this yet as the API remains not well documented.

## Right-click menu to create a port

Let's add a a way to create input/output ports so we can connect our nodes.

First, we need to override the `GetCompatiblePorts` method, to keep things simple we will just return all ports.

```csharp
public override List<Port> GetCompatiblePorts(Port startPort, NodeAdapter nodeAdapter) {
	return ports.ToList();
}
```

Then, we need a method to create ports on a target node.
The direction can be `Direction.Input` or `Direction.Output`, the `Port.Capacity` allow us to connect multiple or just one other port to our port.
We also need to call refresh methods we can see the ports after we add them.

```csharp
private void CreatePort(Node targetNode, Direction direction, Port.Capacity capacity=Port.Capacity.Single) {
	var port = targetNode.InstantiatePort(Orientation.Horizontal, direction, capacity, typeof(float));
	port.portName = (direction == Direction.Input) ? "in" : "out";
	port.source = "";
	
	if (direction == Direction.Input) {
		targetNode.inputContainer.Add(port);
	} else {
		targetNode.outputContainer.Add(port);
	}

	targetNode.RefreshExpandedState();
	targetNode.RefreshPorts();
}
```

Finally, let's add to our contextual menu if we click on a node to create either an input port or an output port

```csharp
public override void BuildContextualMenu(ContextualMenuPopulateEvent evt) {
	base.BuildContextualMenu(evt);
	if (evt.target is GraphView) {
		evt.menu.AppendSeparator();
		System.Action<DropdownMenuAction> createNodeAction = (DropdownMenuAction dropdownMenuAction) => {

			CreateNode(dropdownMenuAction.eventInfo.mousePosition);
		}; 
		evt.menu.AppendAction("Create/Node", createNodeAction);
	} else if (evt.target is Node) {
		var targetNode = evt.target as Node;
		evt.menu.AppendSeparator();

		System.Action<DropdownMenuAction> createInputPortAction = (DropdownMenuAction dropdownMenuAction) => {
			CreatePort(targetNode, Direction.Input);
		}; 
		evt.menu.AppendAction("Create/Input port", createInputPortAction);

		System.Action<DropdownMenuAction> createOutputPortAction = (DropdownMenuAction dropdownMenuAction) => {
			CreatePort(targetNode, Direction.Output, Port.Capacity.Multi);

		}; 
		evt.menu.AppendAction("Create/Output port", createOutputPortAction);
	}
}
```

<img src="{{site.baseurl}}{{imgurl}}simple-graph-view-000.gif">