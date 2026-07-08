+++
title="Posting with Ajax in Razor Pages"
date=2025-06-23

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

I was coding peacefully when something ate more than two hours of my time, so I am writing down what happened.

I was uploading a picture from a Razor Page using Ajax. I had only used simple handlers like `OnGet` and `OnPost` before. Since I wanted to upload an image without submitting a form, Ajax seemed like the only reasonable option. I tried sending a simple request to the server, but it did not work as expected.

### Named handler method

The upload handler was named `UploadPicture`, but the request returned 404 Not Found.
After some searching, I found that Razor Pages has a feature called [named handler methods](https://www.learnrazorpages.com/razor-pages/handler-methods#named-handler-methods).

When creating a handler, start the method name with `OnPost` for a POST request or `OnGet` for a GET request, then append the custom name.
For example, my handler was a POST request named `UploadPicture`, so the method should be named `OnPostUploadPicture`.

### Anti-forgery token

Strangely, even after fixing the handler name, the request still failed. I changed the method to `OnGetUploadPicture` as a test, and the GET request worked.

I searched again and found the answer [here](https://www.talkingdotnet.com/handle-ajax-requests-in-asp-net-core-razor-pages/). Razor Pages automatically applies antiforgery validation to unsafe HTTP methods such as POST, so an AJAX POST also needs to send the request verification token. If the token is missing or invalid, the request usually fails before it reaches the handler, often as 400 Bad Request.

I added this under the input element:

```cs
@Html.AntiForgeryToken()
```

And added this to the Ajax request:

```js
beforeSend: function (xhr) {
    xhr.setRequestHeader("XSRF-TOKEN",
        $('input:hidden[name="__RequestVerificationToken"]').val());
},
```

### Header

Even after adding the anti-forgery token, I still received 404. After searching again, I realized I had missed this setup:

```cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc();
    services.AddAntiforgery(o => o.HeaderName = "XSRF-TOKEN");
}
```

If you want to change the header name in the modern hosting model, you usually configure it in `Program.cs` like this:

```cs
builder.Services.AddAntiforgery(options =>
{
    options.HeaderName = "XSRF-TOKEN";
});
```

The default form field name is `__RequestVerificationToken`, and when sending it as a header, the header name must match what the app's antiforgery configuration expects. I did not want to touch `Program.cs`, so I changed the Ajax request's header name to `RequestVerificationToken`, and it worked.

References:

https://www.talkingdotnet.com/handle-ajax-requests-in-asp-net-core-razor-pages/
https://www.mikesdotnetting.com/article/308/razor-pages-understanding-handler-methods
https://www.learnrazorpages.com/security/request-verification
https://learn.microsoft.com/en-us/aspnet/core/razor-pages/
https://learn.microsoft.com/en-us/aspnet/core/security/anti-request-forgery
