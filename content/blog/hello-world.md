+++
title = "Hello World Post"
date = 2022-03-01
[taxonomies]
categories = ["Misc"]
tags = ["Memes"]
+++

test

Yay, I finally got Zola working!

It took me forever to figure out how to deploy it on GitHub pages.

*~~I've also spent most of the day tweaking the CSS.~~*

By default Zola wants you to use their [zola-deploy-action](https://github.com/shalzz/zola-deploy-action) GitHub action for deploying to GitHub Pages, but it requires creating one of those security nightmares known as GitHub Personal Access Tokens. There is no way I'm doing that. The solution I found was to just build it on my computer with `zola build -o docs`. This puts the generated html files into the `/docs` folder, and I then just commit it directly to the main branch.

Another main headache was figuring out that Zola treats themes in the `themes` folder as git submodules, so you can't actually customize the CSS there. The solution was to just omit putting the theme in the theme folder and instead including the contents directly into my repo. I don't want to rely on a submodule dependency for something as simple as a blog website. Even if I didn't want to modify the CSS in the theme, GitHub's default `github-pages` action would fail with an error saying it couldn't find the `/themes/simplify` submodule. Ugh.

Anyway, using a native Rust tool is cool and all, but there definitely needs to be a cleaner way to deploy to GitHub pages. There *may* be a way that I'm just missing, but it's not obvious and my way is working now so I don't really care. I really like how my modified theme turned out so I'm sticking with Zola.

A meme to celebrate: 🎉

![dank meme](/images/dank_meme.jpg)