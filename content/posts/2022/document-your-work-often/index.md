+++
date = '2022-02-27'
draft = false
title = 'A lesson on documenting your work early, often and securely'
+++

If there's one thing that I would yell at myself when first getting started with software development or, for that matter, any project in general, it would be to document early, often and securely. Here's my story of creating a software company's wiki, backed by Azure AD, with Git as storage.

It's not a secret that the time investment required for documentation is either underestimated or just not allocated in the first place. Most enterprise scenarios won't ever account for documentation. Elaborations care little for the technical writing that will, or _should_ be required and stakeholders simply do not see the return on investment for documenting systems that, in ten years, might still be running but under a completely different set of eyes from those that originally envisioned the implementation. Open source projects generally fair better in this department, but even there poor devops can cause labourious tasks and lagging updates.

If code is your business I cannot stress enough how you should invest early into documenting your technical processes, projects and other jobs that you might be doing regularly but are thought-expensive.

Identify your minimum requirements of at least getting words onto a screen, securely.

## In an ideal world

**Absolutes:**
- Be backed by markdown files
- Be source controlled in a Git repository, alongside all of our other projects
- Allow for each pushing of new articles, whereby pushing to the repo would trigger a build pipeline
- Have low effort authentication and authorisation, maybe even be managed by an existing SSO option. We already make use of AzureAD almost everywhere

**Nice-to-haves:**
- Mermaid diagram integration 
- Syntax highlighting for codeblocks
- Built in markdown editing

## What we tried

For a number of years we had tried several solutions;

**A Word document in OneDrive**

While Microsoft's realtime collaboration tools are great and worked flawlessly for us - it's Word at the end of the day. It's OneDrive and just yuck. There's no real version control. It's hard to format, and we all know the problems of dragging images around. It's just not a flexible solution at scale. Code is horrific in Word.

**Confluence**

We were already in the Atlassian ecosystem. We used Jira, Bitbucket, and had the unfortunate luck of also using Hipchat for a small period of time. It made sense to try Confluence. Atlassian even has its own SSO capabilities across all products. One account, access to all products. Unfortunately Confluence is also incredibly convoluted. I have no doubt it functions well for large, critical, installations where there's a strict structure, with teams to maintain the many nuts and bolts within Confluence. We're small and don't have time to deal with the governance models Confluence _would_ want to impose on you.

**Azure DevOps Wiki**

When we migrated to Azure Devops and moved repos, CI/CD pipelines and issues to the same platform, we also decided to try out the built in Wiki functionality. Surprisingly it checks off many of the requirements. It being on Microsoft's Azure platform means authentication just works, the wiki is source controlled, it's immediately updated and can be edited in the browser. There is some proprietary(?) formatting to order the menu, but that's just one file. You might think, ok, so that's end of story? Well. Being on Azure Devops Wiki ties you into Azure DevOps. You can't just pick up your docs repo and put it elsehwere.

**GitHub Wiki**

Same problems Azure DevOps suffers of, and no integration with any other SSO providers.

## The solution

Any other solution compromised of expensive cloud-hosted solutions that wanted to store our documents themselves or self-hosting a maintenance heavy application. I wasn't in this to create more problems. I changed thought processes and went to look for static site from markdown generators.

Happily with the advent of ASP.NET Core, static file serving was baked right in and made easier than ever. What I had hoped to do was place whatever static site I would generate behind an ASP.NET Core-served website with AzureAD on the ASP.NET Core end, authenticating any request to any static file. That way, even if the underlying wiki technology changed, authentication (and, cruicially routing) would always remain the same.

I ended up using a new ASP.NET Core MVC app for this, for better control (pun not intended). `dotnet new mvc` or your equivalent options in your IDE of choice should be enough to get started.

