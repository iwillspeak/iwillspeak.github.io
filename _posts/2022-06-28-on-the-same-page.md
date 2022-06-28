---
title: On the Same Page
layout: post
published: true
---

At the end of 2021 GitHub pages deployments [moved to using GitHub actions][announcement]. When a site is deployed you can see the jobs running to generate your site, package it up ready to be deployed, and push it to the GitHub Pages service. Lets try using these actions in a custom GitHub Actions workflow to build and deploy a site without using Jekyll.

There are two main steps in the deployment of a site to GitHub pages:

 * An artifact is generated containing the contents of the site to be deployed. This is carried out by the [`actions/upload-pages-artifact`][upload-pages-action]
 * The artifact is published to the GitHub Pages API. This is the job of [`actions/deploy-pages`][deploy-action].

First up lets start by creating a skeleton workflow to build our site:

```yaml
name: Build Docs

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Install Docket
      run: |
        mkdir -p _site/
        echo "<h1>Hello Custom Pages World" > \
           _site/index.html
```
 
This creates a simple job which builds what can generously be called a website into the `_site/` directory. With our custom site build out of the way the next step is to package it as an artifact. We can do that with the following step added to the `build` job:

```yaml
    - name: Push artifact
      uses: actions/upload-pages-artifact@v0
```

This should produce an artifact `github-pages` as part of the build. Before we can deploy this to Pages there's a little setup we need to do in the UI. Head to the **Settings** tab and enable pages for some placeholder branch such as `gh-pages-stub`. This makes your site available to deploy to. To allow deployments from _other_ branches go to the **Environments** tab and _enable_ branch protections [as outlined in this GitHub issue][troubleshooting]. With those steps out of the way we can move on to adding our deployment job.

To deploy to Pages we need a few permissions. Add the following to the root of the Actions yaml:

```yaml
permissions:
  contents: read
  pages: write
  id-token: write
```

This makes the required authentication tokens available to the pipeline so that the deployment action can run. All that now remains is to add a deployment job to our pipeline:

<!-- {% raw %} -->

```yaml
  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v1
```

The key parts here are the `needs: build` to force it to wait for our build step so that the artifact is available, and the `url: ${{ steps.deployment.outputs.page_url }}` in the environment definition that ties the deployment to the correct Pages site.

<!-- {% endraw %} -->

With all that in place we now have [a working Actions workflow][workflow] building a simple site and deploying to Pages. Check out the [full code for this post][repo] to see it in action, or [gaze upon the output in wonder][output]. 

 [announcement]: https://github.blog/changelog/2021-12-16-github-pages-using-github-actions-for-builds-and-deployments-for-public-repositories/
 [troubleshooting]: https://github.com/actions/deploy-pages/issues/20#issuecomment-1068207408
 [deploy-action]: https://github.com/actions/deploy-pages
 [upload-pages-action]: https://github.com/actions/upload-pages-artifact
 [workflow]: https://github.com/iwillspeak/on-the-same-page/blob/main/.github/workflows/pages.yml
 [repo]: https://github.com/iwillspeak/on-the-same-page
 [output]: http://willspeak.me/on-the-same-page/