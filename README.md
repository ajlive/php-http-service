# php-http-service

This is an experiment where I take some of Mat Ryer's Go code and translate it into PHP to judge whether the insights that inspired Go's (and its standard library's) design are at all generalizeable.

I chose PHP because I'm intimately familiar with it and it's a terrible language. In particular, it has a bafflingly poor type system and (partially as a result of its poor type system) every one of its builtins not only does _all the things_ it does them in surprising ways. Everywhere you turn, you find that what should be a simple family of functions (`array_map`, `array_filter`, and `array_reduce` for example) is instead a bunch of loaded footguns.

My hypothesis is that by writing PHP somewhat like Go (and not just naively translating Go idioms), borrowing patterns from Go's standard library, and ignoring as many of PHP's language features and builtins as you can, you'll end up with better PHP than if you wrote PHP like PHP. I'd also wager Go-like PHP will perform at least as well as PHP-ic PHP, if not better.

Before reading this, see <https://github.com/AndrewLivingston/mr-http-service>, which provides companion code connected to timecodes in a video recording of a GopherCon talk Mat Ryer gave on writing HTTP services.

## The main function

[mr-http-service main()](https://github.com/AndrewLivingston/mr-http-service/blob/main/main.go#L14)

#### Go
```go
func main() {
    if err := run(); err != nil {
        fmt.Fprintf(os.Stderr, "%s\n", err)
        os.Exit(1)
    }
}
```

The main function can be translated naively (mostly) and look pretty reasonable, at least to those of us who prefer to eschew the current Java-esque OO style in PHP and force the language to live up to its half-true claim to have first-class functions (more on that later).

### PHP
```php
function main(): void {
    try {
        run();
    } catch (\Throwable $e) {
        exit($e->getMessage());
    }
}
```

The one important difference is that PHP handles errors with `try`/`catch`. You _can_ pass exception objects around as values but, unlike the Go compiler, PHP won't complain if you fail to handle an exception that's never thrown. A more naive translation makes it easy to shoot ourselves in the foot.

```php
function main(): void {
    $err = run(); // run() returns Throwable
    // ... oops, forgot to:
    // if ($err) exit($err->getMessage())
}
```

## run and setupDatabase functions

### Go
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
(From [mr-http-service run()](https://github.com/AndrewLivingston/mr-http-service/blob/main/main.go#L21))

I'd argue that pulling all but last-resort error handling out of your `main` function and into a `run` function is a good idea in any language, even if one that uses exceptions. So this pattern is something I'd adopt from Go if I wasn't already using it.

But in `run` PHP has to diverge more from the go code than it had to in `main`.

#### PHP
```php
function run(): void {  // throws
    try {
        $db = setupDatabase();
    } catch (\Throwable $e) {
        throw Errors\wrap(
            $e, 'setup database', Exception::class
        );
    } finally {
        dbtidy();
    }
    // ...
}
```

Using `finally` instead of `defer` isn't a big difference: `defer` is basically a function-scoped `finally`. In fact, if we didn't want `dbtidy` appearing in every `finally` block in `run` and also at the end, we could (and probably would), call it in a `finally` block where `run` is called--in this case inside `main`.

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

Another minor difference is that `Errors\wrap` (which I would write as an analogue to Dave Cheney's [`errors.Wrap`](https://github.com/pkg/errors/blob/master/errors.go#L184)) has to accept an exception class as its third argument, otherwise it wouldn't know which exception class to wrap the previous error in. Ah, for the simplicity of Go with its single `Error` interface. PHP's `Throwable` interface is similar, but as usual inheritance means we can't have nice things. So I added the class parameter and here I've provided it with th base exception class, `Exception`.

(I could make this class argument to `wrap` optional with a default of `Exception`, and many PHP devs might, but I'm not a fan of optional arguments in general. When I'm considering adding parameters to a function I prefer to be forced to decide whether I need a config struct/object or whether I need more functions.)

The big difference beetween Go and PHP here is that `setupDatabase` only returns a `DbConn` instead of returning both a `DbConn` and a `dbtidy` cleanup function. To see why, consider a more naive translation.

```php
function run(): void {  // throws
    try {
        list($db, $dbtidy) = setupDatabaseNaive();
    } catch (\Throwable $e) {
        throw Errors\wrap($e, 'setup database', get_class($e));
    } finally {
        $dbtidy();
    }
    // ...
}

function setupDatabaseNaive(): array {
    return [new DbConn, new Exception];
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

## Server

[mr-http-service server type](https://github.com/AndrewLivingston/mr-http-service/blob/main/server.go#L8)
[mr-http-service server creation](https://github.com/AndrewLivingston/mr-http-service/blob/main/main.go#L27)

Mat suggests using a "server" struct to hold what might otherwise be global variables like the database connection. This is another pattern that makes sense in any language.

### Go
```go
type server struct {
    db     *dbConn
    router *router
    email  EmailSender
}

// ServeHTTP makes server satisfy the http.Handler
// interface
func (s *server) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    s.router.ServeHTTP(w, r)
    // ^ delegates to router for implementation. Also,
    // ServeHTTP is intended to panic on error, with
    // recovery happening in middleware
}
```

Mat wasn't explicit, but I assume that `EmailSender` is an interface because it's an agent noun (ends in "er") and and it isn't a pointer. `router` is an agent noun, but the fact that `s.router` is a pointer tells us that `router` is a type.

We can write a PHP version that is nearly identical and is (almost) idomatic PHP.

### PHP
```php
class Server implements Http\Handler {
    public DbConn $db;
    public Router $router;
    public EmailSender $email;

    // serveHTTP method from Http\Handler
    public function serveHTTP(ResponseWriter $w, Request $r): void {
        $this->router->serveHTTP($w, $r);
        // ^ assume this call can
        // throw new Exception(
        //     'throwing matches Go panic behavior'
        // );
    }
}
```

PHP uses doesn't have pointers, so there's no hint from the syntax that `DbConn` and `Router` are types (classes, specifically) while EmailSender is an interface.

Unfortunately, PHP types must declare that they implement an interface. But that's really the only difference. Since `ServeHTTP` is one of relatively few functions in the Go standard library that is designed to panic instead of returning an error, PHP's error handling idiom is appropriate here.

Matt shows [one way of populating the server struct inside run()](https://github.com/AndrewLivingston/mr-http-service/blob/main/main.go#L27).

```go
srv := &server{
    db:     db, // from setup   Database
    router: &router{},
    email:  emailSender{}, // concrete implementation
}
```

This method avoids using a constructor, which is nicely explicit. There's nothing hidden from you behind a function. You can do this in PHP, but it's not idiomatic.

```php
$srv = new Server;
$srv->db = $db;
$srv->router = new Router;
$srv->email = new EmailSenderConcrete;
```

One problem is that at least some PHP static type checkers may not detect that the wrong type of variable is being assigned, say, a string for `router`, which then becomes a runtime error. So it's best to follow the PHP idiom of providing a constructor.

Mat says he always seems to end up with [a constructor for his server types](https://github.com/AndrewLivingston/mr-http-service/blob/main/server.go#L20) since some setup/initialization is required.

```go
func newServer() *server {
    s := &server{}
    // ^ leaves dependencies as nil (could be passed as
    // arguments if not many, but better to explicitly
    // assign later)
    s.routes() // more on this later
    return s
}
```

So it's not a big leap from Mat's parameter-less constructor to a PHP constructor with parameters.

```php
class Server implements Http\Handler {
    public function __construct(
        public DbConn $db,
        public Router $router,
        public EmailSender $email,
    ) {
        routes($this);
    }

    public function serveHTTP(ResponseWriter $w, Request $r): void {
        $this->router->serveHTTP($w, $r);
    }
}

// in run()
$srv = new Server(
    db: $db,
    router: new Router,
    email: new EmailSenderConcrete,
);
```

## The routes function and handlers

[mr-http-service routes.go](https://github.com/AndrewLivingston/mr-http-service/blob/main/routes.go)

Mat puts all his routes and their registration inside a separate routes.go
```go
func (s *server) routes() {
    s.router.Get("/api/", s.handleAPI())
    s.router.Get("/about", s.handleAbout())
    s.router.Get("/", s.handleIndex())
    s.router.Post("/greet", s.handleGreeting("Hello %s"))
    // ...
}

// handlers have the form
func (s *server) handleSomething(params...) http.HandlerFunc {
    // ...
    return func(w http.ResponseWriter, r *http.Request) {
        // ...
    }
}

// concrete example
func (s *server) handleGreeting(format string) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        name := r.FormValue("name")
        greeting := fmt.Sprintf(format, name)
        fmt.Fprint(w, greeting)
        emailAddress := s.db.Query(
            "SELECT address FROM emails WHERE name = ?",
            name,
        )
        s.email.Send(emailAddress, "Geetings!", greeting)
    }
}

// ...
```

Go's syntax and semantics for methods (single dispatch polymorphic functions) is one of my favorite things about the language.

```go
func (t *dispatchType) name(params...) returnType {
    body
}
```

1. You don't declare a method on the type as you do in OO languages.
1. You don't even have to declare a method in the same file as the type.
1. The syntax makes it _very_ clear which paramater is the dispatch parameter.
1. You can dispatch on _any_ named type, including functions, maps, slices, etc! You can even do:

```go
type myInt int

func (n *myInt) successor() int {
    return int(*n) + 1
}

func main() {
    n := myInt(5)
    fmt.Println(n.successor())
    // 6
}
```

Obviously methods on functions and slices are more useful than the above, but it's neat that you can define methods on an `int` alias.

Mat uses this feature of Go to allow him to both:

* make handlers methods on the `server` type to give them easy access to the db, router, emailer, etc, while reserving the handlers' parameters for additional values or dependencies each handler might need; and
* declare handlers wherever it makes sense to declare them, whether in `routes.go` or elsewhere.

PHP doesn't allow both these things at the same time. Which is unfortunate, but not the end of the world. Since you certainly would not put all your apps' handlers onto the `Server` class, the obvious thing to do is make the first argument to every handler a `Server`.

In the following PHP code, assume I've written the interfaces `Http\ResponseWriter` and `Http\Request`.

```php
function routes(Server $s): void {
    $s->router->post('/greet', handleGreeting($s, 'Hello, %s!'));
    // ...
}

// handlers have the form
function handleSomething(Server $s, ...$params) Closure {
    // ...
    return function(w http.ResponseWriter, r *http.Request) use ($params) {
        // ...
    }
}

// a concrete example
function handleGreeting(Server $s, string $format): Closure {
    return function(Http\ResponseWriter $w, Http\Request $r) use ($s, $format) {
        $name = $r->formValue("name");
        $greeting = sprintf($format, $name);

        $w->write($greeting);

        $emailAddress = $s->db->query(
            "SELECT address FROM emails WHERE name = ?",
            $name,
        );
        $s->email->send($emailAddress, "Greetings!", $greeting);
    };
}
```

It's a little clunkier, particularly because I have to explicitly `use` the dependency parameters, but otherwise doesn't look bad.

But...it is bad. It's all thanks to the return type `Closure`. Similar to the restrictions on array type declarations, function type declarations are too general to be very useful: you can't declare the parameter types or return type of the function, so returning any anonymous function of any kind will make PHP happy—runtime-error happy.

### A digression

If you're not familiar with Go, you may be asking, "so...what's an `http.HandlerFunc`?" Buckle up, because we're about to leverage Go's dispatch semantics to follow in the Go standard library's footsteps and do something awesome.

Go's `http` package provides a `Handler` interface.

```go
package http

type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
    // ^ panic on error
}
```

`ResponseWriter` is itself an interface declaring a `Write` method, and `Request` is a concrete type in the `http` package. But we won't concern ourselves with them right now. Suffice it to say that `Request` is a model of an http request, and `ResponseWriter` is responsible for responding to the http request.

Because it's an interface, you're free to write your own type, say a struct of some kind, that implements `Handler`. But, since most request handlers are most clearly written as simple functions, Go declares a concrete type, `HandlerFunc`...that implements `Handler`! I never get tired of thinking about this. It's just so good.

```go
// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers. If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// Handler that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```

This is the actual source code for `HandlerFunc` in <https://golang.org/src/net/http/server.go>. You can define concrete methods on _interfaces as well as types_ in Go, so by defining that very simple `ServeHTTP`, a `HandlerFunc` is also a `Handler`.

Furthermore, since `HandlerFunc` is just a type alias for `func(ResponseWriter, *Request)`, any function with that signature is a `HandlerFunc`

Recap:

1. Go's `http` package uses the `Handler` interface everywhere in its implementation.
1. `HandlerFunc` is a type alias for `func(ResponseWriter, *Request)`, so any function with that signature is a `HandlerFunc`.
2. The `ServeHTTP` method is defined on a `HandlerFunc`, so a `HandlerFunc` is a `Handler`.
3. Therefore, any function with signature `func(ResponseWriter, *Request)` is an `http.Handler`. QED.

Go has done more than any other language I've used to demonstrate that, when done correctly, simple, static typing can be as flexible as dynamic typing. The extra structure might might make a dynamic-loving dev think, "why not just use a dynamic language?" But, dynamic dev using a dynamic language, you have to use TDD and write eight unit tests for every function before you can actually write that function lest your code explode into bug meat in production. Your first draft design for a module's API may suck, but since you developed using TDD, you'll never change it. You'll just live with the suck. Also, you don't actually do TDD because nobody does.

On the other hand, using Go I can play around with my API until I get it right, try implementing it, change my mind, and keep iterating (quickly, since Go's compiler is blazing fast). Then when I'm happy, I write a test or two to ensure I don't have null pointer errors, index out of bounds errors, or value errors.

(What do you call a large codebase written in a duck-typed language? A ducksterfuck.)

### PHP can't have nice things, or, PHP needs Handlers

Specifically, PHP can't have a `HandlerFunc`. To get statically checked handlers, we need to write a `Handler` interface and then make concrete implementations.

```php
namespace Http;

interface Handler {
    public function serveHTTP(ResponseWriter $w, Request $r): void;
    // ^ throw on error
}

// elsewhere
final class HandlerGreet implements Http\Handler {
    public function __construct(
        private Server $server,
        private string $format,
    ) {}

    public function serveHTTP(Http\ResponseWriter $w, Http\Request $r): void {
        $s = $this->s;
        $format = $this->format;
        $name = $r->formValue("name");
        $greeting = sprintf($format, $name);

        $w->write($greeting);

        $emailAddress = $s->db->query(
            "SELECT address FROM emails WHERE name = ?",
            $name,
        );
        $s->email->send($emailAddress, "Greetings!", $greeting);
    }
}

// routes.php
function routes(Server $s): void {
    $s->router->post('/greet', new HandlerGreet($s, 'Hello, %s!'));
    // ...
}
```

People who like OO will probably think this looks better anyway, and I understand. And I pity you. But! This is probably the simplest reasonable handler pattern you can get in PHP that type checks. It adds the unnecessary indirection of passing the server and format through the constructor to instance properties, but that's the worst of it.

Because we're using such a simple interface you'd hope nobody who came to this code later would be tempted to create inheritance relationships between handlers. But you simply can't trust diehard—or inexperienced—OO devs. It should be obvious that a `Handler` is just a wrapper around a single function. It should be.

It should be.

## Middleware

[mr-http-service middleware](https://github.com/AndrewLivingston/mr-http-service/blob/main/middleware.go#L5-L28)

Middleware is also easy to write simply in Go using decorators.

```go
// adminOnly is a decorator that simply passes along a HandlerFunc, first
// ensuring the user has admin access
func (s *server) adminOnly(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // can run code before calling wrapped handler h:
        if !currentUser(r).IsAdmin {
            http.NotFound(w, r)
            return // don't call wrapped handler h at all
        }
        h(w, r)
        // can also run code after calling wrapped handler h
    }
}

// example of using middleware on an admin route:
func (s *server) routes() {
    s.router.Get("/admin", s.adminOnly(s.handleAdminIndex()))
    // ...
}
```

We can do almost as well with PHP at the expense of using an anonymous handler class.

```php
function adminOnly(Http\Handler $h): Http\Handler {
    return new class($h) implements Http\Handler {
        public function __construct(
            private Http\Handler $h,
        ) {}

        public function serveHTTP(Http\ResponseWriter $w, Http\Request $r): void {
            $h = $this->h;
            $user = new CurrentUser($r);
            if (! $user->IsAdmin) {
                Http\notFound($w, $r);
                return;
            }
            $h->serveHTTP($w, $r);
        }
    };
}

function routes(Server $s): void {
    $s->router->get('/admin', adminOnly(new HandlerAdminIndex($s)));
}
```

Look at all those `Http\Handler`s peppering that code...

This works just fine, but anonymous classes aren't fun to see in stack traces. Likely we'd make a named handler class instead.

```php
function adminOnly(Http\Handler $h): Http\Handler {
    return new _HandlerMiddlewareAdminOnly($h);
}

class _HandlerMiddlewareAdminOnly implements Http\Handler {
    public function __construct(
        private Http\Handler $h,
    ) {}

    public function serveHTTP(Http\ResponseWriter $w, Http\Request $r): void {
        $h = $this->h;
        $user = new CurrentUser($r);
        if (! $user->IsAdmin) {
            Http\notFound($w, $r);
            return;
        }
        $h->serveHTTP($w, $r);
    }
}
```

Once again PHP proves that it's able to add more complexity than Go at the expense of more verbosity. But I like this well enough.

Either way I'll never have to see a handler class for middleware show up directly in `routes`.

## But would you really rewrite Go standard library interfaces in PHP?

My answer is an unqualified yes. Let me show you one of my other favorite things in Go's standard library.

```go
package io

// Writer is the interface that wraps the basic Write method.
//
// Write writes len(p) bytes from p to the underlying data stream.
// It returns the number of bytes written from p (0 <= n <= len(p))
// and any error encountered that caused the write to stop early.
// Write must return a non-nil error if it returns n < len(p).
// Write must not modify the slice data, even temporarily.
//
// Implementations must not retain p.
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

This is the source for `io.Writer`. Because Go got interfaces right its standard library contains some of the most beautiful abstractions I've ever seen, and `io.Writer` is one of the best of a good bunch.

`io.Writer` unifies all notions of output behind a single abstraction: taking some bytes and ejecting them out somehwere in the wider world. And because it's used everywhere in the standard library, you'll discover that _Go's standard library is far more reusable than any dynamic language's standard library._ Static typing combined with a desire for flexibility forced interfaces on Go, and interfaces forced awesomeness on the standard library.

For example, the `fmt` package includes this function:
```go
func Fprint(w io.Writer, a ...interface{}) (n int, err error)
```

This means that you can write your own `io.Writer` that, say, outputs text to a giant laser pointed at the moon and write your own name up there using only the print functions in Go's standard library.

```go
fmt.Fprint(w, "PROPERTY OF ELON MUSK")
```

Hey, Python, where's my `fprint(writer, *vals)`?? You're dynamic. You're duck-typed. Isn't this the kind of abstraction people who defend dynamic languages would point to as an advantage of dynamism?

Python could easily have an `fprint`. But it doesn't—possibly _because_ it's dynamic. Without restrictions, without boundaries, creativity suffers. Static typing forced Go to think very carefully about abstractions, and as a result (and because Go is _very_ smart) it came up with a perfected notion of interface (the first such?). And as a result of that, we have `io.Writer`.

(There are also the expected `fmt.Print` functions that will let Elon Musk claim stdout as his property.)

I'm not actually ragging on Python. It was invented roughly a million years ago and it still has better abstractions and a better standard library than three quarters of the languages invented since, including many statically typed languages with some borked notion of interface. That's a hell of a legacy. And it's still my second favorite dynamic language behind Clojure.

(Much of what I like about both of them are their data types and the literal syntax for those types. _Where's my set literal literally every other programming language?_)
