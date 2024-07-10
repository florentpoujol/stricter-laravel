# Stricter Laravel

Laravel is a great framework but a lot lenient in many ways, which some like me doesn't find a good point.

This document aims to be a collection of tips/advices on how to use or not use some features of Laravel, why, and the alternatives that I prefer.

The goal is to still use Laravel of course, but in a way that **I find** more approachable, easier to understand, with less abstraction/indirection and more static analysis friendly.

Remember that ease of maintainability is more important than raw speed of development.

1. [General](#general)
2. [Use dependency injection whenever possible](#use-dependency-injection-whenever-possible)
3. [Do not use helper functions or facades](#do-not-use-helper-functions-or-facades)
4. [Do not use Facade aliases](do-not-use-facade-aliases)


## General

Here is some general advices/guidelines:

Do not use features that aren't statically analysable.
By statically analysable, I mean by a type checker like PHPStan.

Aims for having the least abstraction level possible.
Use services directly instead of helpers, and inject theses services instead or relying on the service locator pattern (via a helper or facade).

Aims for having one way to do each things and do not allow them to be doable.

With IDEs and AI, writing more code isn't slower and often make the intent of the code clearer (and thus probably more maintainable).
So aims to have clear intent, clear flow of information


## Use dependency injection whenever possible

At least where it's easy (like in controllers and Artisan commands), there is no reason to not use dependency injection instead of using helpers or Facades.

This helps see what are the actual dependencies of your code are (what other services your code needs to run).

If your type the dependencies against interfaces, this also may help testing by injecting a mock or fake object instead of the actual implementation.

Remember that in Laravel controllers, you can use dependency injection both in the constructor and in each methods that are target or a route.

Ie:

```php
// instead of this

class Controller
{
	public function home()
	{
		return view('home');
	}

	public function login()
	{
		$request = request();

		// do stuff...

		return redirect('home');
	}
}

// do this

class Controller
{
	public function __construct(
		// constructor injection (may helps your controller be more concise if the service is used in a lot of methods)
		// dependencies like that can often be marked as private and readonly
		private readonly ViewFactoryInterface $viewFactory, 
		private readonly Request $request, 
	) {
	}

	public function home(): ViewInterface
	{
		return $this->viewFactory->make('home');
	}

	public function redirectToHome(Redirector $redirect): Response // method injection
	{
		// $this->request

		// do stuff...

		return $redirect->to('home');
	}
}
```

Whenever you are in a service-locator situation where you must resolve a service directly from the application instance (with `Application->make()` method for instance), prefer using an interface or class FQCN instead of a string alias.
The reason is the same, it's to better see/understand what is going on, to reduce the "magic" that is the string alias.

Ie: 
```php
$container = Container::getInstance();

// instead of
$container->make('url');

// do
$container->make(\Illuminate\Routing\UrlGenerator::class);
// or with the interface
$container->make(\Illuminate\Contracts\Routing\UrlGenerator::class); // typically return an instance or \Illuminate\Routing\UrlGenerator::class (or any other custom service that has been binded to the interface)
```


## Do not use helper functions or facades

Laravel define several helper functions, accessible on the global namespace, that are shortcuts for instantiating specific classes (like `now()` that instantiate a `Carbon` object) or for using specific services (like `action` that calls the action method on the `UrlGenerator` service).

I think most of them provides very little values and actually hide what is going on behind one level of abstraction.

They also mixes procedural-looking code inside code that is by nature heavily object oriented, which I find weird.

Facades are a static-looking proxy for services, they are an actually class, almost empty that just proxy any static method call to an underlying service.

### Exceptions

They are useful in places where autocompletion or importing full namespace is not easy like in Tinker.

Debugging functions like `dd()` and `dump()` are also fine.

### Alternatives

There are two types of helpers that can be easily replaced.

Some just instantiate a specific class, others call the corresponding method on a specific service.  
So instead of calling the helper, just do what it does.

Some examples:
- `now()` can be replaces by `new Carbon` or `Carbon::now()` (or even the built-in `DateTime*` classes)
- `collect()` can be replaced by `new Collection()` or whatever collection class you are interested in
- `redirect()` can be replaced by calling the `to` method on the `\Illuminate\Routing\Redirector` service (alternatively the `to` method on the `Redirect` facade)
- `request()` can be replaced by injecting the Request object in your class/method
- ...


## Do not use facade aliases 

Before Laravel 11, there is a `app.aliases` configuration array, which map a short name with a class FQCN.
It is only ever used for facades but can be used for any classes.

This allow to use facades (or other classes) without even using their FQCN or adding the `use` statement, which, like helpers is only ever maybe useful in Tinker.

I prefer to empty completely this list so that no alias is registered. Note that the autoloader callback is still being registered, which is a little annoying when your are step-debugging with xDebug.

In Laravel 11+, this config entry has been removed. The registering of the aliases is part of the app bootstrap (TODO verify that).
To prevent that happening, you can manually make so that the loader is registered, with an empty alias list.

Add this code to the bootstrap `app.php` file :
```php
$loader = new AliasLoader(); // with empty aliases
$loader->setRegistered(true); // thee autoloader will not be added
AliasLoader::setInstance($loader);
```


## TODO

- no higherorderproxy
- custom query builders instead of scope (trait or base builder instead, base builder allow to change the type of the underlying method)
- typed config and request/input/uqery methods
- misc
	- type everything
	- no hasfactory on models
	- prevent lazy loading
	- prevent polymorphic relationships without morph map
	- use fillable
	- preventSilentlyDiscardingAttributes preventAccessingMissingAttributes shouldBeStrict
	- don't use string name for service, use interfaces FQCN
	- use carbonimmutable instead of carbon
	- dont use collections as array