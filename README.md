>A [WebGL](http://docs.unity3d.com/Manual/webgl-gettingstarted.html) Unity project is just a normal HTML page running a 3 000 000 lines of code JavaScript file.

The heart of HTML pages are [links](http://va.lent.in) which can be opened either in the same window/tab or in a popup window/tab. Turns out that it's not that simple with a Unity WebGL project precisely because of one of the browser's security mechanisms.

Modern browsers open links in new tabs/windows only when they clearly see that this action resulted from a user input event, like mouse click for example. If not, Firefox displays this when a JavaScript uses ```window.open()``` code to open a new window:

![](/Images/ffnewwindow.png)

This is fine in a context of a normal web page when you click links. But in Unity actual input is processed within *Player Loop* in a moment after you clicked an object which is too late for a browser to consider this action to be originated from a user event.

So, if you use ```Application.ExternalEval("window.open('http://google.com');");``` in Unity when a mouse button is pressed this will be already too late and your window will be blocked by default.

A proper solution here would be the same described in [Unity WebGL Docs](http://docs.unity3d.com/Manual/webgl-cursorfullscreen.html) — send a message to JavaScript from Unity when a button is pressed so JavaScript would execute actual window opening code when user releases the button.

This can be done in a few ways described in Unity WebGL Docs: [Interacting With Browser Scripting](http://docs.unity3d.com/Manual/webgl-interactingwithbrowserscripting.html). But let's see how one can write a WebGL JS Plugin to do this.

A WebGL plugin is a **.jslib** file placed anywhere in your project. It's a good idea to store plugins somewhere like **/Assets/Plugins/WebGL** but it's not required anymore. New *Plugin Inspector* in Unity 5 makes life much easier:

![](/Images/plugin.png)

The contents of OpenWindow.jslib file is the following:

```
var OpenWindowPlugin = {
    openWindow: function(link)
    {
    	var url = Pointer_stringify(link);
        document.onmouseup = function()
        {
        	window.open(url);
        	document.onmouseup = null;
        }
    }
};

mergeInto(LibraryManager.library, OpenWindowPlugin);
```

It's a JS file with an object containing one function — **openWindow**. This function receives an URL from Unity and set's up **onmouseup** event for entire web page which when opens this URL. *Notice: this is just the most simple example code.*

Then, the simplest script to use this plugin would be the following:

```
using UnityEngine;
using System.Runtime.InteropServices;

public class Link : MonoBehaviour {
	public void OpenLinkJSPlugin() {
		#if !UNITY_EDITOR
		openWindow("http://unity3d.com");
		#endif
	}

	[DllImport("__Internal")]
	private static extern void openWindow(string url);
}
```

Notice ```#if !UNITY_EDITOR``` in the script — this is a preprocessor define to include this code only when executed not in Unity Editor, because this code will not work there.

The next thing is we need to code another small script to  actually execute this code on **mouseDown** event when we click an UI Button for example. The issue with Buttons is that they fire **onClick** event only when mouse button is released, which is too late for us. This means that we need another script and we need to put it on our Button element in UI hierarchy:

```
using UnityEngine;
using UnityEngine.EventSystems;
using System;
using UnityEngine.Events;

public class PressHandler : MonoBehaviour, IPointerDownHandler {
	[Serializable]
	public class ButtonPressEvent : UnityEvent { } 

	public ButtonPressEvent OnPress = new ButtonPressEvent();

	public void OnPointerDown(PointerEventData eventData) 	{
		OnPress.Invoke();
	}
}
```

Put this script on your UI Button and set **OnPress** event to call our **OpenLinkJSPlugin** function.
