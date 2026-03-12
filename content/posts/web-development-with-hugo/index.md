+++
title = 'Publishing your own site with Hugo and GitHub pages'
date = 2026-03-11T15:23:00-05:00
draft = false
summary = 'How I developed this site with Hugo, and used GitHub to build, deploy, and host my static webpage.'
tags = ['hugo', 'web-development', 'github']
+++

I had a need to get my own site, and I was looking to get something simple
(meaning little HTML and CSS wrangling) up and running as cheaply as possible.
It was actually pretty easy to get a website up and running using a combination
of Hugo and Github Pages to do this. I divided this post in three parts that
correspond to the steps needed for you to publish your website as well.

## Part 1: Hugo

Hugo is a static site generator. That means you write content in Markdown, run
a command, and it generates a folder full of plain HTML files. There’s no need
to learn scripting languages or frameworks, as all you need to do is to install
the Hugo binary, run `hugo new site mysite`, and it performs the work for you.

Rather than using one of the many pre‑made themes, I decided to make my own so
I could have more control of the colors and font. A theme in Hugo is mostly
just a handful of template files and some configuration and so it is not hard
to follow to the untrained eye. I started by generating a default theme with 
the `hugo new theme mytheme` and then modified the files it generated.

While I was working on the design I used `hugo server` to spin up a local web
server. It watches your files and reloads the page whenever you save, which is
handy when you’re tweaking CSS or trying to figure out why your menu floated to
the wrong side.

A few other tips for newcomers:

1. Put your Markdown posts in `content/posts/` and separate them via folders.
   That way, content such as pictures can coexist with the posts that reference
   them.
2. The front‑matter (that little TOML or YAML block at the top) can have a lot
   of knobs about the post that can be tweaked depending on how you want it to
   look like, such as setting a [custom summary](https://gohugo.io/content-management/summaries/) for the post.
3. When you’re ready to build for real, run `hugo` and look in the `public/`
   directory; that’s what GitHub Pages will serve.

Once the theme looked the way I wanted and my sample posts rendered nicely, I
was ready to hand the generated site off to GitHub.

## Part 2: GitHub configuration

GitHub Pages can serve static sites straight from a repository, and you can get
it to build the site for you with a GitHub Actions workflow so you don’t have to
run `hugo` locally every time.

Here’s the basic flow I followed:

1. Create a new repository on GitHub (I named mine after [the website](https://github.com/sibucan/luisramirezrivera-dev)). Clone it
   locally and copy your Hugo site into it.
2. Commit everything and push to your main branch.
3. Add a workflow file under `.github/workflows/build.yml`. The YAML runs on
   every push, checks out the repo, installs Hugo, runs `hugo`, and then
   deploys the contents of `public/` on the main branch.

I used the workflow that the [Hugo documentation recommends](https://gohugo.io/host-and-deploy/host-on-github-pages),
with a couple of changes (your mileage may vary):

```yaml
name: Build and deploy
on:
  push:
    branches:
      - main
  workflow_dispatch:
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v6
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Setup Go
        uses: actions/setup-go@v6
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: false
      - name: Setup Node.js
        uses: actions/setup-node@v6
        with:
          node-version: ${{ env.NODE_VERSION }}
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5
      - name: Create directory for user-specific executable files
        run: |
          mkdir -p "${HOME}/.local"
      - name: Install Hugo
        run: |
          curl -sLJO "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          mkdir "${HOME}/.local/hugo"
          tar -C "${HOME}/.local/hugo" -xf "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          rm "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          echo "${HOME}/.local/hugo" >> "${GITHUB_PATH}"
      - name: Verify installations
        run: |
          echo "Go: $(go version)"
          echo "Hugo: $(hugo version)"
          echo "Node.js: $(node --version)"
      - name: Install Node.js dependencies
        run: |
          [[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true
      - name: Configure Git
        run: |
          git config core.quotepath false
      - name: Build the site
        run: |
          hugo build \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/" \
            --cacheDir "${{ runner.temp }}/hugo_cache"
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

Once that file is committed and pushed, GitHub Actions takes over. Every time I
push new content or change the theme, the workflow runs, and a few minutes later
the latest version of the site is live at `https://<username>.github.io/<repo>`.
You can also configure Pages to serve from the `gh-pages` branch directly if
you prefer to build locally; I just like the automation.

If you only want to host the site at the root of a user/organization account
(e.g. `username.github.io`), you can push the `public/` output to the `main`
branch instead and point GitHub Pages at that branch in the repository settings.

## Part 3: DNS

Eventually, I wanted to use my own domain instead of the GitHub URL. That
involves two small things:

1. **Tell GitHub what domain:** either put a `CNAME` file in the top of
   `public/` containing your domain (e.g. `example.com`), or set it in the
   [Pages settings](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/verifying-your-custom-domain-for-github-pages) on GitHub.
2. **Update your domain’s DNS records:** at the registrar or DNS provider for
   `example.com`, create the records GitHub asks for. For an apex/root domain you
   add four `A` records pointing to GitHub’s IPs; for a `www` subdomain you can
   use a `CNAME` to `username.github.io`.

After a few minutes for DNS to propagate, visiting `example.com` loads the same
pages that GitHub was serving at the default address. If you forget the CNAME
file, GitHub will complain on the settings page; if the DNS isn’t set up
typically you’ll just see the “Page not found” message when trying to hit your
custom URL.

And that’s the whole story: Hugo builds the site, GitHub Pages hosts it, and a
couple of DNS entries make it live on your own domain. It takes a little setup
the first time, but once it’s running all you do is write Markdown, push to
GitHub, and your blog updates itself. Welcome to the world of static site
generators! 
