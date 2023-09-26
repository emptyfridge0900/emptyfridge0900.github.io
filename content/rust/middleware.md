+++
title="Middleware"
date=2023-09-26

[taxonomies]
categories = ["Post"]
tags = ["post","Rust"]
+++

요즘 axum을 공부하고 있는데 지난 이틀동안은 Service trait가 어떻게 만들어 졌는지 살펴보았다.

[링크](https://tokio.rs/blog/2021-05-14-inventing-the-service-trait)

내가 보기에 편하게 주절주절 써놓은 것이지 위의 아티클을 번역한 건 아니다.


아래와 같은 api가 있다고 치자
```rs
// Create a server that listens on port 3000
let server = Server::new("127.0.0.1:3000").await?;

// Somehow run the user's application
server.run(the_users_application).await?;
```

the_users_application은 어떻게 생겼을까?
```rs
fn handle_request(request: HttpRequest) -> HttpResponse {
    // ...
}
```
위에 HttpRequest와 HttpResponse는 Tower 라이브러리에서 제공되는 구조체이다.

run함수는 아래와 같이 생겼을 것이다.
```rs, hl_lines=4
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

run함수는 HttpRequest를 받아서 HttpResponse를 return하는 cosure를 파라미터로 받는다.
그럼 handle_request함수는 아래와 같이 구현할 수 있다.
```rs
fn handle_request(request: HttpRequest) -> HttpResponse {
    // ...
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

fn handle_request(request: HttpRequest) -> HttpResponse {
    if request.path() == "/" {
        HttpResponse::ok("Hello, World!")
    } else {
        HttpResponse::not_found()
    }
}

server.run(handle_request).await?;
```

하지만 우리의 서버는 비동기적으로 요청을 받을 수 없다. 그러니 아래와 같이 바꿔주자
```rs, hl_lines=26 28
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
        F: Fn(HttpRequest) -> Fut,
        // ...which is a `Future` whose `Output` is an `HttpResponse`
        Fut: Future<Output = HttpResponse>,
    {
        let listener = TcpListener::bind(self.addr).await?;

        loop {
            let mut connection = listener.accept().await?;
            let request = read_http_request(&mut connection).await?;

            task::spawn(async move {
                // Await the future returned by `handler`
                let response = handler(request).await;

                write_http_response(connection, response).await?;
            });
        }
    }
}
```

서버 파라미터가 바뀌었으니 handle_request도 비동기 처리를 할수 있게되었다

```rs

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
    } else if request.path() == "/important-data" {
        // We can now do async stuff in here
        let some_data = fetch_data_from_database().await;
        make_response(some_data)
    } else {
        HttpResponse::not_found()
    }
}
```

서버의 run함수가 error를 처리할수 있게 한번 더 업그레이드 해주자
```rs,hl_lines=28 38-41
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
        Fut: Future<Output = Result<HttpResponse, Error>>,
    {
        let listener = TcpListener::bind(self.addr).await?;

        loop {
            let mut connection = listener.accept().await?;
            let request = read_http_request(&mut connection).await?;

            task::spawn(async move {
                // Pattern match on the result of the response future
                match handler(request).await {
                    Ok(response) => write_http_response(connection, response).await?,
                    Err(error) => handle_error_somehow(error, connection),
                }
            });
        }
    }
}
```


## 기능 추가 

*타임아웃 기능*과 그리고 *content-type:application/json을 헤더에 추가하는 기능*을 추가해보자

handle_request를 사용하는 새로운 handler를 만들자
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

그리고 handler_with_timeout을 사용하는 handler를 만들자
```rs
async fn handler_with_timeout_and_content_type(
    request: HttpRequest,
) -> Result<HttpResponse, Error> {
    let mut response = handler_with_timeout(request).await?;
    response.set_header("Content-Type", "application/json");
    Ok(response)
}
```

이 방식은 잘 작동하겠지만 스케일 하기에는 쉽지 않다.
지금은 2개의 handler를 추가 했지만 나중에 몇개의 handler를 더 추가 한다면 handler를 추가할 때마다 중간 핸들러 호출의 체인을 하드 코딩해야 하므로 추가하기 힘들다.
```rs
let final_handler = with_content_type(with_timeout(handle_request));
```
예를 들어 handler_with_timeout_and_content_type 함수는 handler_with_timeout 함수를 필요로 하고 handler_with_timeout 함수는 handle_request 함수를 필요로 한다. 그래서 엄격하게 순서를 지켜야하는데 이 함수들을 유연하게 compose하는 방법이 없을까?

```rs
async fn handle_request<F>(
    fn: F
) -> impl Fn(HttpRequest) -> impl Future<Output = Result<HttpResponse, Error>> 
where
    F: Fn(HttpRequest) -> Future<Output = Result<HttpResponse, Error>>
{
}
```
이런 핸들러를 쓸 수 있다면 좋을텐데 rust에서는 안된다. 특히 impl Fn() -> impl Future 는 불가능하다.
Box를 사용해서 리턴할수는 있지만 퍼포먼스가 느려져서 배제한다.

### Handler trait

Server::run이 F: Fn(HttpRequest) -> Fut 클로저를 받아들이게 하지 말고 async fn(HttpRequest) -> Result<HttpResponse, Error> 을 캡슐화 하는 Trait를 만들자
```rs, hl_lines=26 28
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
    async fn run<T>(self, mut handler: T) -> Result<(), Error>
    where
        T: Handler,
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