Your next move is to add Azure AD authentication. This is documentated [here](https://docs.microsoft.com/en-us/azure/active-directory/develop/web-app-quickstart?pivots=devlang-aspnet-core).

The `Program.cs` file will end up looking like this:
```cs
using Microsoft.AspNetCore.Authentication.OpenIdConnect;
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Identity.Web;
using Microsoft.Identity.Web.UI;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

builder.Services
    .AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApp(builder.Configuration.GetSection("AzureAd"));

builder.Services.AddAuthorization();

builder.Services
    .AddRazorPages()
    .AddMicrosoftIdentityUI();

var app = builder.Build();

app.UseHttpsRedirection();

app.UseRouting();

app.UseAuthentication();
app.UseAuthorization();

var options = new DefaultFilesOptions();
options.DefaultFileNames.Clear();
options.DefaultFileNames.Add("index.html");
app.UseDefaultFiles(options);

// Must be after authorisation
app.UseStaticFiles(new StaticFileOptions
{
    OnPrepareResponse = ctx =>
    {
        if (ctx.Context.User is not { Identity: { IsAuthenticated: true } })
        {
            ctx.Context.Response.Redirect("/MicrosoftIdentity/Account/SignIn");
        }
    }
});

app.UseEndpoints(endpoints =>
{
    endpoints.MapRazorPages();
    endpoints.MapControllers();
});

app.Run();
```

So important shout outs here being the enabling of static files. ASP.NET Core will serve our static files, generated by our documentation platform of choice. If the user is not authenticated, via our AzureAD, we direct them back to the Identity login page for Azure AD. Something we don't need to worry about.

We have some additional logic in our Home Controller to serve the static index file, generated by MkDocs - you _could_ use ASP.NET Core's new minimal APIs to strip some of this out, but I prefer the control (again, not a pun) of controllers.

Our home controller looks like this:
```cs
using System.IO;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc;

namespace Controllers;

[Route("/")]
public class HomeController : Controller
{
    public IActionResult Index()
    {
        if (HttpContext.User is { Identity: { IsAuthenticated: true } })
        {
            var path = Path.Combine(Directory.GetCurrentDirectory(), "wwwroot", "index.html");
            return PhysicalFile(path, "text/html");
        }

        return Redirect("/MicrosoftIdentity/Account/SignIn");
    }
}
```

It's important to note that our documentation will eventually sit in the `wwwroot` folder of our ASP.NET Core site.

We're using [MkDocs](https://www.mkdocs.org/), more specifically [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) to generate a static site from markdown documentation. Adding a new repository, separated from our ASP.NET Core site, hosting the documentation, would allow for an agnostic host and client. I.e. docs can exist without ASP.NET Core and ASP.NET Core can exist without docs.

MkDocs is a Python-based framework. While you won't be dealing with Python code, you will be dealing with `pip`. It is the bare minimum required to get MkDocs to work. This approach, of hosting the docs site through a ASP.NET Core host, however, does allow you to switch out MkDocs.

Our MkDocs configuration looks like this:
```yml
site_name: Docs

theme:
  name: material
  custom_dir: overrides
  palette:
    - scheme: default
      toggle:
        icon: material/toggle-switch-off-outline 
        name: Switch to dark mode
      primary: green
      accent: teal
    - scheme: slate 
      toggle:
        icon: material/toggle-switch
        name: Switch to light mode
      primary: green
      accent: teal
  features:
    - navigation.tracking
    - navigation.tabs
    - navigation.sections
    - header.autohide

markdown_extensions:
  - pymdownx.highlight:
      anchor_linenums: true
  - pymdownx.superfences:
      custom_fences:
        - name: mermaid
          class: mermaid
          format: !!python/name:pymdownx.superfences.fence_code_format
  - pymdownx.tasklist:
      custom_checkbox: true
  - pymdownx.inlinehilite
  - pymdownx.snippets
  - def_list
  - attr_list
  - md_in_html
  - meta

plugins:
  - search

extra:
  generator: false
  ```
  
  Once you're happy, build the static docs site and place it in the `wwwroot` folder of ASP.NET Core and watch your docs be served, securely.