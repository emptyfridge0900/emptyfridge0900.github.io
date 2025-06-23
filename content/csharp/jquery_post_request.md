+++
title="Razor Page에서 Ajax로 Post 요청하기"
date=2025-06-23

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

오늘도 평화롭게 코딩하고 있는데 내 시간을 2시간 넘게 잡아먹는 일이 생겨서 어떤 일이 생겼는지 간단 하게 적어본다.  

Razor page에서 ajax를 사용하여 사진을 업로드 하는 작업을 하고 있었다. 항상 OnGet, OnPost 같이 간단한 함수만 사용하다가 form 제출 없이 사진 업로드를 하려고 하다보니 ajax외에는 방법이 없는거 같았다. 그래서 서버에 간단한 요청을 날리는 것을 해보려고 했는데 생각대로 되지 않았다.  
### Named handler method
사진업로드하는 handler이름이 UploadPicture였는데 404 not found 에러가 났다.  
구글링을 좀 해보니 razor page에는 [named handler methods](https://www.learnrazorpages.com/razor-pages/handler-methods#named-handler-methods)라는 기능이 있었다.
Handler를 만들때 post request를 보내고 싶으면 OnPost, get request를 보내고 싶으면 OnGet으로 핸들러 이름을 시작하고, 그 뒤에 원하는 이름을 적으면 된다.  
예를 들어 나의 핸들러는 Post request이고 UploadPicture라는 이름의 함수였으니까 OnPostUploadPicture라고 이름하면 된다.

### Anti Forgery Token
이상하게도 핸들러 이름이 분명 정확한데 여전히 404가 반환되었다. 그래서 이름을 GetOnUploadPicture로 바꾸어 봤는데 Get요청 이건 잘 되었다. 
또 구글링을 했고 [여기서](https://www.talkingdotnet.com/handle-ajax-requests-in-asp-net-core-razor-pages/) 답을 찾았다. razor page는 자동으로 csrf 공격을 방어하게 되있고 anti forgery token을 추가해주어야한다.  
나는 그냥 input element 아래에 
```cs
@Html.AntiForgeryToken() 
```
추가해주었다. 그리고 ajax request에는 아래의 코드를 추가했다
```js
beforeSend: function (xhr) {
    xhr.setRequestHeader("XSRF-TOKEN",
        $('input:hidden[name="__RequestVerificationToken"]').val());
},
```

### Header  
Anti forgery token을 추가했는데도 404가 반환되어서 다시 구글링해보니, 내가
```cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc();
    services.AddAntiforgery(o => o.HeaderName = "XSRF-TOKEN");
}
```
이부분을 빼먹어서 그랬다. 기본적인 이름은 RequestVerificationToken 로 설정되고 만약 헤더이름을 바꾸고 싶다면 Program.cs 설정하는 부분에서 위의 코드를 사용하여 anti forgery token 헤더이름을 바꾸면된다. 나는 Program.cs에 설정 건드리기 귀찮아서 그냥 토큰 헤더이름을 RequestVerificationToken 로 바꾸어 주었더니 잘 동작했다.


참고한 자료  
https://www.talkingdotnet.com/handle-ajax-requests-in-asp-net-core-razor-pages/  
https://www.mikesdotnetting.com/article/308/razor-pages-understanding-handler-methods  
https://www.learnrazorpages.com/security/request-verification  