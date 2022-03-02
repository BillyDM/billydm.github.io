+++
title = "Hello World Post"
date = 2022-03-01
[taxonomies]
categories = ["Misc"]
tags = ["Memes"]
+++

Yay, I finally got Zola working!

It took me forever to figure out how to deploy it on GitHub pages.

*~~I've also spent most of the day tweaking the CSS.~~*

> Update - I figured out how to get the `zola-deploy-action` GitHub action to work.

At first I thought the default [zola-deploy-action](https://github.com/shalzz/zola-deploy-action) GitHub action for deploying to GitHub Pages required creating one of those security nightmares known as GitHub Personal Access Tokens, as per the [official docs](https://www.getzola.org/documentation/deployment/github-pages/) at the time of this writing. There was no way I was doing that. So at first I tried just building it on my computer with `zola build -o docs`. This put the generated html files into the `/docs` folder, which I then just commit directly to the main branch.

But then after posting about my initial blog post on Discord, someone pointed out that you don't actually need to create the GitHub Personal Access Token to use the deploy script, and that the official docs were outdated. Fun.

Even when I did that, I found the default script example in the [zola-deploy-action](https://github.com/shalzz/zola-deploy-action) repo has it try to build into the `docs` directory, which then spits out an error message saying `/entrypoint.sh: line 63: cd: docs: No such file or directory`. Changing this to use the default root directory fixed this. Guess I need to file another issue with the documentation.

Another headache was figuring out that Zola treats themes in the `themes` folder as git submodules, so you can't actually customize the CSS there. The solution was to just omit putting the theme in the theme folder and instead including the contents directly into my repo. I don't want to rely on a submodule dependency for something as simple as a blog website.

Also, I was told I should add [RSS Autodiscovery](https://www.petefreitag.com/item/384.cfm) to my site so I did that.

All in all, Zola has some documentation issues that need addressing, but in the end I really like how my modified theme turned out. Being able to write blogposts in markdown and seeing the result live with `zola serve` is really cool.

A meme to celebrate: 🎉

![dank meme](/images/dank_meme.jpg)