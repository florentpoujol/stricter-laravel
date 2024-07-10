# Stricter Laravel

Laravel is a great framework but a lot lenient in many ways, which some like me doesn't find a good point.

This document aims to be a collection of tips/advices on how to use or not use some features of Laravel, why, and the alternatives that I prefer.

The goal is to still use Laravel of course, but in a way that **I find** more approachable, easier to understand, with less abstraction/indirection and more static analysis friendly.

Remember that ease of maintainability is more important than raw speed of development.

1. [General](#general)
2. [Use dependency injection whenever possible](#use-dependency-injection-whenever-possible)
3. [Do not use helper functions or facades](#do-not-use-helper-functions-or-facades)
4. [Do not use Facade aliases](#do-not-use-facade-aliases)
5. [Do not use higher order proxys](#do-not-use-higher-order-proxys)
6. [Do not use the HasFactory trait on models](#do-not-use-the-hasfactory-trait-on-models)
7. [Do not use classes as array](#do-not-use-classes-as-array)
8. [Always use strict models/Eloquent](#always-use-strict-modelseloquent)
9. [Cleanup your project](#cleanup-your-project)

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

This allow to use facades (or other classes) without even using their FQCN or adding the `use` statement, which, like helpers may only ever be useful in Tinker.

I prefer to empty completely this list so that no alias is registered. Note that the autoloader callback is still being registered, which is a little annoying when your are step-debugging with xDebug.

In Laravel 11+, this config entry has been removed. The registering of the aliases is part of the app bootstrap (TODO verify that).
To prevent that happening, you can manually make so that the loader is registered, with an empty alias list.

Add this code to the bootstrap `app.php` file :
```php
$loader = new AliasLoader(); // with empty aliases
$loader->setRegistered(true); // thee autoloader will not be added
AliasLoader::setInstance($loader);
```


## Do not use [higher order proxys](https://laravel.com/docs/11.x/collections#higher-order-messages)

Because there are not statically analysable, and there is not way to make them so.

Instead, use the method normally and pass a closure to them. With short closure, the code is barely longer.
```php
$users = User::where(...)->get();

$users->each(fn (User $user) => $user->markAsVip());

$sum = $users->sum(fn (User $user) => $user->votes);
```


## Do not use the HasFactory trait on models

The `HasFactory` trait, used on models, provide a `factory()` static method that returns the factory of the model.
Since the factory itself is always named after the model, it is as much clear to work with the factory directly, which prevent an indirection and a trait on all models.

```php
// instead of 
$factory = User::factory();

// do
$factory = UserFactory::new();
```


## Don't use classes as array

PHP as a few interfaces that allow collection classes to behave like arrays: mostly [ArrayAccess](https://www.php.net/manual/en/class.arrayaccess.php), [Countable](https://www.php.net/manual/en/class.countable.php) and the iterators.

Mistaking objects for arrays by using the square brackets on them or passing them to the `count()` function I think leads to code that it confusing. 

Laravel collections are such classes.
The idea is to just not use these features on what you know is an object, when they provide methods to do the same things.

Ie:
```php
$users->count(); // instead of count($users)

// instead of $users['the key']:
$users->get('the key'); // for a base collection
$users->find('the PK'); // for an eloquent collection
```

Typically these classes also implements one of the interface that makes them iterable.  
I have no problem however to use them with `foreach()` as I prefer this "native" approach to iteration over passing a closure to the `each()` method that exists on the collections.


## Always use strict models/Eloquent

Models have dynamic attributes, that may or may not exists.  
It is also easy for the unfamiliar to produce N+1 queries.

To alleviate some of the problems caused by that [Laravel allow to configure three behaviours](https://laravel.com/docs/11.x/eloquent#configuring-eloquent-strictness).

Eloquent can prevent when lazy loading (N+1 queries) is happening and then throw an exception.  

Then as explained in the doc, Laravel can also throw an exception if you try to set an unfillable attribute.
If that happens it may indicate that you are fetching to much fields from the database, or are not properly restricting the data you consider when validating a form for instance.
For that to properly work, you should always mention all the fillable attributes in the model's `$fillable` property to clearly mark which are.

Finally, and that's not mentioned in the doc, Laravel can also prevent accessing an attribute that do not exists (with the model's `preventAccessingMissingAttributes()` method).  
If that happens, it may be because you are not fetching enough fields from the database for instance.

The Model's `shouldBeStrict()` method can be used to set all three behaviours at the same time.  
So in your app main service provider, you should have the line
```php
Model::shouldBeStrict(! $this->app->isProduction());
``` 

Typically these behaviour are enabled in the local and testing environment, where they are harmless and where its easy to fix but not in Production since by default they cause an uncaught exception.

Alternatively you can still enable it in prod but with an easy way to deactivate it, or catch the exception to only produces a log for instance.  
Each of these behaviour have there own exceptions (so they are easy to handle in the main exception handler), and you can also provide for each of them a specific callback, with the methods like `Model::handleLazyLoadingViolationUsing($callback)`.


## Cleanup your project

The [Laravel starter project](https://github.com/laravel/laravel) repo is what will become your project. Once you install Laravel all these files become yours. 

**They are your responsibility, Laravel will not come to maintain them.**  

This is the same thing with files that are generated with the Artisan make command: once they are generated, they are yours, so check them.
Note that you can modify stubs, see https://laravel.com/docs/11.x/artisan#stub-customization and https://laravel-news.com/customizing-stubs-in-laravel

This is your job to check that all these files are needed, that all their content is needed, and that it match you code quality standards.

So you have to make sure that you understand every configuration options, what every middleware do, etc...

Be advised though, that removing stuff that you do not know or understand at all may break the app.  
Ideally, you should do that after having a working app with some content and test and before the first deployment to production.  
Then remove things one by one and carefully check every time that every things still works.

This advice was important before Laravel 11 because the starter project had **a lot** of stuff, which is thankfully not the case any more since L11 only has minimal configuration and almost no more code.


## TODO

- custom query builders instead of scope (trait or base builder instead, base builder allow to change the type of the underlying method)
- typed config and request/input/uqery methods
- use multiple methods instead of method overloading
- misc
	- prevent polymorphic relationships without morph map
	- use carbonimmutable instead of carbon
	- alway use array with validation