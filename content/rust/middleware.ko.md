+++
title="Middleware"
date=2023-09-26

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

요즘 axum을 공부하고 있는데, 지난 이틀 동안은 `Service` trait가 어떻게 만들어졌는지 살펴봤다.

[링크](https://tokio.rs/blog/2021-05-14-inventing-the-service-trait)

내가 이해하기 편하도록 풀어서 적은 글이지, 위 article을 그대로 번역한 글은 아니다.


아래와 같은 API가 있다고 하자.
```rs
// Create a server that listens on port 3000
let server = Server::new("127.0.0.1:3000").await?;

// Somehow run the user's application
server.run(the_users_application).await?;
```

`the_users_application`은 어떻게 생겼을까?
```rs
fn handle_request(request: HttpRequest) -> HttpResponse {
    // ...
}
```
위에 `HttpRequest`와 `HttpResponse`는 Tower가 제공하는 타입이 아니라, 예시로 만든 HTTP 프레임워크가 제공한다고 가정한 구조체이다.

`run` 함수는 아래와 같이 생겼을 것이다.
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

`run` 함수는 `HttpRequest`를 받아 `HttpResponse`를 return하는 closure를 parameter로 받는다.
그러면 `handle_request` 함수는 아래와 같이 구현할 수 있다.
```rs diff
fn handle_request(request: HttpRequest) -> HttpResponse {
    // ...
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

fn handle_request(request: HttpRequest) -> HttpResponse {
+    if request.path() == "/" {
+        HttpResponse::ok("Hello, World!")
+    } else {
+        HttpResponse::not_found()
+    }
}

server.run(handle_request).await?;
```

하지만 이 설계에서는 handler가 요청을 비동기적으로 처리할 수 없다. DB 조회나 외부 API 호출을 기다리는 동안 다른 작업을 처리할 수 있게 하려면 아래와 같이 바꿔야 한다.
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

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

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

server parameter가 바뀌었으니 `handle_request`도 비동기 처리를 할 수 있게 되었다.

```rs diff

fn handle_request(request: HttpRequest) -> HttpResponse {
    if request.path() == "/" {
        HttpResponse::ok("Hello, World!")
    } else {
        HttpResponse::not_found()
    }
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

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

server의 `run` 함수가 error를 처리할 수 있도록 한 번 더 업그레이드해 보자.
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

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

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


## 기능 추가

*timeout 기능*과 *`Content-Type: application/json` header를 추가하는 기능*을 붙여 보자.

앞에서 `run`이 `Result<HttpResponse, Error>`를 받도록 바뀌었으므로, 여기서부터 `handle_request`도 `Result<HttpResponse, Error>`를 반환한다고 가정한다.

`handle_request`를 사용하는 새로운 handler를 만든다.
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

그리고 `handler_with_timeout`을 사용하는 handler를 하나 더 만든다.
```rs
async fn handler_with_timeout_and_content_type(
    request: HttpRequest,
) -> Result<HttpResponse, Error> {
    let mut response = handler_with_timeout(request).await?;
    response.set_header("Content-Type", "application/json");
    Ok(response)
}
```

이 방식은 동작하긴 하지만 확장하기 쉽지 않다.
지금은 handler를 2개만 추가했지만, 나중에 handler가 더 늘어나면 매번 중간 handler 호출 chain을 직접 하드 코딩해야 한다.
```rs
let final_handler = with_content_type(with_timeout(handle_request));
```
예를 들어 `handler_with_timeout_and_content_type` 함수는 `handler_with_timeout` 함수를 필요로 하고, `handler_with_timeout` 함수는 `handle_request` 함수를 필요로 한다. 순서를 엄격하게 지켜야 하는데, 이 함수들을 더 유연하게 compose할 방법은 없을까?

```rs
async fn handle_request<F>(
    fn: F
) -> impl Fn(HttpRequest) -> impl Future<Output = Result<HttpResponse, Error>>
where
    F: Fn(HttpRequest) -> Future<Output = Result<HttpResponse, Error>>
{
}
```
이런 형태의 핸들러 변환 함수를 쓸 수 있다면 좋겠지만, Rust에서는 `impl Trait`를 이런 위치에 중첩해서 쓸 수 없다. 특히 `impl Fn() -> impl Future` 형태는 불가능하다.
`Box<dyn Future<...>>`를 사용해서 우회할 수는 있지만 heap allocation과 dynamic dispatch 비용이 있으므로 여기서는 배제한다.

### Handler trait

`Server::run`이 `F: Fn(HttpRequest) -> Fut` closure를 직접 받게 하지 말고, `async fn(HttpRequest) -> Result<HttpResponse, Error>` 형태를 캡슐화하는 trait를 만들자.
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

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

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

원문이 쓰인 2021년에는 Rust stable에서 async 메소드를 가진 trait를 지원하지 않았다. 현재 Rust 1.75+에서는 `async fn` in trait가 안정화되었지만, Tower의 `Service`처럼 future 타입을 associated type으로 노출하고 `poll_ready`까지 포함하는 설계에는 아래 방식이 더 잘 맞는다.
1. `Pin<Box<dyn Future<Output = Result<HttpResponse, Error>>>>`을 return하는 method
2. `type Future`를 associated type으로 가지는 `Handler`
둘 중 하나의 방식을 택할 수 있다.

두 번째 방식으로 가자.
```rs
trait Handler {
    type Future: Future<Output = Result<HttpResponse, Error>>;

    fn call(&mut self, request: HttpRequest) -> Self::Future;
}
```

기존의 `handle_request` 함수를 `Handler` trait 구현으로 바꿔주자.
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

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

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

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

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


이렇게 하면 compile error가 뜰 것이다.
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
`self`가 async block 안으로 들어가면서, 필요한 lifetime만큼 살아남지 못해서 생기는 error다.


trait bound에 `Clone`을 추가해 보자.
```rs diff
impl<T> Handler for Timeout<T>
where
-    T: Handler
+    T: Handler + Clone,
```
그래도 error가 난다.
compiler는 `'static` lifetime이 필요하다고 말한다.
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

`'static`을 추가하자. 이제 compile된다.
```rs diff
impl<T> Handler for Timeout<T>
where
-    T: Handler + Clone
+    T: Handler + Clone + 'static,
```


###  Content-Type Handler

`T` type에 `Clone`, `'static`을 추가하는 것을 잊지 말자.
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


이제 handler들을 합성하기 쉬워졌다.
```rs
let handler = RequestHandler;
let handler = Timeout::new(handler, Duration::from_secs(30));
let handler = JsonContentType::new(handler);

// `handler` has type `JsonContentType<Timeout<RequestHandler>>`

server.run(handler).await
```

## Handler를 더 유연하게...
현재 handler는 `HttpRequest`만 다룰 수 있다. 좀 더 generic한 handler를 만들어보자.
```rs diff
trait Handler {
    type Future: Future<Output = Result<HttpResponse, Error>>;

    fn call(&mut self, request: HttpRequest) -> Self::Future;
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

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

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

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

timeout handler는 조금 다르다. 다른 handler를 감싸고 있고 비동기 timeout도 추가하기 때문이다.
감싸고 있는 handler가 같은 타입을 유지하는 한, 요청과 응답 타입에는 크게 신경 쓰지 않아도 된다.

error type은 다르다. `tokio::time::timeout`은 `Result<T, tokio::time::error::Elapsed>`를 return한다.
우리는 `tokio::time::error::Elapsed`를 내부 handler의 error type인 `T::Error`로 변환할 수 있어야 한다.
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

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

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
`JsonContentType` handler도 앞의 두 handler와는 조금 다르다. 요청과 error type은 크게 신경 쓰지 않지만, response type은 신경 써야 한다.
response type은 반드시 `set_header`를 호출할 수 있는 타입이어야 한다.
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

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

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

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

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

## Service trait의 등장
`Handler` trait는 server에서도 client에서도 사용할 수 있다. server와 client 양쪽에서 쓰일 수 있다면 `Handler`라는 이름은 부적절하다. client는 요청을 "handle"하는 입장이 아니기 때문이다. 그러니 `Handler` 대신 `Service`라고 부르자.
```rs diff
trait Handler<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn call(&mut self, request: Request) -> Self::Future;
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

-trait Handler<Request> {
+trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn call(&mut self, request: Request) -> Self::Future;
}
```
이제 Tower가 정의하는 `Service` trait에 가까워졌다. Tower에는 이미 구현된 `Timeout`, `Retry`, `RateLimit` 같은 service들이 있다.

`Timeout`과 `JsonContentType` 같은 타입은 middleware라고 부른다. 다른 service를 감싸기 때문이다. `RequestHandler` 같은 타입은 leaf service라고 부른다. 중첩된 service들 중 가장 말단에 있기 때문이다. response는 leaf service에서 생성되고, middleware는 그 response를 감싸거나 변형한다.

## 배압
backpressure는 생산 속도를 소비 속도가 따라가지 못할 때 발생한다.
동시에 처리할 수 있는 요청 수의 최댓값을 제한하는 `ConcurrencyLimit` middleware를 만든다고 생각해보자. 처리 가능한 부하의 상한선을 지켜주는 service가 있으면 좋을 것이다.
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
정원이 차면 자리가 빌 때까지 기다려야 한다. 하지만 요청을 메모리에 계속 대기시키면 메모리 사용량이 커진다.

아래 같은 method가 있으면 요청을 무작정 줄 세워 둘 필요가 없다.
```rs
trait Service<R> {
    async fn ready(&mut self);
}
```
`service.call(request).await`를 하기 전에 `service.ready().await`로 capacity가 있는지 확인하면 메모리를 아낄 수 있다.

원문이 쓰인 2021년에는 async 함수를 trait에서 사용할 수 없었다. 현재는 가능하지만, `ReadyFuture`라는 associated type을 하나 더 추가해서 Future를 리턴하면 예전과 같은 lifetime 문제를 야기할 수 있다. 우리는 Future trait에서 아이디어를 얻을 수 있다. 바로 poll_ready 함수를 사용하는 것이다.

```rs
use std::task::{Context, Poll};

trait Service<R> {
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<()>;
}
```
service capacity가 부족하면 `poll_ready`는 `Poll::Pending`을 return하고, capacity가 다시 생기면 `Context`의 waker를 사용해 caller에게 알려준다.
`poll_ready`가 `Future`를 return하지 않는다는 것은 기다리지 않고 ready 상태를 빠르게 확인할 수 있다는 뜻이다. `poll_ready`를 호출했는데 `Poll::Pending`을 받으면, 기다리는 대신 다른 일을 하기로 결정할 수도 있다. 무엇보다 service가 얼마나 자주 `Poll::Pending`을 return하는지 평가해서 load balancer를 만들 수도 있다.
이런 식으로 caller와 capacity 상태를 소통하는 것을 backpressure propagation이라고 한다. caller에게 요청이 너무 많으니 속도를 줄이라고 알려주는 것과 같다. backpressure를 다루는 다른 방법으로는 buffering과 load shedding이 있다.

마지막으로 capacity를 예약하는 동안 error가 발생할 수 있으므로 `poll_ready`는 `Poll<Result<(), Self::Error>>`를 return해야 한다.

```rs
trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn call(&mut self, request: Request) -> Self::Future;
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

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
이렇게 Tower의 `Service` trait가 완성되었다.

많은 middleware는 자신만의 backpressure를 추가하지 않고, 감싸고 있는 service의 `poll_ready` 구현을 그대로 사용한다. 하지만 middleware 수준의 backpressure도 의미 있는 사용처가 있다. 예를 들면 rate limiting, load balancing, auto scaling 같은 경우다.

마지막으로 아래는 service를 사용하는 가장 흔한 방식이다.
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
