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
8. [Do not use macros](#do-not-use-macros)
9. [Always use strict models/Eloquent](#always-use-strict-modelseloquent)
10. [Always enforce the morph map](#always-enforce-the-morph-map)
11. [Cleanup your project](#cleanup-your-project)
12. [Use multiple methods instead of "method overloading"](#use-multiple-methods-instead-of-method-overloading)
13. [Use specific typed methods instead generic ones that return mixed](#use-specific-typed-methods-instead-generic-ones-that-return-mixed)

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


## Do not use classes as array

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


## Do not use macros

Several built-in Laravel classes are "macroable", they have the Macroable trait, which allow to dynamically add methods to them at runtime.  
See the example with collections: https://laravel.com/docs/11.x/collections#extending-collections

These can be made statically analysable if you redeclare the class in a file, with the method on it.
PHPStorm and PHPStan are nice enough to merge both definitions, even though PHPStorm may complain about duplicate definition of the class.

For instance you can add file `extra_definitions.php` at the root of the project with a content like so to declare the `toUpper()` method of the collection in a PHPDoc.
```php
namespace Illuminate\Support {
	/**
	 * @method self toUpper()
	 */
	class Collection
	{
		//
	}
}
```

Note a limitation of that technique is that it's not possible to declare an instance method that return `static`, since  `@method static toUpper()` declares the method as being static.


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


## Always enforce the morph map

When you use polymorphic relationships, one of the field involved contains the "type" of the model.
By model the value is the models FQCN, so for instance `App\Model\User`, which contain 10 useless characters (`App\Model\`) but that still must be indexed.

As a simple optimisation Laravel allow you to alias you model FQCN's to shorter string via the MorphMap : https://laravel.com/docs/11.x/eloquent-relationships#custom-polymorphic-types

Typically it is defined in a service provider and I suggest that you always use the `Relation::enforceMorphMap()` method even when you don't yet need polymorphic relations.  
That way you will get an exception when you will define your first one if you forget to setup an alias.


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

This goes also for the dependencies, the starter project comes with Pint and Sail but it is up to you to decide if you want to use them or not. It is particularly important to remove unused dependencies.


## Use multiple methods instead of "method overloading"

Method overloading is ability to declare the same method multiple times, but with different signatures (and different body).

You can not do that in PHP, yet Laravel has many methods that have multiple distinct signatures.  
This can only be achieved by declaring many of the arguments optional and have many union types even if for most signatures the arguments are not optional and can only accept a single type.

Take the config `getString(string $key, null|string $default = null)` method we added in the next section. It actually has 2 different intended signatures:
- `getString(string $key): string`
- `getString(string $key, string $default): string`

The `$default` argument isn't actually nullable. We set it as nullable only because to have to give it a default value to make it optional.  
Since there is no natural pertinent default value for this use case, we set it to null.  
Here we could give each of these signatures its own name: `getString()` and `getStringOrDefault()` for instance.

In the case of the query builder, we can define multiple version of the `where()` method to be much clearer on the expected arguments:
- `where(string $field, string $operator, string|int|float|bool $value): static`
- `whereGroup(Closure $group): static`
- `whereExpression(Expression $expression): static`


## Use specific typed methods instead generic ones that return mixed

This is particularly important if you use static analysers at max level where you can not do anything with values that are of type `mixed`.  
This is because mixed can be anything, it can be null or an object that is not castable for instance, so you can not even cast them to anything.

This is problematic for values that are coming from the configuration, or a request body or query string for instance because the methods to extract information from these naturally returns mixed.

What we can do is not use these methods and instead introduce new, more specific methods that have proper return types.

### Configuration

The whole configuration is held in a singleton that is the `Config\Repository` class which only has a generic `get(string $key, mixed $default): mixed` method to extract values.

Laravel 11 added the `string(): string`, `integer(): int`, `float(): float`, `boolean(): bool`, and `array(): array` methods, to get value of the corresponding type which throws and exception if the value is not correct (including null).  

The idea is to replace the built-in repo by a new one that has more methods, one that return each scalar type + arrays, properly typed and one more version for each that return the type or null.

```php
final class Config extends Illuminate\Support\Config
{
	public function getString(string $key, null|string $default = null): string
	{
		$value = $this->get($key, $default);

		if (! is_string($value)) {
			throw new InvalidArgumentException("Configuration value for key [$key] must be a string, '" . get_debug_type($value) . "' given.");
		}

        return $value;
	}

	public function getStringOrNull(string $key): null|string
	{
		$value = $this->get($key);

		if ($value === null) { // this if is the only difference with the previous method
			return null;
		}

		if (! is_string($value)) {
			throw new InvalidArgumentException("Configuration value for key [$key] must be a string, '" . get_debug_type($value) . "' given.");
		}

        return $value;
	}

	// then same thing with all other scalars types

	public function getInt(string $key, null|int $default = null): int {}
	public function getIntOrNull(string $key): null|int {}

	public function getFloat(string $key, null|float $default = null): float {}
	public function getFloatOrNull(string $key): null|float {}

	public function getBool(string $key, null|bool $default = null): bool {}
	public function getBoolOrNull(string $key): null|bool {}

	public function getArray(string $key, null|array $default = null): array {}
	public function getArrayOrNull(string $key): null|array {}
}
```

Then you need to register this class instead of the base one in the Container. You can do that in your main service provider.

```php
public function register(): void
{
	$customConfigRepo = new Config($this->app->make(ConfigContract::class)->all());

	$this->app->singleton(ConfigContract::class, $customConfigRepo);
	$this->app->singleton('config', $customConfigRepo);

	//---
}
```

### Request body and query string

The Illuminate Request object has the `Illuminate/Http/Concerns/InteractsWithInput` trait that contains the method used to extract values from the body or query string (among others).

Typically you use mostly the `post(): array|string|null` (to get values from the body), `query(): array|string|null` (to get values from the query string) or `input(): mixed` (to get values from either) methods. 
For a long time you could also retrieve a boolean or a date from the body with the `boolean` and `` methods.

Laravel 9 added other type specific methods: `integer`, `float`, `string` and `enum`.

If you want other version of these methods, like with other names or that may return null or that distinguish between retrieving a value from the body or the query string, you can have them own your own Request object that inherit from Laravel's one.

Then you can edit the `public/index.php` file in your project to use the correct Request object.  
Of course when you inject the request to methods or constructor, you must use your request class instead of the base one.

In some situations, you may need to actually register your custom request object with the same aliases that the base object has.   
In you app's main service provider you can write this: 
```php
public function register(): void
{
	$request = $this->app->make(MyCustomRequest::class);

	$this->app->singleton('request', $request);
	$this->app->singleton(\Illuminate\Http\Request::class, $request);
	$this->app->singleton(\Symfony\Component\HttpFoundation\Request::class, $request);
}
```

















## TODO

- custom query builders instead of scope (trait or base builder instead, base builder allow to change the type of the underlying method)

