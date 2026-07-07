+++
title="Tag Helpers"
date=2025-06-16

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

Razor Page를 사용한 지 오래되었는데, 몇 년 만에 다시 사용해 보았다.
User input을 받는 form을 만들고 있었는데 `Input.Email`이 바인딩되지 않아서 한참을 고생했다.
`[BindProperty]`도 붙였고, `.cshtml`과 `.cshtml.cs` 파일명도 일치했다. 그런데 왜 submit 버튼을 누르면 404가 뜨는 걸까?
```html
<div class="form-group">
    <label asp-for="Input.Email">email</label>
    <input asp-for="Input.Email" class="form-control" />
    <span asp-validation-for="Input.Email" class="text-danger"></span>
</div>
```

예제 코드를 IDE에 넣고 보니 `<input asp-for="...">`가 단순한 HTML input이 아니라 `InputTagHelper`의 대상이 되는 Razor markup이라는 것을 발견했다. Tag Helper는 C#으로 작성되고, element name이나 attribute name을 기준으로 Razor markup에 적용되어 HTML 생성을 도와준다.

다시 내 코드로 돌아가 보니, 내 input은 Tag Helper가 적용된 element가 아니라 그냥 HTML element였다.
예제 코드를 다시 훑어보았고, `_ViewImports.cshtml` 파일에 `@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers`가 선언되어 있는 것을 발견했다.

`@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers`를 `_ViewImports.cshtml`에 추가하고 나서야 `asp-for="Input.Email"`이 `InputTagHelper`로 처리되었고, email을 바인딩할 수 있었다.

Microsoft 기술, 특히 Razor에는 convention이 많다. Razor Page를 처음 접한다면 Microsoft 공식 문서를 한 번 훑어보는 것이 꽤 도움이 될 것 같다.

### Ref

- Tag Helpers in ASP.NET Core: <https://learn.microsoft.com/en-us/aspnet/core/mvc/views/tag-helpers/intro>
- Input Tag Helper: <https://learn.microsoft.com/en-us/aspnet/core/mvc/views/working-with-forms#the-input-tag-helper>
