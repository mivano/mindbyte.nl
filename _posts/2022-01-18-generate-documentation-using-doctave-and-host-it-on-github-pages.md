---
published: true
title: Generate documentation using Doctave and host it on GitHub Pages
tags:
  - GitHub
  - Pages
  - Actions
  - Documentation
---
Documentation as code is something I [blogged](https://mindbyte.nl/2021/11/08/use-mermaid-diagrams-with-images-inside-your-documentation-using-github.html) about before. In that specific case, it was about converting Mermaid diagrams to images so you could browse the documentation on GitHub in an easy way.

Still, it is a cumbersome process that includes Pull Requests and additional workflow. Also, browsing through the preview of markdown on the GitHub website is not as easy as it could be. You also miss a sensible navigation tree and a practical search option. It would be very cool if we can convert the markdown to a static site and host it somewhere. 

Using Jekyll, you can do a lot already, but there are many things to handle, like templates and layout. There is a simpler solution; [Doctave](https://www.doctave.com/). This SaaS allows you to host generated documentation on their platform. It is driven by an [open source tool](https://github.com/Doctave/doctave) which can be used to generate your own site and host it somewhere else. It is an opinionated static site generator with nice features like search, dark theme, navigation tree, and mermaid rendering. There is not a lot of customization, but that is fine for what it does.

So how to use it? It boils down to having a documentation folder like `/docs/` with markdown files. Make sure there is a file called `README.md` (mind the casing), which acts as the start page. In the root of your repository, you need a file `doctave.yml`, containing at least a title. This whole process is documented nicely on their [website](https://cli.doctave.com/tutorial). I also needed to add a folder `/docs/_include` containing an empty file called `.nojekyll` to prevent Jekyll from creating a site as well.

When you have the basics and Doctave installed, you can either `serve` or `build` the site. You can add navigation links by changing the `doctave.yml` file and add titles to the files by including frontmatter on top of the markdown. So still some customisation can be done.

But now, we want to host the generated site on GitHub Pages instead of Doctave. This is a bit more involved. So let's create a workflow file using GitHub Actions to build and publish your documentation website.

```yaml
name: Deploy documentation site

on:
  push:
    paths:
    - 'docs/**'
    - '.github/workflows/deploy-docs.yml' 
  workflow_dispatch:

jobs:
  build:
    name: Deploys
    runs-on: ubuntu-latest
    steps:
    - name: 'Checkout GitHub Action'
      uses: actions/checkout@main

    - name: 'Build and deploy'
      shell: bash
      run: |
        brew install doctave/doctave/doctave
        doctave build --release --allow-failed-checks
        
    - name: GitHub Pages
      if: github.ref == 'refs/heads/main'
      uses: crazy-max/ghaction-github-pages@v2.6.0
      with:
        build_dir: site/
      env:
        GITHUB_TOKEN: {{ "{{" }} secrets.GITHUB_TOKEN }}
```

This will checkout the code, install Doctave on the build agent, and build the documentation. Then it will deploy the documentation (placed in the `site` folder) to GitHub Pages. It needs your GitHub token to do that. Notice that I used the `--allow-failed-checks` flag to convert errors into warnings as I still have some links not working correctly. 

When deployed to GitHub Pages, you do need to configure your site under the settings. Pick the `gh-pages` branch and select either a private url or a public one. It will show you the site's url, which should now contain your generated documentation site. Every change you make to your docs is built, and when run on main, it is deployed as well.

For me, this is a great way to share documentation with, for example, other developers and still keep the docs close to your code. Furthermore, no extra steps are needed to convert Mermaid diagrams!