# Request Factories

Test requests in Laravel without all the boilerplate.

_Note: This is a fork of worksome/request-factories so you can still run it on PHP 8.1. If you are running PHP 8.2 or higher, you should use that._

> 💡 Psst. Although our examples use Pest PHP, this works just as well in PHPUnit.

Take a look at the following test:

```php
it('can sign up a user with an international phone number', function () {
    $this->put('/users', [
        'phone' => '+375 154 767 1088',
        'email' => 'foo@bar.com', 🙄
        'name' => 'Luke Downing', 😛
        'company' => 'Worksome', 😒
        'bio' => 'Blah blah blah', 😫
        'profile_picture' => UploadedFile::fake()->image('luke.png', 200, 200), 😭
        'accepts_terms_and_conditions' => true, 🤬
    ]);
    
    expect(User::latest()->first()->phone)->toBe('+375 154 767 1088');
});
```

Oof. See, all we wanted to test was the phone number, but because our route's FormRequest has validation rules, we have to send all of these
additional fields at the same time. This approach has a few downsides:

1. *It muddies the test.* Tests are supposed to be terse and easy to read. This is anything but.
2. *It makes writing tests annoying.* You probably have more than one test for each route. Every test you write requires all of these fields over and over again.
3. *It requires knowledge of the FormRequest.* You'd need to understand what each field in this form does before being able to write a passing test. If you don't, you're likely going to be caught in a trial-and-error loop, or worse, a false positive, whilst creating the test.

We think this experience can be vastly improved. Take a look:

```php
it('can sign up a user with an international phone number', function () {
    SignupRequest::fake();

    $this->put('/users', ['phone' => '+375 154 767 1088']);

    expect(User::latest()->first()->phone)->toBe('+375 154 767 1088');
});
```

Soooooo much nicer. And all thanks to Request Factories. Let's dive in...

## Installation

You can install the package as a developer dependency via Composer:

```bash
composer require --dev dcaswel/request-factories 
```

## Usage