하지만 rust는 async 매소드를 가진 trait를 지원하지 않아서
1. Pin<Box<dyn Future<Output = Result<HttpResponse, Error>>> 을 리턴하는 매소드
2. type Future을 associated type으로 가지는 Handler
둘중 하나의 방식을 택할 수 있다.

두번째 방식으로 하자
```rs
trait Handler {
    type Future: Future<Output = Result<HttpResponse, Error>>;

    fn call(&mut self, request: HttpRequest) -> Self::Future;
}
```

기존의 handle_request 함수를 Handler trait로 바꿔주자
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


컴파일 에러가 뜰것이다
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
self가 async block으로 빨려들어가서 lifetime이 끝까지 살아남지 못해서 생긴 에러


trait bound에 clone을 추가해주자
```rs, hl_lines=3
impl<T> Handler for Timeout<T>
where
    T: Handler + Clone,
```
그래도 에러가 뜬다.
컴파일러가 말하길 static lifetime이 필요하다고 한다
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

'static을 추가해주자. 이제 컴파일 잘 된다
```rs, hl_lines=3
impl<T> Handler for Timeout<T>
where
    T: Handler + Clone + 'static,
```


###  Content-Type Handler

T Type에 Clone, 'static 추가해주는걸 잊지말자
```rs, hl_lines=8
#[derive(Clone)]
struct JsonContentType<T> {
    inner_handler: T,
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

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


이제는 hadler들을 합성하기에 수월해졌다
```rs
let handler = RequestHandler;
let handler = Timeout::new(handler, Duration::from_secs(30));
let handler = JsonContentType::new(handler);

// `handler` has type `JsonContentType<Timeout<RequestHandler>>`

server.run(handler).await
```

## Handler를 더 유연하게...
우리의 handler는 현재 HttpRequest만 다룰 수있다. 좀더 generic한 handler를 만들어보자
```rs
trait Handler {
    type Future: Future<Output = Result<HttpResponse, Error>>;

    fn call(&mut self, request: HttpRequest) -> Self::Future;
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

trait Handler<Request> {
    type Response;

    // Error should also be an associated type. No reason for that to be a
    // hardcoded type
    type Error;

    // Our future type from before, but now it's output must use
    // the associated `Response` and `Error` types
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    // `call` is unchanged, but note that `Request` here is our generic
    // `Request` type parameter and not the `HttpRequest` type we've used
    // until now
    fn call(&mut self, request: Request) -> Self::Future;
}
```
### Request Handler
```rs
impl Handler for RequestHandler {
    type Future = Pin<Box<dyn Future<Output = Result<HttpResponse, Error>>>>;
    fn call(&mut self, request: HttpRequest) -> Self::Future {
    }
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

impl Handler<HttpRequest> for RequestHandler {
    type Response = HttpResponse;
    type Error = Error;
    type Future = Pin<Box<dyn Future<Output = Result<HttpResponse, Error>>>>;

