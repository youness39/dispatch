Dispatch PHP 5.3 Utility Library
================================
At the very least, `dispatch()` is a front controller for your web app. It doesn't give you the full MVC setup, but it lets you define url routes and segregate your app logic from your views.

## Requirements
* PHP 5.3
* `mcrypt` extension if you want to use encrypted cookies and wish to use `encrypt()` and `decrypt()` functions
* `apc` extension if you want to use `cache()` and `cache_invalidate()`

## Installation
Dispatch can be installed by using `composer`. In your `composer.json` file, do the following:

```javascript
{
  "require": {
    "php": ">= 5.3.0",
    ...
    "dispatch/dispatch": "dev-master"
  }
}
```

After adding the appropriate `require` entries, do a `composer install` or `composer update` to install the package.

If you don't use `composer`, you can download and include [src/dispatch.php](https://github.com/noodlehaus/dispatch/raw/master/src/dispatch.php) directly in your application.

Note that Dispatch functions are all loaded into the global namespace.

## Configuration Variables

Certain properties and behaviours of Dispatch can be configured via the following `config()` entries:

* `config('source', 'inifile.ini')` makes the contents of `inifile.ini` accessible via `config()` calls
* `config('routing.base', 'string/to/strip')` lets you specify a string to strip from the URI before routing
* `config('views.root')` is used by `render()` and `partial()`, defaults to `./views`
* `config('views.layout')` is used by `render()`, defaults to `layout`
* `config('cookies.secret')` encryption salt to be used by `encrypt()`, `decrypt()`, `set_cookie()` and `get_cookie()`
* `config('cookies.flash')` cookie name to be used by `flash()` for setting messages

## Quick and Basic
A typical PHP app using dispatch() will look like this.

```php
<?php
// include the library
include 'dispatch.php';

// define your routes
get('/greet', function () {
	// render a view
	render('greet-form');
});

// post handler
post('/greet', function () {
	$name = from($_POST, 'name');
	// render a view while passing some locals
	render('greet-show', array('name' => $name));
});

// put handler
put('/users', function () {
  // ...
});

// delete handler
delete('/users/:id', function ($id) {
  // ...
});

// serve your site
dispatch();
?>
```

## URI Rewriting and Stripping
Setting `routing.base` to a string will strip that string from the URI before it is routed. Two use cases for this are when you don't have access to URI rewriting on your server, and if your dispatch application resides in a subdirectory.

```php
<?php
// example 1: want to strip the index.php part from the URI
config('routing.base', 'index.php');

get('/users', function () {
  echo "listing users...";
});

// requested URI = /index.php/users
// response = "listing users..."

// example 2: our app lives in /mysite
config('routing.base', 'mysite');

get('/users', function () {
  echo "listing users...";
});

// requested URI = /mysite/users
// response = "listing users..."
?>
```

## RESTful Resources
If you have a class that supports all or some of the default REST actions, you can easily publish them using `restify()`. By default, `restify()` will create all REST routes for your class. You can selectively publish actions by passing them to the function. To make a class support `restify()`, you need to implement some or all of the following methods:

* `onIndex` - for the resource list
* `onNew` - for the resource creation form
* `onCreate` - for the creation action
* `onShow($id)` - for viewing a resource
* `onEdit($id)` - for the resource edit form
* `onUpdate($id)` - for the resource edit action
* `onDelete($id)` - for the resource delete action

Note that the routes published by `restify()` uses the symbol `:id` to identify the resource.

```php
// resource to publish
class Users {
  public function onIndex() {}
  public function onNew() {}
  public function onCreate() {}
  public function onShow($id) {}
  public function onEdit($id) {}
  public function onUpdate($id) {}
  public function onDelete($id) {}
}

// publish the instance, with all endpoints, under /users
restify('/users', new Users());

// resource with just some of the REST endpoints
class Pages {
  public function onIndex() {
    echo "Pages::onIndex\n";
  }
  public function onShow($id) {
    echo "Pages::onShow {$id}\n";
  }
}

// publish object under /pages, but with just the available actions
restify('/pages', new Pages(), array('index', 'show'));
?>
```

