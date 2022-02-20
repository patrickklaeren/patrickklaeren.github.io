---
layout: post
title: Document your work early, often and securely
permalink: /2022/azure-ad-team-wiki.html
---

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

...
