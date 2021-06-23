# php-http-service

This is an experiment where I take some of Mat Ryer's Go code and translate it into PHP to judge whether the insights that inspired Go's (and its standard library's) design are at all generalizeable.

I chose PHP because I'm intimately familiar with it and it's a terrible language. In particular, it has a bafflingly poor type system and (partially as a result of its poor type system) every one of its builtins not only does _all the things_ it does them in surprising ways. What should be simple functions like `array_map` are instead are loaded footguns.

My hypothesis is that by writing PHP like Go, though not just naively translating Go idioms, borrowing patterns from its standard library, and ignoring all of PHP's language features and builtins, you'll end up with better PHP than if you wrote PHP like PHP. I'd also wager Go-like PHP will perform at least as well as PHP-ic PHP, if not better.

Before reading this, see <https://github.com/AndrewLivingston/mr-http-service>, which provides companion code connected to timecodes in a video recording of a GopherCon talk Mat Ryer gave on writing HTTP services.

## main.php

[mr-http-service/main.go](https://github.com/AndrewLivingston/mr-http-service/blob/main/main.go)

#### The main function

[mr-http-service main()](https://github.com/AndrewLivingston/mr-http-service/blob/main/main.go#L14)

The main function can be translated naively (mostly) and look pretty reasonable, at least to those of us who prefer to eschew the current Java-esque OO style in PHP and force the language to live up to its (half-true) claim to have first-class functions.

###### Go
```go
func main() {
    if err := run(); err != nil {
        fmt.Fprintf(os.Stderr, "%s\n", err)
        os.Exit(1)
    }
}
```

##### PHP
```php
function main(): void {
    try {
        run();
    } catch (\Throwable $e) {
        exit($e->getMessage());
    }
}
```

The one important difference is that PHP handles errors errors with `try`/`catch`. You _can_ pass exception objects around as values but, unlike the Go compiler, PHP won't complain if you fail to handle an exception unless you throw it. Ie, a more naive translation makes it easy to shoot ourselves in the foot:

```php
function main(): void {
    $err = run(); // run() returns \Throwable
    // ... oops, forgot to:
    // if ($err) exit($err->getMessage())
}
```

####

Here PHP has to diverge more from the go code.

##### Go
```go
func run() (err error) {
    db, dbtidy, err := setupDatabase()
    if err != nil {
        return errors.Wrap(err, "setup database")
    }
    defer dbtidy()
    // ...
}
```

###### PHP
```php
function run(): void {  // throws
    try {
        $db = setupDatabase();
    } catch (\Throwable $e) {
        throw \Errors\wrap(
            $e, 'setup database', \Exception::class
        );
    } finally {
        dbtidy();
    }
    // ...
}
```

Using `finally` instead of `defer` isn't a big difference: `defer` is basically a function-wide `finally`. In fact, if we didn't want `dbtidy` appearing in every `finally` block in `run` and also at the end, we could (and probably would), handle it in where `run` is called: in `main`.

```php
function main(): void {
    try {
        run();
    } catch (\Throwable $e) {
        exit($e->getMessage());
    } finally {
        dbtidy();
    }
}
```

Another minor difference is that `Errors\wrap` (presumably an analogue to Dave Cheney's `errors.Wrap`) has to accept an exception class as its third argument, otherwise it wouldn't know which exception class to wrap the previous error in. Ah, for the simplicity of Go with its single `Error` interface. I've provided the base exception class, `Exception`.

I could make this class argument to `wrap` optional with a default of `Exception`, and many PHP devs might, but I'm not a fan of optional arguments in general. When I'm considering adding parameters to a function I prefer to be forced to decide whether I need a config struct/object or whether I need more functions.

The big difference beetween Go and PHP here is that  `setupDatabase` only returns a `DbConn` instead of returning both a `DbConn` and a `dbtidy` cleanup function. To see why, consider a more naive translation.

```php
function run(): void {  // throws
    try {
        list($db, $dbtidy) = setupDatabaseNaive();
    } catch (\Throwable $e) {
        throw \Errors\wrap($e, 'setup database', get_class($e));
    } finally {
        $dbtidy();
    }
    // ...
}

function setupDatabaseNaive(): array {
    return [new DbConn, new \Exception];
}
```

PHP doesn't care if you mix types inside arrays, so you can fake Go's multiple returns, but at a cost: you can't declare the types of elements in arrays, so a static type checker doesn't have any idea what `$db` and `$dbtidy` are.

But there's a larger problem: because PHP uses `try`/`catch` for error handling, there's no way for a PHP function that throws to partially succeed. Let's say you change the `$dbtidy()` call in the `finally` to just printing the value of the `$dbtidy` variable.

```php
finally {
    error_log(json_encode($dbtidy));
    // json because PHP prints all falsey values as ''
}
```

If you ran your naive version on the command line, you'd get the following:

```
PHP Warning:  Undefined variable $dbtidy
null
```

Because `setupDatabaseNaive` threw before its result could be assigned to `$dbtidy`, that variable never existed. Note that this is not a problem for PHP (version 8): it just logs a warning and fumbles onward taking the value of the undefined variable to be `null`. (Did I mention PHP is a terrible language?)

So we have little option but to make `dbtidy` a named function and call it explicitly. This may or may not be a problem in any given use case, but it does mean in some cases you can't stucture the code the way you think is best.
