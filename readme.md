# Stricter Laravel

Laravel is a great framework but lenient in many ways, which some like me doesn't find a good point.

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
14. [Use custom query builders or repositories instead of scopes](#use-custom-query-builders-or-repositories-instead-of-scopes)

## General

Here is some general advices/guidelines:

Do not use features that aren't statically analysable. By statically analysable, I mean by a type checker like PHPStan.

Aims for having the least abstraction level possible.  
Use services directly instead of helpers, and inject these services instead or relying on the service locator pattern (via a helper or facade).

Aims for having one way to do each thing and do not allow them to be doable.

With IDEs and AI, writing more code isn't slower and often make the intent of the code clearer (and thus probably more maintainable).  
So aims to have clear intent, and clear flow of information.


## Use dependency injection whenever possible

At least where it's easy (like in controllers and Artisan commands), there is no reason to not use dependency injection instead of using helpers or Facades.

This helps see what are the actual dependencies of your code are (what other services your code needs to run).

If your type the dependencies against interfaces, this also may help testing by injecting a mock or fake object instead of the actual implementation, which can be a good idea for external dependencies like databases.

Remember that in Laravel controllers, you can use dependency injection both in the constructor and in each method that are target of a route.

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
        // constructor injection (may help your controller be more concise if the service is used in a lot of methods)
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

Whenever you are in a service-locator situation where you must resolve a service directly from the application instance (with `Application->make()` method for instance), prefer using an interface or class FQCN (Fully Qualified Class Name, the full name of the class with the whole namespace) instead of a string alias.
The reason is the same, it's to better see/understand what is going on, to reduce the "magic" that is the string alias.

Ie: 
```php
$container = Container::getInstance();

// instead of
$container->make('url');

// do
$container->make(\Illuminate\Routing\UrlGenerator::class);
// or with the interface
$container->make(\Illuminate\Contracts\Routing\UrlGenerator::class); // typically return an instance of \Illuminate\Routing\UrlGenerator::class (or any other custom service that has been bound to the interface)
```


## Do not use helper functions or facades

Laravel define several helper functions, accessible on the global namespace, that are shortcuts for instantiating specific classes (like `now()` that instantiate a `Carbon` object) or for using specific services (like `action()` that calls the action method on the `UrlGenerator` service).

I think most of them provides very little values and actually hide what is going on behind one level of abstraction (or more).

They also mixes procedural-looking code inside code that is by nature heavily object-oriented, which I find weird.

Facades are a static-looking proxy for services, they are an actual class, almost empty that just proxy any static method calls to an underlying service.

### Exceptions

They are useful in places where autocompletion or importing full namespace is not easy like in Tinker.

Debugging functions like `dd()` and `dump()` are also fine since they are temporary and not "production code".

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

Before Laravel 11, there was a `app.aliases` configuration array, which map a short name with a class FQCN.  
It is only ever used for facades but can be used for any classes.

This allows to use facades (or other classes) without even using their FQCN or adding the `use` statement, which, like helpers may only ever be useful in Tinker.

I prefer to empty completely this list so that no alias is registered. Note that the autoloader callback is still being registered, which is a little annoying when you are step-debugging with xDebug.

In Laravel 11+, this config entry has been removed. The registering of the aliases is part of the app bootstrap (TODO verify that).
To prevent that happening, you can manually make so that the loader is registered, with an empty alias list and in a way that the autoloader will not be created.

Add this code to the bootstrap `app.php` file :
```php
$loader = new AliasLoader(); // with empty aliases
$loader->setRegistered(true); // the autoloader will not be added
AliasLoader::setInstance($loader);
```


## Do not use [higher order proxys](https://laravel.com/docs/11.x/collections#higher-order-messages)

Because there are not statically analysable, and there is no way to make them so.

Instead, use the method normally and pass a closure to them. With short closure, the code is barely longer.
```php
$users = User::where(...)->get();

// instead of 
$users->each->markAsVip();
// do
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

Mistaking objects for arrays by using the square brackets on them or passing them to the `count()` function I think leads to code that is confusing. 

Laravel collections are such classes.  
The idea is to just not use these features on what you know is an object, when they provide methods to do the same things.

Ie:
```php
$users->count(); // instead of count($users)

// instead of $users['the key']:
$users->get('the key'); // for a base collection
$users->find('the PK'); // for an eloquent collection
```

Typically, these classes also implements one of the interface that makes them iterable.    
I have no problem however to use them with `foreach()` as I prefer this "native" approach to iteration over passing a closure to the `each()` method that exists on the collections.


## Do not use macros

Several built-in Laravel classes are "macroable", they have the `Macroable` trait, which allow to dynamically add methods to them at runtime.  
See the example with collections: https://laravel.com/docs/11.x/collections#extending-collections

These can be made statically analysable if you redeclare the class in a file, with the method on it.  
PHPStorm and PHPStan are nice enough to merge both definitions, even though PHPStorm may complain about duplicate definition of the class.

For instance, you can add file `extra_definitions.php` at the root of the project with a content like so to declare the `toUpper()` method of the collection in a PHPDoc.
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

Models have dynamic attributes ("properties"), that may or may not exist.  
It is also easy for the unfamiliar to produce N+1 queries.

To alleviate some of the problems caused by that [Laravel allow to configure three behaviours](https://laravel.com/docs/11.x/eloquent#configuring-eloquent-strictness).

Eloquent can prevent when lazy loading (N+1 queries) is happening and then throw an exception.  

Then as explained in the doc, Laravel can also throw an exception if you try to set an unfillable attribute.
If that happens it may indicate that you are fetching to much fields from the database, or are not properly restricting the data you consider when validating a form for instance.
For that to properly work, you should always mention all the fillable attributes in the model's `$fillable` property to clearly mark which attributes are indeed fillable.

Finally, and that's not mentioned in the doc, Laravel can also prevent accessing an attribute that do not exists (with the model's `preventAccessingMissingAttributes()` method).  
If that happens, it may be because you are not fetching enough fields from the database for instance.

The Model's `shouldBeStrict()` method can be used to set all three behaviours at the same time.  
So in your app main service provider, you should have the line
```php
Model::shouldBeStrict(! $this->app->isProduction());
``` 

Typically, these behaviour are enabled in the local and testing environment, where they are harmless and where its easy to fix but not in Production since by default they cause an uncaught exception.

Alternatively you can still enable it in prod but with an easy way to deactivate it, or catch the exception to only produces a log for instance.  
Each of these behaviour have their own exceptions (so they are easy to handle in the main exception handler), and you can also provide for each of them a specific callback, with the methods like `Model::handleLazyLoadingViolationUsing($callback)`.


## Always enforce the morph map

When you use polymorphic relationships, one of the field involved contains the "type" of the model.
By model the value is the models FQCN, so for instance `App\Model\User`, which contain 10 useless characters (`App\Model\`) but that still must be indexed.

As a simple optimisation Laravel allow you to alias you model FQCN's to shorter string via the MorphMap : https://laravel.com/docs/11.x/eloquent-relationships#custom-polymorphic-types

Typically, it is defined in a service provider and I suggest that you always use the `Relation::enforceMorphMap()` method even when you don't yet need polymorphic relations.  
That way you will get an exception when you will define your first one if you forget to configure an alias.


## Cleanup your project

The [Laravel starter project](https://github.com/laravel/laravel) repo is what will become your project. Once you install Laravel all these files become yours. 

**They are your responsibility, Laravel will not come to maintain them. This is as if you wrote them yourself.**  

This is the same thing with files that are generated with the Artisan make command: once they are generated, they are yours, so check them.
Note that you can modify stubs, see https://laravel.com/docs/11.x/artisan#stub-customization and https://laravel-news.com/customizing-stubs-in-laravel

This is your job to check that all these files:
- are needed
- that all their content is needed
- and that it match you code quality standards.

So you have to make sure that you understand every configuration options, what every middleware do, etc...

Be advised though, that removing stuff that you do not know or understand at all may break the app.  
Ideally, you should do that after having a working app with some content and test and before the first deployment to production.  
Then remove things one by one and carefully check every time that everything still works.

This advice was important before Laravel 11 because the starter project had **a lot** of stuff, which is thankfully not the case anymore since the L11 starter project only has minimal configuration and almost no more code.

This goes also for the dependencies, the starter project comes with Pint and Sail for instance, but it is up to you to decide if you want to use them or not. It is particularly important to remove unused dependencies.


## Use multiple methods instead of "method overloading"

Method overloading is the ability to declare the same method multiple times, but with different signatures (and different body).

You can not do that in PHP, yet Laravel has many methods that have multiple distinct signatures.  
This can only be achieved by declaring many of the arguments optional and have many union types even if for most signatures the arguments are not optional and can only accept a single type.

Take the config `getString(string $key, null|string $default = null)` method that we will add in the next section. It actually has 2 different intended signatures:
- `getString(string $key): string`
- `getString(string $key, string $default): string`

The `$default` argument isn't actually nullable. We set it as nullable only because to have to give it a default value to make it optional.  
Since there is no natural pertinent default value for this use case, we set it to null.  
Here we could give each of these signatures its own name: `getString(string $key)` and `getStringOrDefault(string $key, string $default)` for instance.

In the case of the query builder, we can define multiple version of the `where()` method to be much clearer on the expected arguments:
- `where(string $field, string $operator, string|int|float|bool $value): static`
- `whereGroup(Closure $group): static`
- `whereExpression(Expression $expression): static`


## Use specific typed methods instead generic ones that return mixed

This is particularly important if you use static analysers at max level where you can not do anything with values that are of type `mixed`.  
This is because mixed can be anything, it can be null or an object that is not castable for instance, so you can not even cast these values to anything.

This is problematic for values that are coming from the configuration, or a request body or query string for instance because the methods to extract information from these naturally returns mixed.

What we can do is not use these methods and instead introduce new, more specific methods that have proper return types.

### Configuration

The whole configuration is held in a singleton that is the `Config\Repository` class which only has a generic `get(string $key, mixed $default): mixed` method to extract values.

Laravel 11 added the `string(): string`, `integer(): int`, `float(): float`, `boolean(): bool`, and `array(): array` methods, to get value of the corresponding type which throws and exception if the value is not correct (including null).  

The idea is to replace the built-in repo by a new one that has more methods, one that return each scalar type + arrays, properly typed and one more version for each that return the type or null.

```php
final class Config extends Illuminate\Support\Config
{
    public function getStringOrFail(string $key): string
    {
        $value = $this->get($key);
        
        if (! is_string($value)) {
            $message = "Missing configuration key [$key].";
            
            $type = get_debug_type($value);
            if ($type !== 'null') {
                $message = "Configuration value for key [$key] must be a string, '$type' given.";
            }
            
            throw new InvalidArgumentException($message);
        }
    
        return $value;
    }
    
    public function getStringOrDefault(string $key, string $default): string
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
    
    public function getIntOrFail(string $key): int {}
    public function getIntOrDefault(string $key, int $default): int {}
    public function getIntOrNull(string $key): null|int {}
    
    // ...
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

The Illuminate `Request` object has the `Illuminate\Http\Concerns\InteractsWithInput` trait that contains the methods used to extract values from the body or query string (among others).

Typically, you use mostly the `post(): array|string|null` (to get values from the body), `query(): array|string|null` (to get values from the query string) or `input(): mixed` (to get values from either) methods. 
For a long time you could also retrieve a boolean or a date from the body with the `boolean` method.

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


## Use custom query builders or repositories instead of scopes

One of the most defining feature of Laravel is the ability to work with the database seamlessly through the models.  
This is actually a characteristics of the [Active Record pattern](https://martinfowler.com/eaaCatalog/activeRecord.html), also used by Ruby on Rails for instance.

You start a query by calling what looks like a static method on the models when in fact it is an instance method of the query builder.  
And you can call on the query builder instances method that are actually defined on the model with a different name (the local scopes).

Of course these call patterns are not wanted because they are not statically analysable and I think they are more confusing than anything.

The first pattern can be very easily avoided by calling the actual static `query(): Illuminate\Database\Eloquent\Builder` method on the model, which return the Eloquent query builder instance.
```php
// instead of
User::where(...)->get();

// do
User::query()->where(...)->get();
```

Another alternative, that is practical only for the methods that you use the most, maybe like `where()` and `find()` for instance, is to actually declare them as static methods on the model.

The second pattern is the use of scopes, global or local : https://laravel.com/docs/11.x/eloquent#query-scopes

The solutions for that (to not use any scope) is more involved and require to move your scope methods either in repositories, or in custom query builders classes.

### Repositories

First, repositories are not a pattern specific to DataMapper ORMs. You can absolutely do Active Record inside a repository (even though it may feel a little weird).  
**Do what you think is best for your app**.

Second, repositories allows for easy mocking/replacement during tests (which custom query builders like I will show in the next section do not allow), and thus integration tests without the database that are probably easier to setup and faster to run.

The idea here is to have specific classes (one per model) where you actually do SQL requests, which are the best place to have the method that do more work.  
Here is some example from the documentation, as a repository:
```php
use Illuminate\Database\Eloquent\Builder;

final class AncienUserRepository
{
    private function getBuilder(): Builder
    {
        $builder = User::query();
        
        // add your "global" scopes here or in dedicated methods
        $builder->where('created_at', '<', new DateTimeImmutable('- 2000 years'));
        
        // if a scope should be shared with multiple models, add it in a method in a trait, or use a dedicated class to encapsulate them
        
        return $builder;
    }
        
    // A "local" scope.
    // If the method is public, it can even be used from outside the repository
    public function whereIsPopular(Builder $builder): void
    {
        $builder->where('votes', '>', 100);
    }
    
    // a regular method, that should be called from a controller or service, that uses both scopes defined here
    public function getPopularUsers(): Collection
    {
        $builder = $this->getBuilder();
        
        // then use the local scope in dedicated methods of the repository which you query from your services or controllers
        $this->whereIsPopular($builder);
        
        return $builder->get();
    }
}
```

Note that with the simple code as shown here, there is no way to do a query through the repository without the "global scopes".

But all of that is easily statically analysable, encapsulate the "complexity" of building an SQL requests and is easily injectable as dependency in any services or controllers.

### Custom query builder

The other solution is also not mentioned in the documentation. But you can as easily have dedicated custom query builder classes, that inherit or compose the Eloquent query builder.

The query builder used by each model is defined in the static `$builder` property that you can override in each of your models.  
Before Laravel 11, it was hardcoded in the `newEloquentBuilder()` method, which you can also override.

Here is an example of query builder method, similar to the repository of above:
```php
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Query\Builder as BaseBuilder;

final class AncienUserBuilder extends Builder
{
    public function __construct(BaseBuilder $baseBuilder)
    {
        parent::__construct($baseBuilder);
        
        // add your "global" scopes here or in dedicated methods
        $this->where('created_at', '<', new DateTimeImmutable('- 2000 years'));
        
        // if a scope should be shared with multiple models, add it in a method in a trait, or use a dedicated class to encapsulate them
    }
	
    // A "local" scope.
    // If the method is public, it can even be used from outside the repository
    public function whereIsPopular(): self
    {
        $this->where('votes', '>', 100);
        
        return $this;
    }
    
    // a regular method, that should be called from a controller or service, that uses both scopes defined here
    /**
     * @template User
     * 
     * @return Collection<User>
     */
    public function getPopularUsers(): Collection
    {
        return $this
            ->whereIsPopular() 
            ->get();
    }
}

// in the body of the User model:
protected static string $builder = AncienUserBuilder::class

// and then use it from a controller or service:
$user = User::query()->getPopularUsers();
// or if you do not want to have "repository like" methods, just use any methods of the query builder directly, but in this case you would not benefit from the generic annotations of the getPopularUsers() method
$user = User::query()->whereIsPopular()->get();
```

Having your own query builder also allow to define any other methods that are not model-specific.  
In this case you either must define them in a trait that is added to all builders, or have all builder extends a base one, which is the one that extends the Eloquent builder.

For instance in the section [12. Use multiple methods instead of "method overloading"](#use-multiple-methods-instead-of-method-overloading), I gave some versions of the where method, that we can no write:

```php
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Query\Builder as BaseBuilder;

abstract class CommonBuilder extends Builder
{
    /**
     * @param string $column
     * @param string $operator
     * @param int|float|string|bool $value
     * @param 'and'|'or' $boolean
     */
    public function where($column, $operator = null, $value = null, $boolean = 'and'): static
    {
        // even if we can not change the signature of the $operator argument
        // we can check in the call that it is indeed the operator, thus having the same effect
        if (! in_array($operator, $this->query->operators, true)) {
            throw new UnexpectedValueException('Call the where() method only with this signature: where(string $column, string $operator, int|float|string|bool $value)');
        }
        
        parent::where($column, $operator, $value, $boolean);
        
        return $this;
    }

    public function whereGroup(Closure $group): static
    {
        parent::where($group);
        
        return $this;
    }
}
```

Ideally in this example we should have redefined the `where()` signature to what we want it to be, but in this case we can't since our builder extends from Eloquent's and the options to change a method argument signature is very limited.

So in this example we just throw an exception if we detect that the signature isn't correct because the operator argument is clearly not an operator, which still forces to use the method the way we want.

The alternative would be to have our builder decorate the Eloquent builder instead of extending it.
