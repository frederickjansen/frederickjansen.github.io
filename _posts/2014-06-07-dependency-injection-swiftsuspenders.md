---
layout: post
title: Dependency injection tutorial using SwiftSuspenders (or Robotlegs)
excerpt: A look at the various ways of dependency injection in SwiftSuspenders.
---

There are various ways to handle dependency injection with SwiftSuspenders, some of them new to 2.0, some that have been around since 1.x. When looking at the documentation however, it wasn't clear to me what the recommended way of working is, and what has been deprecated. The documentation of 2.0 is quite poor and [warns you](https://github.com/robotlegs/swiftsuspenders/blob/master/README.md) not to read the heavily outdated 1.x docs. Then you come across a [blog post](http://tillschneidereit.net/2011/02/05/swiftsuspenders-1-6-a-tale-of-small-changes-and-big-plans/) about upcoming changes in 2.0 and after trying to use one-way binding, you discover in a closed [Github ticket](https://github.com/tschneidereit/SwiftSuspenders/issues/44) that it wasn't implemented after all. So this post will be a short tutorial on the various ways you can handle dependency injection in SwiftSuspenders 2.1 (which is what Robotlegs 2.2 is using).

----

All of the examples below have the same outcome (apart from the final one), but there are some important distinctions in how they allow you to access injected properties afterwards.

## Properties

This is the basic way and what you see in most tutorials, injecting into public properties. The problem here is that you may not want your properties to be public. Especially if you're working with multiple people, they might end up accessing code in a way that you never intended. In my opinion, this produces the most readable code though.
SwiftSuspenders 2.0 added optional injection through metadata, which you can set using the *optional* attribute.

{% highlight as3 %}
package {
	class Injectee {

		[Inject]
		public var foo:Clazz;

		[Inject(optional=true, name="bar")]
		public var bar:Clazz;
	}
}
{% endhighlight %}


## Methods

Methods are the second of your options. You can inject into multiple methods, with one or more parameters. Since the properties are private, they can't be accessed from the outside and users won't receive any code completion from their IDE. Readability could be better though.
Here you can handle optional injection using default parameters.

{% highlight as3 %}
package {
	class Injectee {

		private var _foo:Clazz;
		private var _bar:Clazz;

		[Inject(name="", name="bar")]
		public function setDependencies( foo:Clazz, bar:Clazz = null ):void
		{
			_foo = foo;
			_bar = bar;
		}
	}
}
{% endhighlight %}

## Setters

Next we have setters, falling somewhere in between the readability of properties and the access limitation of methods. Your IDE will still give you code completion and allows you to write to properties, but not read them.

{% highlight as3 %}
package {
	class Injectee {

		private var _foo:Clazz;
		private var _bar:Clazz;

		[Inject]
		public function set foo( foo:Clazz ):void
		{
			_foo = foo;
		}

		[Inject(name="bar")]
		public function set bar( bar:Clazz = null ):void
		{
			_bar = bar;
		}
	}
}
{% endhighlight %}

## Constructor

Finally there's constructor injection. Make note that the [Inject] tag is set at the class level. If you have a limited set of injections, this option is worth considering.

{% highlight as3 %}
package {
	[Inject(name="", name="bar")]
	class Injectee {

		private var _foo:Clazz;
		private var _bar:Clazz;

		public function Injectee( foo:Clazz, bar:Clazz = null ):void
		{
			_foo = foo;
			_bar = bar;
		}
	}
}
{% endhighlight %}

## Constructor without metadata

This one is different from all other cases in that it using no metadata. You can simply inject into a constructor by instantiating a class using the Injector. In this case there's no named injection of course. I personally wouldn't use this, as figuring out where the data is coming from can get confusing.

{% highlight as3 %}
var injector:Injector = new Injector();
injector.map( Clazz );
var injectee:Injectee = injector.instantiateUnmapped( Injectee );
{% endhighlight %}

{% highlight as3 %}
package {
	class Injectee {

		private var _foo:Clazz;
		private var _bar:Clazz;

		public function Injectee( foo:Clazz, bar:Clazz = null ):void
		{
			_foo = foo;
			_bar = bar;
		}
	}
}
{% endhighlight %}
