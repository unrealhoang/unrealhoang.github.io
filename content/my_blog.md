+++
title = "Welcome to my blog"
date = 2018-06-20
tags = ["Hello World", "ci", "static page"]
+++

This is `/dev/random`.

Some random bits from a biased quantum-based random generator, or so-called my brain.

For the very first post, I'm gonna write on how I set this blog up, so that if you
are interested, you can try it my way.
<!-- more -->

# Blog Engine

As a Rust lover, I choose [Zola](https://www.getzola.org/) as my driver, it's
a nice **written in Rust** static page generator. Just like Golang, Rust can compile
the whole project into nice single binary that can be downloaded and run everywhere.
That is also the way I set CI to deploy this blog, but more on that later.

# Hosting

[Github Page](https://pages.github.com) is home for us developer, and it's also free. Why not?

# Deploy

To deploy to Github Page, I have to push the final HTMLs to `master` branch,
so I keep the sources (blogs content, templates, configurations) in a `source` branch.
After done writing blog posts and configuring, We build using:

```bash
gutenberg build
```

Now Gutenberg will generate the nice HTMLs, minified CSSs all into `public` folder, we need to push
that folder as the content of `master` branch, using:

```bash
git subtree push --prefix public origin master
```

# CI

Everytime I change something I need to run the Deployment steps again. 
Let's be lazy and ask someone else to do that for us, introducing 
[TravisCI](https://travis-ci.com).  
Register your Github project with TravisCI and we are ready to roll, be careful not to register
your project with both `https://travis-ci.com` and `https://travis-ci.org` because that was
what I did, and it caused some confusions for me.

After that, put `.travis.yml` file into your project with following content:

```yaml
language: minimal

branches:
  only:
  - source

before_script:
  # Download and unzip the zola executable
  # Replace the version numbers in the URL by the version you want to use
  - curl -s -L https://github.com/getzola/zola/releases/download/v0.11.0/zola-v0.11.0-x86_64-unknown-linux-gnu.tar.gz | sudo tar xvzf - -C /usr/local/bin

script:
  - zola build

deploy:
  provider: pages
  skip-cleanup: true
  github-token: $GITHUB_TOKEN  # Set in the settings page of your repository, as a secure variable
  keep-history: true
  local-dir: public
  target-branch: master
  on:
    branch: source
```

Push it on and wait for the result:

```bash
git push origin source
```

That's all of it. Hope to see you soon.