    fn call(&mut self, request: Request) -> Self::Future {
        // same as before
    }
}
```
### Timeout Handler

타임아웃 핸들러는 좀 다르다. 다른 핸들러를 감싸고 있고 비동기 timeout도 추가했기 때문이다.
감싸고 있는 핸들러가 같은 타입을 가지고 있는한 요청과 응답 타입에는 신경쓰지 않아도 된다.

에러타입은 좀 다르다. tokio::time::timeout은 Result<T, tokio::time::error::Elapsed> 을 리턴한다
우리는 tokio::time::error::Elapsed 타입을 내부 핸들러의 에러타입(T::Error)으로 변환해야한다
```rs

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

impl<R, T> Handler<R> for Timeout<T>
where
    // The actual type of request must not contain
    // references. The compiler would tell us to add
    // this if we didn't
    R: 'static,
    // `T` must accept requests of type `R`
    T: Handler<R> + Clone + 'static,
    // We must be able to convert an `Elapsed` into
    // `T`'s error type
    T::Error: From<tokio::time::error::Elapsed>,
{
    // Our response type is the same as `T`'s, since we
    // don't have to modify it
    type Response = T::Response;

    // Error type is also the same
    type Error = T::Error;

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
JsonContentType Handler도 앞의 두 핸들러와는 좀 다르다. 요청과 에러 타입에 대해서는 신경 안쓰지만 응답 타입에 대해서는 신경써야한다.
응답 타입은 반드시 set_header를 콜 할수 있는 타입이여야 한다.
```rs
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

impl<R, T> Handler<R> for JsonContentType<T>
where
    R: 'static,
    // `T` must accept requests of any type `R` and return
    // responses of type `HttpResponse`
    T: Handler<R, Response = HttpResponse> + Clone + 'static,
{
    type Response = HttpResponse;

    // Our error type is whatever `T`'s error type is
    type Error = T::Error;

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


```rs
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
        T: Handler<HttpRequest, Response = HttpResponse>,
    {
        // ...
    }
}
```

## Service trait의 등장
Handler trait는 server에서도 client에서도 사용될 수 있다. server, client 둘다 사용 될 수 있기 때문에 Handler 라는 이름은 부적절하다. client는 요청을 handle하지 않기 때문이다. 그러니 Handler대신 Service라고 부르자
```rs
trait Handler<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn call(&mut self, request: Request) -> Self::Future;
}

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn call(&mut self, request: Request) -> Self::Future;
}
```
이것은 Tower에서 정의하는 Service trait에 근접했다. Tower에서는 이미 구현되있는 Timout, Retry, RateLimit 같은 service들이 존재한다.

Timeout 과 JsonContentType 같은 타입을 middleware라고 부른다. 얘들은 다른 service를 감싸기 때문. Request Handler같은 타입은 leaf service라고 부른다. 중첩된 서비스들 중 말단에 위치해있기 때문이다. 응답은 leaf service에서 생성되고 변조는 middleware에서 일어난다.

## 배압
배압은 생성하는 속도를 소모하는 속도가 못 따라갈때 일어난다.
동시 처리하는 요청의 최대 값을 설정 하는 rate limit middleware을 만든다고 생각해보자. 처리할수 있는 부하의 양의 상한선을 지켜주는 서비스가 있으면 좋을 것이다.
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
정원이 차면 자리가 빌때까지 기다려야하는데 요청을 메모리에 대기 시키기 때문에 메모리 손실이 일어남. 

이러한 method가 있으면 요청을 줄세워 놓은 필요가 없음.
```rs
trait Service<R> {
    async fn ready(&mut self);
}
```
service.call(request).await 하기 전에 service.ready().await 로 자리가 있는지 확인하면 메모리를 아낄수 있다.

하지만 async 함수를 trait에서 사용하지 못한다. ReadyFuture 이라는 associated type을 하나 더 추가하는 방법도 생각해볼 수 있지만 Future를 리턴하면 예전과 같은 lifetime 문제를 야기할수 있다. 우리는 Future trai에서 아이디어를 얻을 수 있다. 바로 poll_ready 함수를 사용하는 것이다.

```rs
use std::task::{Context, Poll};

trait Service<R> {
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<()>;
}
```
만약 서비스가 수용량이 부족하면 poll_ready가 Poll::Pending을 리턴하고 caller에게 capacity가 다시 가능하게 되면 Context의 waker를 사용하여 통지한다.
poll_ready가 Future를 리턴하지 않는다는 말은 우리는 기다리지 않고 신속하게 ready 상태를 체크할수 있다는 말이다. 만약 우리가 poll_ready를 부르고 Poll::Pending을 리턴받으면, 기다리는 대신 다른 일을 하기로 결정할 수도 있다. 무엇보다도 이것은 서비스가 얼마나 자주 Poll::Pending을 리턴하는지 평가해서 load balancer를 만들수 있게 해준다. 
이런식으로 caller와 capacity에 대해 소통하는 것을 backpressure propagation이라고 한다. caller한테 요청이 너무 많으니 좀 줄이라고 말하는 것과 같다. 다른 방법으로 backpressure를 다루는 방법은 buffering, load shedding 이 있다.

마지막으로 capacity 예비를 하는 동안 에러가 발생할 수 있으므로 poll_ready 는 Poll<Result<(), Self::Error>> 를 리턴해야 할 것이다.

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
이렇게 tower의 Service trait가 완성되었다.

많은 middleware들이 자신만의 backpresure를 추가 하지 않고 자기가 감싸고 있는 service의 poll_ready 의 구현을 가져다 쓴다. 하지만 middleware에서의 backpressure는 의미있는 사용법들이 있다. 예를 들어 rate limiting, load balancing, 그리고 auto scaling등이다.

마지막으로 아래는 service를 사용하는 제일 흔한방법이다
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