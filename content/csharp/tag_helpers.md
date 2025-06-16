+++
title="Tag Helpers"
date=2025-06-16

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

Razor page를 사용한지 오래되었는데 몇년만에 다시 사용해봤다.  
User input을 받는 form을 만들고 있는데 form에 Input.Email이 바인딩 안되어서 한참을 고생했다.  
[BindProperty]도 썼고, cshtml, cshtml.cs 파일명도 일치하고... 그런데 왜 submit 버튼을 누르면 404가 뜰까?
```html
<div class="form-group">
    <label asp-for="Input.Email">email</label>
    <input asp-for="Input.Email" class="form-control" />
    <span asp-validation-for="Input.Email" class="text-danger"></span>
</div>
```

예제 코드를 ide에 넣고 들려보니 input이 InputTagHelper이고 HtmlFragmentElemet attribute가 붙어있는 것을 발견했다.  
다시 내 코드로 돌아가보니 내 input은 그냥 html element였다.  
예제코드를 다시 훑어보았고 _ViewImports.cshtml 파일에 @addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers가 선언되있는걸 발견했다.  
[@addTagHelper](https://learn.microsoft.com/en-us/aspnet/core/mvc/views/tag-helpers/intro?view=aspnetcore-9.0) *, Microsoft.AspNetCore.Mvc.TagHelpers 를 파일 맨 위에 추가하고서야 email을 바인딩 할 수 있었다.  
마소기술 특히 razor에는 convention이 많으니 혹시 razor page를 처음 접하게 되는 사람이 있다면 마소 공식 홈페이지를 한번 다 읽어보는게 도움될 것 같다.

