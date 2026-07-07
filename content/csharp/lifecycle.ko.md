+++
title="ASP.NET Core Razor component lifecycle"
date=2025-06-09

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++


[ComponentBase methods](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.components.componentbase)

Blazor component lifecycle은 대략 다음 순서로 진행된다.

1. Parameter가 설정된다.
2. 초기화 로직이 실행된다.
3. parameter 값이 처리된다.
4. component가 render된다.
5. DOM 업데이트 이후 after-render callback이 실행된다.

`OnAfterRender`와 `OnAfterRenderAsync`는 prerendering/static SSR 중에는 호출되지 않는다. 아직 live browser DOM이 없기 때문이다.

### SetParametersAsync
`SetParametersAsync`는 component가 parent나 route로부터 parameter를 받을 때 호출된다. 이 method를 override한다면, 기본 parameter assignment 동작을 의도적으로 대체하는 경우가 아니라면 `base.SetParametersAsync(parameters)`를 호출해야 한다.

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
component instance마다 한 번만 실행되어야 하는 초기화 작업에는 `OnInitialized`/`OnInitializedAsync`를 사용한다.

```cs
protected override async Task OnInitializedAsync()
{
    await base.OnInitializedAsync();
}
```

### OnParametersSetAsync
현재 parameter 값에 따라 수행해야 하는 작업은 `OnParametersSet`/`OnParametersSetAsync`에 둔다. parent render 과정에서 새 parameter 값이 전달될 수 있으므로 이 method들은 여러 번 실행될 수 있다.

```cs
protected override async Task OnParametersSetAsync()
{
    await LoadDataAsync();

    await base.OnParametersSetAsync();
}
```

### OnAfterRenderAsync
render된 DOM element가 필요한 작업에는 `OnAfterRender`/`OnAfterRenderAsync`를 사용한다. 예를 들어 `ElementReference`를 이용한 JS interop이 여기에 해당한다. `firstRender`는 해당 component instance가 처음 render될 때만 `true`다. `OnAfterRenderAsync`가 반환한 `Task`가 완료되어도 자동으로 다시 render가 발생하지는 않는다.

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
