# Tailwind CSS 4.1 + Tailwind Plus (with esbuild) in ASP.NET Core MVC

Tailwind CLI builds CSS → `wwwroot/css/site.css`
esbuild bundles JS (incl. Tailwind Plus) → `wwwroot/js/site.js`
Static files served via `UseStaticFiles()`. Works with `dotnet watch` + CI/CD.

---

## Prerequisites

* .NET 9 SDK
* Node.js + Yarn (or npm if you must)
* A new ASP.NET Core MVC project

---

## 1) Create the project & remove the old defaults

```bash
dotnet new mvc -n MyProject
cd MyProject
```

ASP.NET drops Bootstrap/jQuery into `wwwroot/lib`. You don’t need them:

```bash
rm -r wwwroot/lib
```

*(Delete it already — you’re not going to need it.)*

If your `.csproj` contains a huge `<ItemGroup>` removing bootstrap/jquery files from `wwwroot/lib`, delete that whole block too. It only exists because the template added those files.

---

## 2) Initialize Yarn & install deps (and fix `package.json`)

```bash
yarn init -y
yarn add -D tailwindcss@^4.1.13 @tailwindcss/cli@^4.1.13 esbuild
yarn add @tailwindplus/elements
```

> `yarn init -y` adds `"main": "index.js"` by default. You’re not publishing a Node library — **remove that line**.

**`package.json` (minimal):**

```json
{
  "name": "myproject",
  "version": "1.0.0",
  "license": "MIT",
  "scripts": {
    "dev:css": "tailwindcss -i ./Styles/site.css -o ./wwwroot/css/site.css --watch",
    "build:css": "tailwindcss -i ./Styles/site.css -o ./wwwroot/css/site.css --minify",
    "dev:js": "esbuild Scripts/site.js --bundle --format=esm --target=es2018 --outfile=wwwroot/js/site.js --watch",
    "build:js": "esbuild Scripts/site.js --bundle --minify --format=esm --target=es2018 --outfile=wwwroot/js/site.js"
  },
  "devDependencies": {
    "esbuild": "^0.24.0",
    "tailwindcss": "^4.1.13",
    "@tailwindcss/cli": "^4.1.13"
  },
  "dependencies": {
    "@tailwindplus/elements": "^1.0.0"
  }
}
```

> Why separate CSS & JS builds? Tailwind CLI builds CSS. esbuild bundles JS. That’s the setup.

---

## 3) Inputs

**`Styles/site.css`** (replace the default content)

```css
@import "tailwindcss";
@source "./Views/**/*.cshtml";
@source "./Pages/**/*.cshtml";
```

**`Scripts/site.js`** (replace the default content)

```js
import '@tailwindplus/elements';

console.log('Tailwind Plus ready');
```

*(If you don’t see the log in the console, we gotta problem.)*

---

## 4) Layout imports

Open **`Views/Shared/_Layout.cshtml`**. The template already references `site.css` and `site.js`.
Make sure the JS is loaded as a module so the esbuild ESM bundle runs:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>@ViewData["Title"] - MyProject</title>

  <!-- Tailwind CSS (compiled to wwwroot/css/site.css) -->
  <link rel="stylesheet" href="~/css/site.css" asp-append-version="true" />
</head>
<body class="min-h-screen bg-gray-50 text-gray-900">

  <!-- Simple Tailwind nav -->
  <nav class="bg-gray-800 p-4 text-white flex justify-between">
    <a href="/" class="font-bold">MyProject</a>
    <div class="space-x-4">
      <a href="/" class="hover:text-blue-300">Home</a>
      <a href="/Privacy" class="hover:text-blue-300">Privacy</a>
    </div>
  </nav>

  <main class="container mx-auto px-6 py-10">
    @RenderBody()
  </main>

  @await RenderSectionAsync("Scripts", required: false)

  <!-- JS bundle -->
  <script type="module" src="~/js/site.js" asp-append-version="true"></script>
</body>
</html>
```

*(\`<div class="container"><div class="row"><div class="col">… yeah … no.)*

---

## 5) Serve static files (Program.cs)

Enable static files to serve `wwwroot/**`:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllersWithViews();

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles(); // serves wwwroot/css, wwwroot/js, etc.

app.UseRouting();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```

> If your template had `app.MapStaticAssets();`, keep `UseStaticFiles()` anyway — that’s what serves your own `wwwroot` content.

---

## 6) Dev workflow

Run watchers in parallel:

```bash
dotnet watch       # backend hot reload
yarn dev:css       # Tailwind CLI (CSS)
yarn dev:js        # esbuild (JS)
```

*(Yes, that’s three terminals. Come on dude — you’re not stuck on 1024×768 anymore.)*

---

## 7) Production build

When you’re ready to deploy… just don’t do it on a Friday.

```bash
yarn build:css
yarn build:js
dotnet publish -c Release -o ./out
```

Your compiled CSS/JS will be in `./out/wwwroot`.

---

## 8) Hook frontend into publish (CI/CD)

Make .NET run the frontend build during publish.

**`MyProject.csproj`** — single backslashes (or forward slashes) are fine:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <Target Name="BuildFrontend" BeforeTargets="Publish">
    <Exec Command="yarn install --frozen-lockfile" />
    <Exec Command="yarn build:css" />
    <Exec Command="yarn build:js" />
  </Target>

  <ItemGroup>
    <Folder Include="wwwroot\css\" />
    <Folder Include="wwwroot\js\" />
  </ItemGroup>
</Project>
```

---

## 9) .gitignore essentials

Keep the repo clean:

```
# .NET
bin/
obj/

# Node
node_modules/

# Build outputs
wwwroot/css/*
wwwroot/js/*
!wwwroot/css/.gitkeep
!wwwroot/js/.gitkeep

# logs
*.log
```

---

## 10) Quick sanity checklist

* `~/css/site.css` loads (200 OK in Network tab).
* `~/js/site.js` loads and the console prints **“Tailwind Plus ready”**.
* Edit a Tailwind class in a Razor view → `yarn dev:css` rebuilds.
* Edit `Scripts/site.js` → `yarn dev:js` rebuilds.

Fix anything that fails before adding features.

---

## FAQ

**Why not bundle everything with one tool?**
Tailwind 4.1’s CLI is excellent at CSS and doesn’t need PostCSS/Webpack/etc. esbuild is a fast, tiny JS bundler. Let each do its job.
