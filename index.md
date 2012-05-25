---
layout: master
title: Intravenous
---
### What? Why?
If you are building complex applications it can become very difficult to manage object lifetimes and dependencies. For instance, if you're building a UI you may want to create a dialog. The dialog's model may have dependencies on a logger, Growlesque notification thing, etc. It'll probably also contain some sub-models to display inside the dialog. You need to track all these dependencies and when the dialog closes you want to dispose some of the models (but not all, since the logger is probably a singleton) and so on... Complex! Unless you use an IoC container like this one.

With *Intravenous* you create a _container_ and tell this container which services are available in your system. Then, when you have class that wants to use these services you simply list them as dependencies. You tell the container to create an instance of your class and all dependencies will be automatically resolved (even if your depedencies have dependencies of their own).

You can also create something called a _nested container_. In case of the dialog sample you would use this nested container to create all the view models that are only relevant during the dialog's lifetime. When the dialog closes, you can dispose the nested container and automatically all of the objects created by that container will be disposed as well. Still sounds complex? Read on!

### How do I include it?
It can be loaded as an CommonJS/Node.JS or AMD module. If you don't have either, it will be available as `window.intravenous`. It has no dependencies on other libraries.

### How do I use it?
1. Create a container like this:
	{% highlight javascript %}
var container = intravenous.create();
{% endhighlight %}
1. Next, register some services (can be a constructor function or just an object):
	{% highlight javascript %}
container.register("logger", loggerClass);
container.register("someGlobalData", { data: "hello" });
{% endhighlight %}
1. Then, define a class and its dependencies
	{% highlight javascript %}
var myClass = function(logger, someGlobalData) {
  /* use logger here */
};
myClass.$inject = ["logger", "someGlobalData";
container.register("myClass", myClass);
{% endhighlight %}
1. Finally, get an instance to this class through the container:
	{% highlight javascript %}
var myInstance = container.get("myClass");
{% endhighlight %}

You will now have an instance of `myClass` with all its dependencies resolved.

### What if a service doesn't exist?
It will throw an exception. Alternatively, you can specify optional (or nullable) dependencies by using the `?` suffix, like so:

{% highlight javascript %}
myClass.$inject = ["logger", "optionalDependency?"];
{% endhighlight %}

In this case `optionalDependency` will be injected as `null` if it doesn't exist.

### How can I control object disposal?
Pass in an `onDispose` handler when you create the container:
{% highlight javascript %}
var container = intravenous.create({
  onDispose: function(obj, serviceName) {
    obj.yourDisposeFunction();
  }
});
{% endhighlight %}

Now, whenever you are done with your container, call `dispose` on the container and it will call your `onDispose` callback for every object that needs to be disposed.

### How can I dynamically create new instances of a service?
You can have Intravenous create a factory for you by using the `!` or `Factory` suffix, like so:

{% highlight javascript %}
var widget = function() {
};
widget.$inject = ["foo"];
container.register("widget", widget);

var myClass = function(widgetFactory) {
	this.myWidget = widgetFactory.get();
	/* ... */
	widgetFactory.dispose(this.myWidget);
};
myClass.$inject = ["widgetFactory"];
container.register("myClass", myClass);

/*
	You can now do something like:
	var instance = container.get("myClass");
*/	
{% endhighlight %}

All transient (i.e. non-singleton) dependencies that `myWidget` needs will also be disposed when you dispose `myWidget`. *This is a very powerful feature!* It means that if you are dynamically creating these widgets, the lifetimes of all the widget's dependencies are automatically scoped to the lifetime of the widget!

Additionally, it is very easy to mock this factory when you are doing unit testing.

### When using the factory, can I override dependencies?
Yes, let's say the widget (in the above example) also has a dependency on the `foo` service. If you want to override the `foo` service, you use the `use` syntax:

{% highlight javascript %}
var widget = function(foo) {
	this.foo = foo;
};
widget.$inject = ["foo"];
container.register("widget", widget);

var myClass = function(widgetFactory) {
	this.myWidget = widgetFactory.use("foo", "value1").get();
	this.myWidget2 = widgetFactory.use("foo", fooClass).get();
	/*
		this.myWidget.foo will now be "value1"
		this.myWidget2.foo will now be an instance of "fooClass"
	*/
};
container.register("myClass", myClass);
{% endhighlight %}

The syntax is chainable, so you can override as many dependencies as you like.

### How can I control the lifecycle of a service?
When registering a service using `register` it will default to the `perRequest` lifecycle. There are a number of different lifecycles you can use, though. They are listed below.

Let's say you have a service called `foo` and you are resolving an object `bar` that needs to resolve a large object graph to satisfy all its dependencies. It may mean `foo` is required in a lot of different places.

The available lifecycles:

1. `perRequest`: Regardless of how many times `foo` is required in the object graph, it is only created once. The next call to `container.get` will create a new `foo`, though.

2. `unique`: Every time `foo` is required in the object graph, a new instance is created.

3. `singleton`: As long as the container is not disposed, `foo` will only be created once. It will also be reused across multiple calls to `container.get`.

The lifecycle is specified as the third argument to `register`:
{% highlight javascript %}
container.register("logger", loggerClass, "singleton");
{% endhighlight %}

### How can I specify additional arguments?
When you want to pass additional to your class, simply add them to the `container.get` call:

{% highlight javascript %}
var myClass = function(logger, extra) {
  alert(extra);
};
myClass.$inject = ["logger"];
container.register("myClass", myClass);
var myInstance = container.get("myClass", "hello!");
{% endhighlight %}

This example will alert `hello!`.

### Can I also create instances of services I haven't yet registered?
No. You explicitly need to register all services you need.

### How can I get access to the main intravenous container?
First, make sure you're doing this for the right reasons. Using the container as a service locator is an [anti-pattern](http://blog.ploeh.dk/2010/02/03/ServiceLocatorIsAnAntiPattern.aspx).

If you really want to, take a dependency on the service called `container`, like so:

{% highlight javascript %}
var myClass = function(container) {
  var nested = container.create();
}
myClass.$inject = ["container", /* ... other dependencies */];
{% endhighlight %}

### What if I want to dispose only parts of the container, instead of everything?
Use a nested container and dispose that instead:
{% highlight javascript %}
var container = intravenous.create(/* onDispose handler here */);
var nested = container.create();
var myInstance = nested.get("myClass");
nested.dispose();
{% endhighlight %}

You can also register additional services on the nested container. They will override services registered on the parent container.

Creating a nested container is very lightweight so it's highly recommended to use them.

Please note that `container.create()` is not the same as `intravenous.create()`. The first creates a nested container, the second creates a completely new intravenous container. Typically in your application you will only need a single intravenous container.

### Where did you get inspiration from?
The `$inject` syntax was inspired by [AngularJS](http://angularjs.org/). The nested container was a good idea pilfered from [StructureMap](http://www.structuremap.net). The person making sense of DI in general is [Mark Seemann](http://blog.ploeh.dk).

### What's the license?
[MIT](http://www.opensource.org/licenses/mit-license.php).

### I want to report bugs or submit pull requests!
Use the [GitHub](http://github.com/RoyJacobs/intravenous) page.

Join the discussions in the [Google Group](https://groups.google.com/d/forum/intravenousjs) as well.