+++
title="ASP.NET Core Razor component lifecycle"
date=2025-06-09

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++


[ComponentBase methods](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.components.componentbase)

Blazor component lifecycle is roughly:

1. Parameters are set.
2. Initialization runs.
3. Parameters are processed.
4. The component renders.
5. After-render callbacks run after the DOM is updated.

`OnAfterRender` and `OnAfterRenderAsync` are not called during prerendering/static SSR because there is no live browser DOM yet.

### SetParametersAsync
`SetParametersAsync` is called when the component receives parameters from its parent or from the route. If you override it, call `base.SetParametersAsync(parameters)` unless you are intentionally replacing the default parameter assignment behavior.

```cs
public override async Task SetParametersAsync(ParameterView parameters)
{
    if (parameters.TryGetValue<string>(nameof(Param), out var value))
    {
        if (value is null)
        {
            message = "The value of 'Param' is null.";
        }
        else
        {
            message = $"The value of 'Param' is {value}.";
        }
    }

    await base.SetParametersAsync(parameters);
}
```

### OnInitializedAsync
Use `OnInitialized`/`OnInitializedAsync` for initialization that should run once for the component instance.

```cs
protected override async Task OnInitializedAsync()
{
    await base.OnInitializedAsync();
}
```

### OnParametersSetAsync
Use `OnParametersSet`/`OnParametersSetAsync` when work depends on current parameter values. These methods can run more than once because parent rendering can supply new parameter values.

```cs
protected override async Task OnParametersSetAsync()
{
    await LoadDataAsync();

    await base.OnParametersSetAsync();
}
```

### OnAfterRenderAsync
Use `OnAfterRender`/`OnAfterRenderAsync` for work that needs rendered DOM elements, such as JS interop with `ElementReference`. `firstRender` is true only the first time that component instance renders. Completing the returned `Task` from `OnAfterRenderAsync` does not automatically trigger another render.

```cs
protected override async Task OnAfterRenderAsync(bool firstRender)
{
    if (firstRender)
    {
        await InitializeDomAsync();
    }

    await base.OnAfterRenderAsync(firstRender);
}
```

### Ref

- Razor component lifecycle: <https://learn.microsoft.com/en-us/aspnet/core/blazor/components/lifecycle>
- ComponentBase API: <https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.components.componentbase>
