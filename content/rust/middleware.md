+++
title="Middleware"
date=2023-09-26

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

I have been studying axum recently, and over the last two days I looked into how the `Service` trait was created.

[Link](https://tokio.rs/blog/2021-05-14-inventing-the-service-trait)

This is not a translation of that article. It is my own loose explanation written in a way that is easier for me to follow.

Suppose we have an API like this:

```rs
// Create a server that listens on port 3000
let server = Server::new("127.0.0.1:3000").await?;

// Somehow run the user's application
server.run(the_users_application).await?;
```

What would `the_users_application` look like?

```rs
fn handle_request(request: HttpRequest) -> HttpResponse {
    // ...
}
```

Here, `HttpRequest` and `HttpResponse` are not types provided by Tower. Assume they are structs provided by an example HTTP framework.

The `run` function would probably look like this.

```rs diff, hl_lines=4
impl Server {
    async fn run<F>(self, handler: F) -> Result<(), Error>
    where
        F: Fn(HttpRequest) -> HttpResponse,
    {
        let listener = TcpListener::bind(self.addr).await?;

        loop {
            let mut connection = listener.accept().await?;
            let request = read_http_request(&mut connection).await?;

            task::spawn(async move {
                // Call the handler provided by the user
                let response = handler(request);

                write_http_response(connection, response).await?;
            });
        }
    }
}
```

The `run` function receives a closure that takes an `HttpRequest` and returns an `HttpResponse`.

Then `handle_request` can be implemented like this.

```rs diff
fn handle_request(request: HttpRequest) -> HttpResponse {
    // ...
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

fn handle_request(request: HttpRequest) -> HttpResponse {
+    if request.path() == "/" {
+        HttpResponse::ok("Hello, World!")
+    } else {
+        HttpResponse::not_found()
+    }
}

server.run(handle_request).await?;
```

But with this design, the handler cannot process requests asynchronously. If we want it to do other work while waiting for a database lookup or external API call, we need to change it like this.

```rust diff
impl Server {
    async fn run<F>(self, handler: F) -> Result<(), Error>
    where
        F: Fn(HttpRequest) -> HttpResponse,
    {
        let listener = TcpListener::bind(self.addr).await?;

        loop {
            let mut connection = listener.accept().await?;
            let request = read_http_request(&mut connection).await?;

            task::spawn(async move {
                let response = handler(request);

                write_http_response(connection, response).await?;
            });
        }
    }
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

impl Server {
    async fn run<F, Fut>(self, handler: F) -> Result<(), Error>
    where
        // `handler` now returns a generic type `Fut`...
-        F: Fn(HttpRequest) -> HttpResponse,
+        F: Fn(HttpRequest) -> Fut,
+        // ...which is a `Future` whose `Output` is an `HttpResponse`
+        Fut: Future<Output = HttpResponse>,
    {
        let listener = TcpListener::bind(self.addr).await?;

        loop {
            let mut connection = listener.accept().await?;
            let request = read_http_request(&mut connection).await?;

            task::spawn(async move {
                // Await the future returned by `handler`
-                let response = handler(request);
+                let response = handler(request).await;

                write_http_response(connection, response).await?;
            });
        }
    }
}
```

Now that the server parameter changed, `handle_request` can also process requests asynchronously.

```rs diff
fn handle_request(request: HttpRequest) -> HttpResponse {
    if request.path() == "/" {
        HttpResponse::ok("Hello, World!")
    } else {
        HttpResponse::not_found()
    }
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

async fn handle_request(request: HttpRequest) -> HttpResponse {
    if request.path() == "/" {
        HttpResponse::ok("Hello, World!")
+    } else if request.path() == "/important-data" {
+        // We can now do async stuff in here
+        let some_data = fetch_data_from_database().await;
+        make_response(some_data)
    } else {
        HttpResponse::not_found()
    }
}
```

Let's upgrade the server's `run` function once more so it can handle errors.

```rs diff,hl_lines=28 39
impl Server {
    async fn run<F, Fut>(self, handler: F) -> Result<(), Error>
    where
        F: Fn(HttpRequest) -> Fut,
        Fut: Future<Output = HttpResponse>,
    {
        let listener = TcpListener::bind(self.addr).await?;

        loop {
            let mut connection = listener.accept().await?;
            let request = read_http_request(&mut connection).await?;

            task::spawn(async move {
                let response = handler(request).await;

                write_http_response(connection, response).await?;
            });
        }
    }
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

impl Server {
    async fn run<F, Fut>(self, handler: F) -> Result<(), Error>
    where
        F: Fn(HttpRequest) -> Fut,
        // The response future is now allowed to fail
-        Fut: Future<Output = HttpResponse>,
+        Fut: Future<Output = Result<HttpResponse, Error>>,
    {
        let listener = TcpListener::bind(self.addr).await?;

        loop {
            let mut connection = listener.accept().await?;
            let request = read_http_request(&mut connection).await?;

            task::spawn(async move {
                // Pattern match on the result of the response future
-                let response = handler(request).await;
-
-                write_http_response(connection, response).await?;
+                match handler(request).await {
+                    Ok(response) => write_http_response(connection, response).await?,
+                    Err(error) => handle_error_somehow(error, connection),
+                }
            });
        }
    }
}
```

## Adding features

Let's add a *timeout feature* and a feature that adds the `content-type: application/json` header.

Since `run` was changed above to receive `Result<HttpResponse, Error>`, assume from this point that `handle_request` also returns `Result<HttpResponse, Error>`.

Create a new handler that uses `handle_request`.

```rs
async fn handler_with_timeout(request: HttpRequest) -> Result<HttpResponse, Error> {
    let result = tokio::time::timeout(
        Duration::from_secs(30),
        handle_request(request)
    ).await;

    match result {
        Ok(Ok(response)) => Ok(response),
        Ok(Err(error)) => Err(error),
        Err(_timeout_elapsed) => Err(Error::timeout()),
    }
}
```

Now create a handler that uses `handler_with_timeout`.

```rs
async fn handler_with_timeout_and_content_type(
    request: HttpRequest,
) -> Result<HttpResponse, Error> {
    let mut response = handler_with_timeout(request).await?;
    response.set_header("Content-Type", "application/json");
    Ok(response)
}
```

This approach works, but it does not scale well.

We have added only two handlers so far, but if we add more later, we would need to hard-code the chain of intermediate handler calls every time a handler is added.

```rs
let final_handler = with_content_type(with_timeout(handle_request));
```

For example, `handler_with_timeout_and_content_type` needs `handler_with_timeout`, and `handler_with_timeout` needs `handle_request`. The order must be strict. Is there a way to compose these functions more flexibly?

```rs
async fn handle_request<F>(
    fn: F
) -> impl Fn(HttpRequest) -> impl Future<Output = Result<HttpResponse, Error>>
where
    F: Fn(HttpRequest) -> Future<Output = Result<HttpResponse, Error>>
{
}
```

It would be nice to write a handler-transforming function like this, but Rust cannot nest `impl Trait` in this position. In particular, `impl Fn() -> impl Future` is not possible.

We could work around it with `Box<dyn Future<...>>`, but that has heap allocation and dynamic dispatch costs, so I will leave it out here.

### Handler trait

Instead of making `Server::run` receive an `F: Fn(HttpRequest) -> Fut` closure, create a trait that encapsulates `async fn(HttpRequest) -> Result<HttpResponse, Error>`.

```rs diff, hl_lines=26 28
impl Server {
    async fn run<F, Fut>(self, handler: F) -> Result<(), Error>
    where
        F: Fn(HttpRequest) -> Fut,
        Fut: Future<Output = Result<HttpResponse, Error>>,
    {
        let listener = TcpListener::bind(self.addr).await?;

        loop {
            let mut connection = listener.accept().await?;
            let request = read_http_request(&mut connection).await?;

            task::spawn(async move {
                match handler(request).await {
                    Ok(response) => write_http_response(connection, response).await?,
                    Err(error) => handle_error_somehow(error, connection),
                }
            });
        }
    }
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

impl Server {
-    async fn run<F, Fut>(self, handler: F) -> Result<(), Error>
-    where
-        F: Fn(HttpRequest) -> Fut,
-        Fut: Future<Output = Result<HttpResponse, Error>>,
+    async fn run<T>(self, mut handler: T) -> Result<(), Error>
+    where
+        T: Handler,
    {
        let listener = TcpListener::bind(self.addr).await?;

        loop {
            let mut connection = listener.accept().await?;
            let request = read_http_request(&mut connection).await?;

            task::spawn(async move {
                // have to call `Handler::call` here
                match handler.call(request).await {
                    Ok(response) => write_http_response(connection, response).await?,
                    Err(error) => handle_error_somehow(error, connection),
                }
            });
        }
    }
}
trait Handler {
    async fn call(&mut self, request: HttpRequest) -> Result<HttpResponse, Error>;
}
```

When the original article was written in 2021, stable Rust did not support async methods in traits. Today, Rust 1.75+ has stabilized `async fn` in traits, but for a design like Tower's `Service`, where the future type is exposed as an associated type and `poll_ready` is included, the following approach still fits better.

1. A method returning `Pin<Box<dyn Future<Output = Result<HttpResponse, Error>>>>`
2. A `Handler` with `type Future` as an associated type

We can choose either approach.

Let's use the second one.

```rs
trait Handler {
    type Future: Future<Output = Result<HttpResponse, Error>>;

    fn call(&mut self, request: HttpRequest) -> Self::Future;
}
```

Now convert the existing `handle_request` function into the `Handler` trait.

```rs
async fn handle_request(request: HttpRequest) -> HttpResponse {
    if request.path() == "/" {
        HttpResponse::ok("Hello, World!")
    } else if request.path() == "/important-data" {
        let some_data = fetch_data_from_database().await;
        make_response(some_data)
    } else {
        HttpResponse::not_found()
    }
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

struct RequestHandler;

impl Handler for RequestHandler {
    // We use `Pin<Box<...>>` here for simplicity, but could also define our
    // own `Future` type to avoid the overhead
    type Future = Pin<Box<dyn Future<Output = Result<HttpResponse, Error>>>>;

    fn call(&mut self, request: HttpRequest) -> Self::Future {
        Box::pin(async move {
            // same implementation as we had before
            if request.path() == "/" {
                Ok(HttpResponse::ok("Hello, World!"))
            } else if request.path() == "/important-data" {
                let some_data = fetch_data_from_database().await?;
                Ok(make_response(some_data))
            } else {
                Ok(HttpResponse::not_found())
            }
        })
    }
}
```

### Timeout Handler

```rs
async fn handler_with_timeout(request: HttpRequest) -> Result<HttpResponse, Error> {
    let result = tokio::time::timeout(
        Duration::from_secs(30),
        handle_request(request)
    ).await;

    match result {
        Ok(Ok(response)) => Ok(response),
        Ok(Err(error)) => Err(error),
        Err(_timeout_elapsed) => Err(Error::timeout()),
    }
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

struct Timeout<T> {
    // T will be some type that implements `Handler`
    inner_handler: T,
    duration: Duration,
}

impl<T> Handler for Timeout<T>
where
    T: Handler,
{
    type Future = Pin<Box<dyn Future<Output = Result<HttpResponse, Error>>>>;

    fn call(&mut self, request: HttpRequest) -> Self::Future {
        Box::pin(async move {
            let result = tokio::time::timeout(
                self.duration,
                self.inner_handler.call(request),
            ).await;

            match result {
                Ok(Ok(response)) => Ok(response),
                Ok(Err(error)) => Err(error),
                Err(_timeout) => Err(Error::timeout()),
            }
        })
    }
}
```

This will produce a compile error.

```rs
144 |       fn call(&mut self, request: HttpRequest) -> Self::Future {
    |               --------- this data with an anonymous lifetime `'_`...
145 |           Box::pin(async move {
    |  _____________________________^
146 | |             let result = tokio::time::timeout(
147 | |                 self.duration,
148 | |                 self.inner_handler.call(request),
...   |
155 | |             }
156 | |         })
    | |_________^ ...is captured here, requiring it to live as long as `'static`
```

The error happens because `self` is captured into the async block and cannot live long enough.

Add `Clone` to the trait bound.

```rs diff
impl<T> Handler for Timeout<T>
where
-    T: Handler
+    T: Handler + Clone,
```

It still errors.

The compiler says a static lifetime is required.

```rs
140 |   impl<T> Handler for Timeout<T>
    |        - help: consider adding an explicit lifetime bound...: `T: 'static`
...
149 | /         Box::pin(async move {
150 | |             let result = tokio::time::timeout(
151 | |                 this.duration,
152 | |                 this.inner_handler.call(request),
...   |
159 | |             }
160 | |         })
    | |__________^ ...so that the type `impl Future` will meet its required lifetime bounds
```

Add `'static`. Now it compiles.

```rs diff
impl<T> Handler for Timeout<T>
where
-    T: Handler + Clone
+    T: Handler + Clone + 'static,
```

### Content-Type Handler

Do not forget to add `Clone` and `'static` to the `T` type.

```rs, hl_lines=9
#[derive(Clone)]
struct JsonContentType<T> {
    inner_handler: T,
}


impl<T> Handler for JsonContentType<T>
where
    T: Handler + Clone + 'static,
{
    type Future = Pin<Box<dyn Future<Output = Result<HttpResponse, Error>>>>;

    fn call(&mut self, request: HttpRequest) -> Self::Future {
        let mut this = self.clone();

        Box::pin(async move {
            let mut response = this.inner_handler.call(request).await?;
            response.set_header("Content-Type", "application/json");
            Ok(response)
        })
    }
}
```

Now it is easier to compose handlers.

```rs
let handler = RequestHandler;
let handler = Timeout::new(handler, Duration::from_secs(30));
let handler = JsonContentType::new(handler);

// `handler` has type `JsonContentType<Timeout<RequestHandler>>`

server.run(handler).await
```

## Making Handler more flexible

Our handler can currently handle only `HttpRequest`. Let's make a more generic handler.

```rs diff
trait Handler {
    type Future: Future<Output = Result<HttpResponse, Error>>;

    fn call(&mut self, request: HttpRequest) -> Self::Future;
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

trait Handler{+<Request>+} {
+    type Response;

    // Error should also be an associated type. No reason for that to be a
    // hardcoded type
+    type Error;

    // Our future type from before, but now it's output must use
    // the associated `Response` and `Error` types
-    type Future: Future<Output = Result<HttpResponse, Error>>;
+    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    // `call` is unchanged, but note that `Request` here is our generic
    // `Request` type parameter and not the `HttpRequest` type we've used
    // until now
    fn call(&mut self, request: Request) -> Self::Future;
}
```

### Request Handler

```rs diff
impl Handler for RequestHandler {
    type Future = Pin<Box<dyn Future<Output = Result<HttpResponse, Error>>>>;
    fn call(&mut self, request: HttpRequest) -> Self::Future {
    }
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

impl Handler{+<HttpRequest>+} for RequestHandler {
+    type Response = HttpResponse;
+    type Error = Error;
-    type Future = Pin<Box<dyn Future<Output = Result<HttpResponse, Error>>>>;
+    type Future = Pin<Box<dyn Future<Output = Result<HttpResponse, Error>>>>;

    fn call(&mut self, request: Request) -> Self::Future {
        // same as before
    }
}
```

### Timeout Handler

The timeout handler is a little different because it wraps another handler and also adds an async timeout.

As long as the wrapped handler has the same type, we do not need to care about the request and response types.

The error type is different. `tokio::time::timeout` returns `Result<T, tokio::time::error::Elapsed>`.

We need to convert `tokio::time::error::Elapsed` into the inner handler's error type, `T::Error`.

```rs diff
impl<T> Handler for Timeout<T>
where
    T: Handler,
{
    type Future = Pin<Box<dyn Future<Output = Result<HttpResponse, Error>>>>;

    fn call(&mut self, request: HttpRequest) -> Self::Future {
        Box::pin(async move {
            let result = tokio::time::timeout(
                self.duration,
                self.inner_handler.call(request),
            ).await;

            match result {
                Ok(Ok(response)) => Ok(response),
                Ok(Err(error)) => Err(error),
                Err(_timeout) => Err(Error::timeout()),
            }
        })
    }
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

impl<{+R,+}T> Handler{+<R>+} for Timeout<T>
where
    // The actual type of request must not contain
    // references. The compiler would tell us to add
    // this if we didn't
+    R: 'static,
    // `T` must accept requests of type `R`
-    T: Handler,
+    T: Handler<R> + Clone + 'static,
    // We must be able to convert an `Elapsed` into
    // `T`'s error type
+    T::Error: From<tokio::time::error::Elapsed>,
{
    // Our response type is the same as `T`'s, since we
    // don't have to modify it
+    type Response = T::Response;

    // Error type is also the same
+    type Error = T::Error;

    // Future must output a `Result` with the correct types
    type Future = Pin<Box<dyn Future<Output = Result<T::Response, T::Error>>>>;

    fn call(&mut self, request: R) -> Self::Future {
        let mut this = self.clone();

        Box::pin(async move {
            let result = tokio::time::timeout(
                this.duration,
                this.inner_handler.call(request),
            ).await;

            match result {
                Ok(Ok(response)) => Ok(response),
                Ok(Err(error)) => Err(error),
                Err(elapsed) => {
                    // Convert the error
                    Err(T::Error::from(elapsed))
                }
            }
        })
    }
}
```

### Content-Type Handler

`JsonContentType` is also different from the previous two handlers. It does not care about the request or error type, but it does care about the response type.

The response type must be able to call `set_header`.

```rs diff
impl<T> Handler for JsonContentType<T>
where
    T: Handler + Clone + 'static,
{
    type Future = Pin<Box<dyn Future<Output = Result<HttpResponse, Error>>>>;

    fn call(&mut self, request: HttpRequest) -> Self::Future {
        let mut this = self.clone();

        Box::pin(async move {
            let mut response = this.inner_handler.call(request).await?;
            response.set_header("Content-Type", "application/json");
            Ok(response)
        })
    }
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

impl<{+R,+} T> Handler{+<R>+} for JsonContentType<T>
where
+    R: 'static,
    // `T` must accept requests of any type `R` and return
    // responses of type `HttpResponse`
-    T: Handler + Clone + 'static,
+    T: Handler<R, Response = HttpResponse> + Clone + 'static,
{
+    type Response = HttpResponse;

    // Our error type is whatever `T`'s error type is
+    type Error = T::Error;

    type Future = Pin<Box<dyn Future<Output = Result<Response, T::Error>>>>;

    fn call(&mut self, request: R) -> Self::Future {
        let mut this = self.clone();

        Box::pin(async move {
            let mut response = this.inner_handler.call(request).await?;
            response.set_header("Content-Type", "application/json");
            Ok(response)
        })
    }
}
```

```rs diff
impl Server {
    async fn run<T>(self, mut handler: T) -> Result<(), Error>
    where
        T: Handler,
    {

    }
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

impl Server {
    async fn run<T>(self, mut handler: T) -> Result<(), Error>
    where
-        T: Handler,
+        T: Handler<HttpRequest, Response = HttpResponse>,
    {
        // ...
    }
}
```

## The appearance of the Service trait

The `Handler` trait can be used on both servers and clients. Since it can be used by both, the name `Handler` is not appropriate. A client does not "handle" a request. So let's call it `Service` instead.

```rs diff
trait Handler<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn call(&mut self, request: Request) -> Self::Future;
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

-trait Handler<Request> {
+trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn call(&mut self, request: Request) -> Self::Future;
}
```

This is close to the `Service` trait defined by Tower. Tower already provides implemented services such as `Timeout`, `Retry`, and `RateLimit`.

Types like `Timeout` and `JsonContentType` are called middleware because they wrap another service. Types like `RequestHandler` are called leaf services because they sit at the end of a nested service stack. The response is created by the leaf service, and middleware modifies it.

## Backpressure

Backpressure happens when the production rate is faster than the consumption rate.

Imagine building a `ConcurrencyLimit` middleware that sets the maximum number of requests processed concurrently. It would be useful to have a service that enforces an upper bound on how much load can be processed.

```rs
impl<R, T> Service<R> for ConcurrencyLimit<T> {
    fn call(&mut self, request: R) -> Self::Future {
        // 1. Check a counter for the number of requests currently being
        //    processed.
        // 2. If there is capacity left send the request to `T`
        //    and increment the counter.
        // 3. If not somehow wait until capacity becomes available.
        // 4. When the response has been produced, decrement the counter.
    }
}
```

If capacity is full, the request must wait until a slot opens. But if requests are held in memory while waiting, memory is wasted.

With a method like this, requests do not need to be queued up.

```rs
trait Service<R> {
    async fn ready(&mut self);
}
```

Before calling `service.call(request).await`, we can call `service.ready().await` to check whether there is capacity and save memory.

When the original article was written in 2021, async functions could not be used in traits. Today they can, but adding another associated type such as `ReadyFuture` and returning a future can create the same kind of lifetime problem as before. We can borrow an idea from the `Future` trait: use a `poll_ready` function.

```rs
use std::task::{Context, Poll};

trait Service<R> {
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<()>;
}
```

If the service does not have enough capacity, `poll_ready` returns `Poll::Pending` and uses the `Context` waker to notify the caller when capacity becomes available again.

Because `poll_ready` does not return a future, we can check readiness quickly without waiting. If we call `poll_ready` and receive `Poll::Pending`, we may decide to do something else instead of waiting. More importantly, this lets us build load balancers by evaluating how often a service returns `Poll::Pending`.

Communicating capacity from callee to caller like this is called backpressure propagation. It is like telling the caller, "There are too many requests, please slow down." Other ways to handle backpressure include buffering and load shedding.

Finally, an error can happen while reserving capacity, so `poll_ready` should return `Poll<Result<(), Self::Error>>`.

```rs
trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn call(&mut self, request: Request) -> Self::Future;
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn poll_ready(
        &mut self,
        cx: &mut Context<'_>,
    ) -> Poll<Result<(), Self::Error>>;

    fn call(&mut self, req: Request) -> Self::Future;
}
```

This completes Tower's `Service` trait.

Many middleware types do not add their own backpressure and instead reuse the `poll_ready` implementation of the service they wrap. But backpressure in middleware has meaningful uses, such as rate limiting, load balancing, and auto scaling.

Finally, this is the most common way to use a service.

```rs
use tower::{
    Service,
    // for the `ready` method
    ServiceExt,
};

let response = service
    // wait for the service to have capacity
    .ready().await?
    // send the request
    .call(request).await?;
```
