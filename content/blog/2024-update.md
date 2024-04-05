+++
title = "2024 Update"
date = 2024-04-05
[taxonomies]
categories = ["Misc"]
tags = ["Misc"]
+++

---

# Preface

Alright, I'm back with something to show you! I'm sorry it's been a while. I feel like I should explain what's been going on.

---

# What I was doing the past year?

The past year has been kind of rough for me. I haven't gotten quite as much stuff done as I would have liked to. But I suppose mental and physical health come first. It's taken me much longer to regain my footing than I thought it would.

In a [previous blogpost](../clarifying-some-things) I said I was teaming up with someone who was creating a DAW with a GUI in Flutter that had a sort of similar timeline workflow vision. In a different universe where I was a different kind of person, this would have been an great opportunity.

But as it turns out, it's taken me a long journey to realize I truly stay motivated best on large projects when working alone towards a highly specific vision. As a DAW user myself, GUI is very important to me. If I'm going to be doing this as my career, I want the GUI to be as good as I dream of having it. Otherwise I just don't find the motivation to keep at a multiple-year long project.

That's not to say I don't want other people working on Meadowlark. Just that I figured it's best to only delegate tasks to other people that don't involve core architecture stuff. I do still very much want help for other tasks like DSP, themes, samples, presets, and other various add-ons. I'm by no means an expert at DSP or sound design.

I also ultimately decided against Flutter because I want a library that can also be used to make audio plugins that can be loaded into any DAW, not just Meadowlark. Flutter just doesn't work for plugins *(see my previous blogpost [DAW Frontend Development Struggles](../daw-frontend-development-struggles) for why)*. While I could use something like [Vizia](https://github.com/vizia/vizia), [Iced](https://github.com/iced-rs/iced), or [egui](https://github.com/emilk/egui) for future audio plugins, I would love it if I can have the same framework for both Meadowlark and my plugins. So since I need a solution for Meadowlark anyway, I finally just said screw it, I'm going to create my own GUI library and tailor the features I need and want.

I know making a GUI library is very difficult and time consuming. But I'm not entirely new to this. I've had attempts at a simple GUI library in Rust in the past, and I have some past experience with graphics programming. In a future blogpost, I'll explain how I've devised a way to keep the GUI library implementation as simple as possible while still achieving high performance and flexibility.

Unlike other Rust GUI libraries, mine does not aim for "elegant and hard to misuse". It does not aim to be a "general purpose" GUI library with lots of features. It only contains the features I need. It's not even declarative (although in theory you could write a declarative wrapper around it like what [Relm4](https://github.com/Relm4/Relm4) does).

What my GUI library does give you is lots of manual control over how your widgets are structured, styled, and laid out. It's sort-of a fusion between some of [Iced](https://github.com/iced-rs/iced)'s concepts and traditional retained-mode GUI library concepts (although without the whole multiple-inheritance object-oriented thing). It uses a sort of "I know where things should go" and "no hidden control flow" philosophy.

---

# Coming Up

In my [next blogpost](../rootvg), I'll share a piece of what I accomplished. It's a hardware-accelerated 2d vector graphics rendering library called RootVG!

And in case you're wondering, no I haven't spent this entire time on just RootVG. I'm also getting closer to completing the GUI library as well. Stay tuned for that as well!
