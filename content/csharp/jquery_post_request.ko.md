+++
title="Razor Page에서 Ajax로 Post 요청하기"
date=2025-06-23

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

오늘도 평화롭게 코딩하고 있었는데, 내 시간을 2시간 넘게 잡아먹은 일이 생겨서 간단히 기록해 둔다.

Razor Page에서 AJAX를 사용해 사진을 업로드하는 작업을 하고 있었다. 항상 `OnGet`, `OnPost`처럼 단순한 handler만 사용하다가, form 제출 없이 사진을 업로드하려고 하니 AJAX를 써야 할 것 같았다. 그래서 서버에 간단한 요청을 날려 보려고 했는데 생각대로 되지 않았다.

### Named handler method
사진을 업로드하는 handler 이름은 `UploadPicture`였는데 404 Not Found가 났다.
검색해 보니 Razor Page에는 [named handler methods](https://www.learnrazorpages.com/razor-pages/handler-methods#named-handler-methods)라는 기능이 있었다.

POST request를 처리하려면 handler 이름을 `OnPost`로 시작하고, GET request를 처리하려면 `OnGet`으로 시작한 뒤 원하는 이름을 붙이면 된다.
예를 들어 내 handler는 POST request이고 이름은 `UploadPicture`였으므로 `OnPostUploadPicture`라고 작성해야 한다.

### Anti Forgery Token
이상하게도 handler 이름이 정확한데 여전히 요청이 실패했다. 그래서 이름을 `OnGetUploadPicture`로 바꿔 보았는데 GET 요청은 잘 되었다.

다시 검색했고 [여기서](https://www.talkingdotnet.com/handle-ajax-requests-in-asp-net-core-razor-pages/) 답을 찾았다. Razor Pages는 POST 같은 unsafe HTTP method에 대해 antiforgery validation을 자동으로 적용한다. 따라서 AJAX POST에도 request verification token을 함께 보내야 한다. 일반적으로 token이 없거나 잘못되면 handler까지 도달하지 못하고 400 Bad Request가 난다.

나는 input element 아래에 다음 코드를 추가했다.
```cs
@Html.AntiForgeryToken()
```
그리고 AJAX request에는 아래 코드를 추가했다.
```js
beforeSend: function (xhr) {
    xhr.setRequestHeader("XSRF-TOKEN",
        $('input:hidden[name="__RequestVerificationToken"]').val());
},
```

### Header
Antiforgery token을 추가했는데도 404가 반환되어 다시 찾아보니, 내가 아래 설정을 빼먹고 있었다.
```cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc();
    services.AddAntiforgery(o => o.HeaderName = "XSRF-TOKEN");
}
```
header name을 바꾸고 싶다면 modern hosting model에서는 보통 `Program.cs`에서 아래처럼 설정한다.

```cs
builder.Services.AddAntiforgery(options =>
{
    options.HeaderName = "XSRF-TOKEN";
});
```

기본 form field 이름은 `__RequestVerificationToken`이고, header로 보낼 때는 app의 antiforgery 설정이 기대하는 header name과 맞아야 한다. 나는 `Program.cs` 설정을 건드리기 귀찮아서 AJAX request의 header name을 `RequestVerificationToken`으로 바꾸었더니 잘 동작했다.


참고한 자료
https://www.talkingdotnet.com/handle-ajax-requests-in-asp-net-core-razor-pages/
https://www.mikesdotnetting.com/article/308/razor-pages-understanding-handler-methods
https://www.learnrazorpages.com/security/request-verification
https://learn.microsoft.com/en-us/aspnet/core/razor-pages/
https://learn.microsoft.com/en-us/aspnet/core/security/anti-request-forgery
