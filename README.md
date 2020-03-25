# History Window Using UIElements

With [Unity 2019.1](https://blogs.unity3d.com/2019/04/16/introducing-unity-2019-1/) out and fully up and running with the new [UIElements](https://blogs.unity3d.com/2019/04/23/whats-new-with-uielements-in-2019-1/) the time has come to port all of our existing editor utilities to the new system. To learn I decided to make a simple history window (which Unity is strangely lacking) that maintains a list of past selected objects for which you can cycle through. It looks like this:

![Window](window.png)

> The History Window - made with UIElements!

Nothing too fancy although you will notice that it does look slightly different than editor windows we are used to. I'm actually using the 2019.3 alpha which has the new flat looking style which I just prefer.

## UIElements Overview

If you haven't used UIElements yet well then it's time that you should. Read [this](https://blogs.unity3d.com/2019/04/23/whats-new-with-uielements-in-2019-1/) for a full outline of what it can do. Basically, UIElements is more web based approach to UIs using a hierarchical layout, and CSS like styling. And because it is no longer immediate mode rendering it is much more efficient. It's a big change from IMGUI so that's why I started with something extremely simple that hopefully many of you will find useful as well. 

UIElements is mostly broken into three parts: the UXML which defines the hierachy of your elements, the USS which is basically just CSS that defines the look and layout of the hierarchy, and the C# which defines how the elements behave. So lets dive right in.

## Build a Hierarchy with UXML

```xml
<UXML
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns="UnityEngine.UIElements"
	xsi:schemaLocation="UnityEngine.UIElements ../../UIElementsSchema/UnityEngine.UIElements.xsd">
	<VisualElement class="horizontalRow">
		<Button name="back" class="headerButton" text="Back" />
		<Button name="forward" class="headerButton" text="Forward" />
	</VisualElement>
	<ListView class="selectionList" />
</UXML>
```

First we define a couple of namespaces (Unity does this for you if you create a UXML file in the editor). Next we have a container element which houses the forward and back buttons. These are each given a name so that they can be found in C#.
Last, we have a ListView element which is configured in C# that will display the past selections.

## Make Elements Fancy with USS

```css
.horizontalRow
{
	flex-direction: row;
	justify-content: space-between;
}

.headerButton
{
	margin: 5px;
	width: 100px;
	flex-shrink: 1;
}

.selectionList
{
	flex-grow: 1;
	flex-shrink: 0;
	flex-basis: 0;
}

.selectionItem
{
	flex-direction: row;
	flex-grow: 1;
}

.selectionLabel
{
	padding-left: 6px;
	align-self:center;
}

.selectionLabel.current
{
	-unity-font-style: bold;
}
```

The first three classes are ones that are used directly by the UXML and last three are ones that will be assigned from C#.

## Control UIs with C#

```c#
namespace PiRhoSoft
{
	[InitializeOnLoad]
	public class History : EditorWindow
	{
		private const string _styleSheetPath = "Assets/Scripts/Editor/History/History.uss";
		private const string _uxmlPath = "Assets/Scripts/Editor/History/History.uxml";

		private Button _back;
		private Button _forward;
		private ListView _listView;

		void OnEnable()
		{
			var styleSheet = AssetDatabase.LoadAssetAtPath<StyleSheet>(_styleSheetPath);
			var uxml = AssetDatabase.LoadAssetAtPath<VisualTreeAsset>(_uxmlPath);

			uxml.CloneTree(rootVisualElement);
			rootVisualElement.styleSheets.Add(styleSheet);

			_back = rootVisualElement.Query<Button>("back");
			_back.clickable.clicked += HistoryList.MoveBack;
			_back.SetEnabled(HistoryList.CanMoveBack());

			_forward = rootVisualElement.Query<Button>("forward");
			_forward.clickable.clicked += HistoryList.MoveForward;
			_forward.SetEnabled(HistoryList.CanMoveForward());

			_listView = rootVisualElement.Query<ListView>().First();
			_listView.selectionType = SelectionType.Single;
			_listView.itemsSource = HistoryList.History;
			_listView.makeItem = MakeItem;
			_listView.bindItem = BindItem;
			_listView.onItemChosen += item => Select();
			_listView.onSelectionChanged += selection => Highlight();

			Selection.selectionChanged += Refresh;
		}

		private void OnDisable()
		{
			Selection.selectionChanged -= Refresh;
		}

		private void Refresh()
		{
			_back.SetEnabled(HistoryList.CanMoveBack());
			_forward.SetEnabled(HistoryList.CanMoveForward());
			_listView.Refresh();
		}

		private VisualElement MakeItem()
		{
			var item = new VisualElement();
			item.AddToClassList("selectionItem");

			var label = new Label();
			label.AddToClassList("selectionLabel");

			item.Add(label);
			return item;
		}

		private void BindItem(VisualElement container, int index)
		{
			var label = container.ElementAt(0) as Label;
			label.text = HistoryList.GetName(index);

			if (index == HistoryList.Current)
				label.AddToClassList("current");
			else
				label.RemoveFromClassList("current");
		}

		private void Select()
		{
			if (_listView.selectedIndex != HistoryList.Current)
			{
				HistoryList.Select(_listView.selectedIndex);
				Refresh();
			}
		}

		private void Highlight()
		{
			if (_listView.selectedItem is Object[] obj && obj.Length > 0)
				EditorGUIUtility.PingObject(obj[0]);
		}
	}
}
```

This utilizes a static class (`HistoryList`) that maintains the list of objects that has been selected but we are only focused on the actual UIElements stuff for now, so let's unpack this a little.

**Make sure you change the string constants for the paths at the top of the class if you place the files in a different folder!**

### `OnEnable()`
* Load the style sheet and the uxml file from the `AssetDatabase` and apply them to the root visual container that every editor window has
* Query the hierarchy for the forward and back Buttons, register callbacks for when they are clicked on, and set them to be enabled or disabled
* Initialize the `ListView`
	* `selectionType` - We only want to be able to select a single item from the list at a time
	* `itemsSource` - The `IList` that contains the items we want to display info for
	* `makeItem` - The method that is called to create a `VisualElement` for an item in the list
	* `BindItem` - The method called to rebind elements items to the appropriate index as the list scrolls
	* `onItemChosen` - The method called when an item is double clicked
	* `onSelectionChanged` - The method called when a new item is highlighted
* Refresh the list and buttons when the selection changes

### `MakeItem()`
* Create a container for the item and give it a style class that we created in the USS

### `BindItem()`
* Set the text of the item to the `Object` that it references
* Add a style class that will make it bold based on if it is highlighted. This is a common tactic in JavaScript for webpages and it nice to see that it works here as well!

### `Select()`
* Called when an item is double clicked on in the ListView
* Tell the editor to select the Object and refresh the ListView/Buttons

### `Highlight()`
* Called when an item is focused in the list
* Tell the editor to highlight the object in the hierarchy or project view

There you have it! This only took a couple hours to make despite knowing almost nothing about UIElements beforehand which shows how useful I expect them to be for more complex stuff.

Hope somebody can find this simple utility useful!