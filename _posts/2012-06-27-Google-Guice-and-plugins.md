---
layout: post
title: Google Guice and plugins
---

While refactoring an application to use Google Guice for dependency injection, I ran across some challenges on how to modularize my application with plugins. It turns out that these is a relatively simple solution: Guice's Multibinders.

Lets look at the problem first. Guice favors constructor injection and that often results in big, ugly constructors with dozens of arguments. In my experience this mostly happens when constructing GUI containers with a lot of "static" items (menus, tabs, lists of configuration pages, etc). These lists of items are not truly static because we would like to be able to extend them through plugins, for example.

When we follow a naive approach, a constructor for a JFrame that holds a bunch of tabs may look like this:

```java
@Inject
public MainFrame(
	Panel1 p1, Panel2 p2, Panel3 p3, Panel4 p4,
	Panel5 p5, Panel6 p6, Panel7 p7, Panel8 p8)
{
	addTab(p1);
	addTab(p2);
	addTab(p3);
	addTab(p4);
	addTab(p5);
	addTab(p6);
	addTab(p7);
	addTab(p8);
}
```

This has the advantage that it's perfectly clear which tabs this frame will host and in which order they will appear. Aside from that, it would be easy to add some code to listen to application events and enable and disable specific tabs when required. This code does lack a few things though: It's a lot of work to add tabs to it and it doesn't allow for adding more tabs after the application has been distributed (plugins).

To allow for adding tabs we need to solve several problems:

* Allow for addition of tabs
* Allow tab order to be determined by plugins or the user
* Provide some way for tabs to enable/disable based on application state

Lets start one at a time and first make the list of tabs dynamic from the point of view of the frame that will host them:

```java
@Inject
public MainFrame(Set<JComponent> panels)
{
	for(JComponent panel: panels)
	{
		addTab(panel);
	}
}
```

Now we are not injecting each tab separately, but we inject instead a Set of tabs. This removes the requirement for the frame to know exactly which tabs exist and makes the code somewhat simpler. We still have to specify which tabs go in this frame though. This is done in the Guice Module:

```java
public class UIModule extends AbstractModule {

	@Override
	protected void configure() {
		Multibinder<JComponent> panels = Multibinder.newSetBinder(binder(), JComponent.class);
		panels.addBinding().to(Panel1.class);
		panels.addBinding().to(Panel2.class);
		panels.addBinding().to(Panel3.class);
		panels.addBinding().to(Panel4.class);
		panels.addBinding().to(Panel5.class);
		panels.addBinding().to(Panel6.class);
		panels.addBinding().to(Panel7.class);
		panels.addBinding().to(Panel8.class);
	}
}
```

So what has really changed? We moved the list of panels from one piece of Java code to another piece of Java code. It's still Java code and this didn't help one bit to implement a plugin architecture.

This is where the multi-module aspect of the Multibinder comes in. When a different Guice module defines a binder for the same type (in this case `Set<JComponent>`), they will be merged. At runtime, there will only be one `Set<JComponent>` and all tabs from all Modules will be presented in the frame.

`Set<JComponent>` is a little to generic of a type to be using for this purpose though. We want to do more with this. We need (at a minimum) to specify a name and a way to get a reference to the actual `JComponent` so lets start by defining an interface:

```java
public interface UITab {
	public String getUITabName();
	public JComponent getUITabComponent();
}
```

This interface defines a few methods we'll need to display our tabs. `getUITabName` should return the display name for the tab and `getUITabComponent` is expected to return a reference to the actual UI component.

Each Panel class can now implement this interface, like so:

```java
public class Panel1 extends JPanel implements UITab {
	public String getUITabName() {
		return "Panel 1";
	}

	public JComponent getUITabComponent() {
		return this;
	}
}
```

Each Panel can now be defined in its own module like this:

```java
public class Panel1Module extends AbstractModule {
	@Override
	protected void configure() {
		Multibinder<UITab> panels = Multibinder.newSetBinder(binder(), UITab.class);
		panels.addBinding().to(Panel1.class);
	}
}
```

Now that each of the panels is defined in its own module, we can split them out into separate projects (plugins). Each Panel can get its own jar file. All we still need is a way for the main application to detect the available plugins and load them. The [ServiceLoader](http://docs.oracle.com/javase/6/docs/api/java/util/ServiceLoader.html) class is a good possibility. Some classpath detection of an appliction specific plugin descriptor or an XML file listing all installed plugins are also possible.