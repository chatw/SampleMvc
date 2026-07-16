# Configuring an Existing ASP.NET Core MVC Project to Support Razor (Blazor) Components

This guide walks through adding Blazor Server component support to an existing ASP.NET Core MVC project, and embedding an interactive component inside a Razor view using the `<component>` tag helper.

## 1. Register Blazor Server services

In `Program.cs`, add server-side Blazor services and map the Blazor SignalR hub:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllersWithViews();
builder.Services.AddServerSideBlazor();   // enables Blazor Server components

var app = builder.Build();

// ...existing middleware...

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.MapBlazorHub();   // exposes the /_blazor SignalR endpoint

app.Run();
```

## 2. Update `_Layout.cshtml`

Two additions are required in `Views/Shared/_Layout.cshtml`.

**a) Add `<base href="~/" />` to `<head>`.** Without this, the SignalR negotiate request resolves relative to the current page's URL instead of the site root, which breaks the circuit connection on any route other than `/`.

```html
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <base href="~/" />
    <title>@ViewData["Title"] - SampleMvc</title>
    ...
</head>
```

**b) Add the Blazor Server script**, right before `</body>`:

```html
<script src="_framework/blazor.server.js"></script>
</body>
```

## 3. Create a `Components` folder with `_Imports.razor`

Add a `Components` folder to the project, and inside it a `_Imports.razor` file:

```razor
@using Microsoft.AspNetCore.Components
@using Microsoft.AspNetCore.Components.Web
@using Microsoft.AspNetCore.Components.Web.Virtualization
@using Microsoft.AspNetCore.Components.Forms
```

This import is required for Blazor's event-binding directives (`@onclick`, `@onchange`, `@bind`, etc.) to compile correctly. Without `Microsoft.AspNetCore.Components.Web` in scope, these directives are silently emitted as literal HTML text instead of wired-up event handlers — the component renders, parameters bind, but click handlers do nothing.

## 4. Create the Razor component

Example: `Components/UserCard.razor`

```razor
<h3>UserCard</h3>

<p>@UserName</p>
<p>@Email</p>

<button type="button" @onclick="ShowDetails">Toggle Details</button>

@if (_showDetails)
{
    <p>Job: @Job</p>
    <p>Salary: @Salary.ToString("C")</p>
}

@code {
    [Parameter] public string UserName { get; set; } = string.Empty;
    [Parameter] public string Email { get; set; } = string.Empty;
    [Parameter] public string Job { get; set; } = string.Empty;
    [Parameter] public int Salary { get; set; } = 0;

    private bool _showDetails;

    private void ShowDetails()
    {
        _showDetails = !_showDetails;
    }
}
```

## 5. Embed the component in an MVC view

In any `.cshtml` view (e.g. `Views/Home/Index.cshtml`):

```cshtml
<component type="typeof(SampleMvc.Components.UserCard)"
           render-mode="Server"
           param-UserName="@("John Doe")"
           param-Email="@("john.doe@example.com")"
           param-Job="@("Software Engineer")"
           param-Salary="@(75000)" />
```

Notes on this syntax:

- `type` must be a `typeof()` expression — a plain string will not work.
- Parameters are passed individually as `param-{Name}` attributes, matched against `[Parameter]` properties by name (case-sensitive). There is no bulk `params="..."` syntax.
- `render-mode` options: `Static`, `Server`, `ServerPrerendered`. `Server` skips prerendering and connects the circuit immediately, which is simplest for debugging.

## 6. Verify

Run the project and open the page in a browser:

- View Page Source should show the component's rendered HTML with Blazor diffing markers (`<!--!-->`), and no literal `@onclick`/`@bind` text.
- DevTools → Network → filter "WS" should show a WebSocket connection to `/_blazor` with status `101 Switching Protocols`.
- Clicking the button should toggle the Job/Salary fields without a full page reload.

## Troubleshooting checklist

| Symptom | Likely cause |
|---|---|
| Button does nothing, no console errors | Missing `<base href="~/" />` — check WebSocket connection in Network tab |
| WebSocket connects fine but button still does nothing | Missing `@using Microsoft.AspNetCore.Components.Web` (directly or via `_Imports.razor`) |
| Raw `@onclick="Method"` text visible in View Source | Same as above — event directive not compiled |
| `CS0201` build error | Unrelated C# syntax issue — a bare expression (e.g. `==` instead of `=`) used as a statement |
| Component doesn't render at all | `type="typeof(...)"` used in a plain `.html` file instead of `.cshtml`, or namespace/class name mismatch |