## Route Symbol Filters
This is taken from ExpressJS. Route filters let you map functions against symbols in your routes. These functions then get executed when those symbols are matched.

```php
<?php
// preload blog entry whenever a matching route has :blog_id in it
filter('blog_id', function ($blog_id) {
	$blog = Blog::findOne($blog_id);
	// stash() lets you store stuff for later use (NOT a cache)
	stash('blog', $blog);
});

// here, we have :blog_id in the route, so our preloader gets run
get('/blogs/:blog_id', function ($blog_id) {
	// pick up what we got from the stash
	$blog = stash('blog');
	render('blogs/show', array('blog' => $blog);
});
?>
```

## Caching via APC
If you have `apc.so` enabled, you can make use of dispatch's simple caching functions.

```php
<?php
// fetch something from the cache (ttl param is 60)
$data = cache('users', function () {
  // this function is called as a loader if apc
  // doesn't have 'users' in the cache, whatever
  // it returns gets stored into apc and mapped to
  // the 'users' key
  return array('sheryl', 'addie', 'jaydee');
}, 60);

// invalidate our cached keys (users, products, news)
cache_invalidate('users', 'products', 'news');
```

## Configurations
You can make use of ini files for configuration by doing something like `config('source', 'myconfig.ini')`.
This lets you put configuration settings in ini files instead of making `config()` calls in your code.

```php
<?php
// load the contents of my-settings.ini into config()
config('source', 'my-settings.ini');

// set a different folder for the views
config('views.root', __DIR__.'/myviews');

// get the encryption secret
$secret = config('cookies.secret');
?>
```

## Utility Functions
There are a lot of other useful routines in the library. Documentation is still lacking but they're very small and easy to figure out. Read the source for now.

```php
<?php
// store a config and get it
config('views.root', './views');
config('views.root'); // returns './views'

// stash a var and get it (useful for moving stuff between scopes)
stash('user', $user);
stash('user'); // returns stored $user var

// redirect with a status code
redirect(302, '/index');

// redirect if a condition is met
redirect(403, '/users', !$authenticated);

// redirect only if func is satisfied
redirect('/admin', function () use ($auth) { return !!$auth; });

// redirect only if func is satisfied, and with a diff code
redirect(301, '/admin', function () use ($auth) { return !!$auth; });

// send a http error code and print out a message
error(403, 'Forbidden');

// get the current HTTP method or check the current method
method(); // GET, POST, PUT, DELETE
method('POST'); // true if POST request, false otherwise

// client's IP
client_ip();

// get a value from $_POST, returns null if not set
$name = from($_POST, 'name');

// create an associative array using the passed keys,
// pulling the values from $_POST
$user = from($_POST, array('username', 'email', 'password'));

// try to get a value from $_GET, use a default value if not set
$user = from($_GET, 'username', 'Sranger');

// set a flash message
flash('error', 'Invalid username');

// in a subsequent request, get the flash message
$error = flash('error');

// escape a string
_h('Marley & Me');

// url encode
_u('http://noodlehaus.github.com/dispatch');

// load a partial using some file and locals
$html = partial('users/profile', array('user' => $user));
?>
```

## Related Libraries
* [disptach-mongo](http://github.com/noodlehaus/dispatch-mongo) - wrapper for commonly used mongodb functions for dispatch
* [disptach-elastic](http://github.com/noodlehaus/dispatch-elastic) - wrapper for commonly used elasticsearch operations for dispatch

## Credits

The following projects served as both references and inspirations for Dispatch:

* [ExpressJS](http://expressjs.com)
* [Sinatra](http://sinatrarb.com)
* [BreezePHP](http://breezephp.com)

Thanks to the following contributors for helping improve this tool :)

* Kafene [kafene](https://github.com/kafene)
* Martin Angelov [martingalv](https://github.com/martinaglv)
* Lars [larsbo](https://github.com/larsbo)

## LICENSE
MIT http://noodlehaus.mit-license.org/
