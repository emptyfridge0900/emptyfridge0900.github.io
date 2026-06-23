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

예제 코드를 IDE에 넣고 보니 `<input asp-for="...">`가 단순 HTML input이 아니라 `InputTagHelper`의 대상이 되는 markup이라는 것을 발견했다. Tag Helper는 C#으로 작성되고, element name이나 attribute name을 기준으로 Razor markup에 붙어서 HTML 생성을 도와준다.  
다시 내 코드로 돌아가보니 내 input은 그냥 HTML element였다.  
예제코드를 다시 훑어보았고 `_ViewImports.cshtml` 파일에 `@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers`가 선언되있는걸 발견했다.  
`@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers`를 `_ViewImports.cshtml`에 추가하고서야 `asp-for="Input.Email"`이 `InputTagHelper`로 처리되었고 email을 바인딩 할 수 있었다.  
마소기술 특히 razor에는 convention이 많으니 혹시 razor page를 처음 접하게 되는 사람이 있다면 마소 공식 홈페이지를 한번 다 읽어보는게 도움될 것 같다.

### Ref

- Tag Helpers in ASP.NET Core: <https://learn.microsoft.com/en-us/aspnet/core/mvc/views/tag-helpers/intro>
- Input Tag Helper: <https://learn.microsoft.com/en-us/aspnet/core/mvc/views/working-with-forms#the-input-tag-helper>