First, let's create a new `RequestFactory`. A `RequestFactory` usually complements a `FormRequest`
in your application ([request factories work with standard requests too!](#using-factories-without-form-requests)). You can create a `RequestFactory` using the `make:request-factory` Artisan command:

```bash
php artisan make:request-factory "App\Http\Requests\SignupRequest"
```

Note that we've passed the `SignupRequest` FQCN as an argument. This will create a new request factory
at `tests/RequestFactories/SignupRequestFactory.php`.

You can also pass your desired request factory name as an argument instead:

```bash
php artisan make:request-factory SignupRequestFactory
```

Whilst you're free to name your request factories as you please, we recommend two defaults for a seamless experience:

1. Place them in `tests/RequestFactories`. The Artisan command will do this for you.
2. Use a `Factory` suffix. So `SignupRequest` becomes `SignupRequestFactory`.

### Factory basics

Let's take a look at our newly created `SignupRequestFactory`. You'll see something like this:

```php
namespace Tests\RequestFactories;

use Worksome\RequestFactories;

class SignupRequestFactory extends RequestFactory
{
    public function definition(): array
    {
        return [
            // 'email' => $this->faker->email,
        ];
    }
}
```

If you've used Laravel's [model factories](https://laravel.com/docs/database-testing#defining-model-factories) before,
this will look pretty familiar. That's because the basic concept is the same: a model factory is designed to generate data
for eloquent models, a request factory is designed to generate data for form requests.

The `definition` method should return an array of valid data that can be used when submitting your form. Let's fill it out for
our example `SignupRequestFactory`:

```php
namespace Tests\RequestFactories;

use Worksome\RequestFactories;

class SignupRequestFactory extends RequestFactory
{
    public function definition(): array
    {
        return [
            'phone' => '01234567890',
            'email' => 'foo@bar.com',
            'name' => 'Luke Downing',
            'company' => 'Worksome',
            'bio' => $this->faker->words(300, true),
            'accepts_terms_and_conditions' => true,
        ];
    }
    
    public function files(): array
    {
        return [
            'profile_picture' => $this->file()->image('luke.png', 200, 200),
        ];
    }
}
```

Note that we have access to a `faker` property for easily generating fake content, such as a paragraph
for our bio, along with a `files` method we can declare to segregate files from other request data.

### Usage in tests

So how do we use this factory in our tests? There are a few options, depending on your preferred style.

#### Using `create` on the factory

This method is most similar to Laravel's model factories. The `create` method returns an array,
which you can then pass as data to `put`, `post` or any other request testing method.

```php
it('can sign up a user with an international phone number', function () {
    $data = SignupRequest::factory()->create(['phone' => '+44 1234 567890']);
    
    $this->put('/users', $data)->assertValid();
});
```

#### Using `fake` on the request factory 

Seeing as you only normally make a single request per test, we support registering your factory globally with `fake`. If you're using this approach, 
make sure that it's the *last method you call on the factory*, and that you call it before making a request
to the relevant endpoint.

```php
it('can sign up a user with an international phone number', function () {
    SignupRequestFactory::new()->fake();
    
    $this->put('/users')->assertValid();
});
```

#### Using `fake` on the form request

If you've used Laravel model factories, you'll likely be used to calling `::factory()` on eloquent models to get a 
new factory instance. Request factories have similar functionality available. You don't need to do anything to enable this;
we automatically register the `::fake()` and `::factory()` method on all FormRequests via macros!

You can use these methods in your tests instead of instantiating the request factory directly:

```php
it('can sign up a user with an international phone number', function () {
    // Using the factory method...
    SignupRequest::factory()->fake();
    
    // ...or using the fake method
    SignupRequest::fake();
    
    $this->put('/users')->assertValid();
});
```

#### Using `fakeRequest` in Pest PHP

If you're using Pest, we provide a higher order method that you can chain onto your tests:

```php
// You can provide the form request FQCN...
it('can sign up a user with an international phone number', function () {
    $this->put('/users')->assertValid();
})->fakeRequest(SignupRequest::class);

// Or the request factory FQCN...
it('can sign up a user with an international phone number', function () {
    $this->put('/users')->assertValid();
})->fakeRequest(SignupRequestFactory::class);

// Or even a closure that returns a request factory...
it('can sign up a user with an international phone number', function () {
    $this->put('/users')->assertValid();
})->fakeRequest(fn () => SignupRequest::factory());
```

You can even chain factory methods onto the end of the `fakeRequest` method:

```php
it('can sign up a user with an international phone number', function () {
    $this->put('/users')->assertValid();
})
    ->fakeRequest(SignupRequest::class)
    ->state(['name' => 'Jane Bloggs']);
```

#### Overriding request factory data

It's important to note the order of importance request factories take when injecting data into your request.

1. Any data passed to `get`, `post`, `put`, `patch`, `delete` or similar methods will always take precedence.
2. Data defined using `state`, or methods called on a factory that alter state will be next in line.
3. Data defined in the factory `definition` and `files` methods come last, only filling out missing properties from the request.

Let's take a look at an example to illustrate this order of importance:

```php
it('can sign up a user with an international phone number', function () {
    SignupRequest::factory()->state(['name' => 'Oliver Nybroe', 'email' => 'oliver@worksome.com'])->fake();
    
    $this->put('/users', ['email' => 'luke@worksome.com'])->assertValid();
});
```

The default email defined in `SignupRequestFactory` is `foo@bar.com`. The default name is `Luke Downing`.
Because we override the `name` property using the `state` method before calling `fake`, the name used in
the form request will actually be `Oliver Nybroe`, not `Luke Downing`. 

However, because we pass `luke@worksome.com` as data to the `put` method, that will take priority over
*all other defined data*, both `foo@bar.com` and `oliver@worksome.com`.

### The power of factories

Factories are really cool, because they allow us to create a domain-specific-language for our feature tests. Because factories
are classes, we can add declarative methods that serve as state transformers.

```php
// In our factory...
class SignupRequestFactory extends RequestFactory
{
    // After the definition...
    public function withOversizedProfilePicture(): static
    {
        return $this->state(['profile_picture' => $this->file()->image('profile.png', 2001, 2001)])
    }
}

// In our test...
it('does not allow profile pictures larger than 2000 pixels', function () {
    SignupRequest::factory()->withOversizedProfilePicture()->fake();
    
    $this->put('/users')->assertInvalid(['profile_picture' => 'size']);
});
```

You can also use dot-notation in the `state` method to alter deeply nested keys in your request data.

```php
it('requires a postcode with the first line of an address', function () {
    SignupRequest::factory()->state(['address.line_one' => '1 Test Street'])->fake();
    
    $this->put('/users')->assertInvalid(['address.postcode' => 'required']);
});
```

The `state` method is your friend for any data you want to add or change on your factory. What about if you'd like to omit a property
from the request? Try the `without` method!

```php
it('requires an email address', function () {
    SignupRequest::factory()->without('email')->fake();
    
    $this->put('/users')->assertInvalid(['email' => 'required']);
});
```

> 💡 You can use dot syntax in the `without` method to unset deeply nested keys

You can also pass an array to `without` to unset multiple properties at once.

Sometimes, you'll have a property that you want to be based on the value of other properties.
In that case, you can provide a closure as the property value, which receives an array of all other parameters:

```php
class SignupRequestFactory extends RequestFactory
{
    public function definition(): array
    {
        return [
            'name' => 'Luke Downing',
            'company' => 'Worksome',
            'email' => fn ($properties) => Str::of($properties['name'])
                ->replace(' ', '.')
                ->append("@{$properties['company']}.com")
                ->lower()
                ->__toString(), // luke.downing@worksome.com
        ];
    }
}
```

Occasionally, you'll notice that multiple requests across your application share a similar subset of fields. For example,
a signup form and a payment form might both contain an address array. Rather than duplicating these fields in your factory, you can 
nest factories inside factories:

```php
class SignupRequestFactory extends RequestFactory
{
    public function definition(): array
    {
        return [
            'name' => 'Luke Downing',
            'company' => 'Worksome',
            'address' => AddressRequestFactory::new(),
        ];
    }
}
```

Now, when the `SignupRequestFactory` is created, it will resolve the `AddressRequestFactory` for you
and fill the `address` property with all fields contained in the `AddressRequestFactory` definition.
Pretty cool hey?

Request factories work hand in hand with model factories too. Imagine that you want to pass a `User` ID
to your form request, but you need to create the user in the database in order to do so. It's as simple
as instantiating the `UserFactory` in your request factory definition:

```php
class StoreMovieController extends RequestFactory
{
    public function definition(): array
    {
        return [
            'name' => 'My Cool Movie'
            'owner_id' => User::factory(),
        ];
    }
}
```

Because the `UserFactory` isn't created until compile time, we avoid any unexpected models being persisted to your test database
when you manually override the `owner_id` field.

### Using factories without form requests

Not every controller in your app requires a backing form request. Thankfully, we also support faking a generic request:

```php
it('lets a guest sign up to the newsletter', function () {
    NewsletterSignupFactory::new()->fake();
    
    post('/newsletter', ['email' => 'foo@bar.com'])->assertRedirect('/thanks');
});
```

## Solving common issues

### I'm getting a `CouldNotLocateRequestFactoryException`

When using the `::fake()` or `::factory()` methods on a `FormRequest`, we attempt to auto-locate the relevant
request factory for you. If your directory structure doesn't match for whatever reason, this exception
will be thrown.

It can easily be resolved by adding a `public static $factory` property to your form request:

```php
class SignupRequest extends FormRequest
{
    public static $factory = SignupRequestFactory::class; 
}
```

### I call multiple routes in a single test and want to fake both

No sweat. Just place a call to `fake` on the relevant request factory before making each request:

```php
it('allows a user to sign up and update their profile', function () {
    SignupRequest::fake();
    post('/signup');
    
    ProfileRequest::fake();
    post('/profile')->assertValid();
});
```

### I don't want to use the default location for storing request factories

Not a problem. We provide a config file you may publish where you can edit the
path and namespace of request factories. First, publish the config file:

```bash
php artisan vendor:publish --tag=request-factories
```

Now, in the newly created `config/request-factories.php`, alter the `path` and `namespace`
keys to suit your requirements:

```php
return [
    'path' => base_path('request_factories'),
    'namespace' => 'App\\RequestFactories',
];
```

## Testing

We pride ourselves on a thorough test suite and strict static analysis. You can run all of our checks via a composer script:

```bash
composer test
```

To make it incredibly easy to contribute, we also provide a docker-compose file that will spin up a container
with all the necessary dependencies installed. Assuming you have docker installed, just run:

```bash
docker-compose run --rm composer install # Only needed the first time
docker-compose run --rm composer test # Run tests and static analysis 
```

## Changelog

Please see [CHANGELOG](CHANGELOG.md) for more information on what has changed recently.

## Credits

- [Luke Downing](https://github.com/lukeraymonddowning)
- [All Contributors](../../contributors)

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
