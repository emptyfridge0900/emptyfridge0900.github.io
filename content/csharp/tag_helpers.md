+++
title="Tag Helpers"
date=2025-06-16

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

It had been a long time since I used Razor Pages, and I tried them again after several years.
I was building a form that receives user input, but `Input.Email` was not binding, so I struggled with it for a while.

I used `[BindProperty]`, and the `.cshtml` and `.cshtml.cs` filenames matched. So why did clicking the submit button return 404?

```html
<div class="form-group">
    <label asp-for="Input.Email">email</label>
    <input asp-for="Input.Email" class="form-control" />
    <span asp-validation-for="Input.Email" class="text-danger"></span>
</div>
```

After putting the example code into the IDE, I realized that `<input asp-for="...">` is not just a normal HTML input. It is markup targeted by `InputTagHelper`. Tag Helpers are written in C# and attach to Razor markup based on element names or attribute names to help generate HTML.

When I went back to my code, my input was just a plain HTML element.
I looked through the example again and found that `_ViewImports.cshtml` contained this declaration:

```cshtml
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

Only after adding `@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers` to `_ViewImports.cshtml` did `asp-for="Input.Email"` get processed by `InputTagHelper`, and email binding started working.

Microsoft technologies, especially Razor, have many conventions. If someone is new to Razor Pages, reading the official Microsoft documentation once is probably worth it.

### Ref

- Tag Helpers in ASP.NET Core: <https://learn.microsoft.com/en-us/aspnet/core/mvc/views/tag-helpers/intro>
- Input Tag Helper: <https://learn.microsoft.com/en-us/aspnet/core/mvc/views/working-with-forms#the-input-tag-helper>
