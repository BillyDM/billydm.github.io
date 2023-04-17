+++
title = "Why I'm Taking a Break from Meadowlark"
date = 2023-04-14
[taxonomies]
categories = ["Misc"]
tags = []
+++

---

# Preface

I want to explain in this blog why I'm putting Meadowlark and all its sub-projects in a temporary hiatus. And maybe this article can be insightful for anyone else wanting to create a large-scale open source project.

In short, I've been feeling overwhelmed, depressed, and anxious lately. You can probably see my frustrations start to pile up in my previous blog [DAW Frontend Development Struggles](../daw-frontend-development-struggles), but the issues go deeper than that.

I think the core issue is that I started to break down once I realize just how much of a social job it is to be the manager of a large project. I realized that if I wanted to create the original vision I had for Meadowlark, I would probably need to have more developers working on large chunks of the project. I'm really just not a people-oriented person, and it's causing me to question if this is the kind of career I want in life.

Also there has obviously been a *lot* of excitement and interest around this project, but the publicity is starting to get to me.

---

# Some Background

For context, I am an extremely introverted person, and I'm also autistic.

Growing up I've always had a passion for mathematics and building things. I've built a lot of stuff out of K'Nex as a kid, and later my Lego Mindstorms kit sparked my interest in programming. As a teen I taught myself how to make Flash games and programming in general.

My interest in music production and music software started when our parents decided to get each of us kids a musical instrument one summer. Being into electronic music (*queue early 2000s techno*), I got one of those cheap Casio keyboards. The salesman convinced us to throw in a boxed copy of FL Studio with it. I ended up playing around with FL Studio far more than I did with the keyboard.

I hated school growing up, although I still managed to get good grades. After highschool I went into college for computer engineering, *but* I quickly had a mental breakdown and dropped out after just two weeks. Luckily my parents are very supportive and they've been helping out.

Ever since then I've been struggling to figure out what I want to do with my life. After college I've had a couple of temporary jobs here and there, but nothing I was happy with. I also have bad driving anxiety which doesn't help with job security here in the Midwest USA. I've also tried various things like game development (especially mobile game development) and music, but that venture didn't turn out like I hoped. For a while I took a class on IT and even landed an IT job with a startup, but then my health went really south and put me out of commission for a long time (I"m better now though).

Also I'm not interested nor do I feel the desire to get married and/or start a family of my own. So I feel like I need something else to bring fulfillment in my life. I'm a creative person, so naturally I want to create something.

A few years ago I got the idea to combine my passion for music software and programming and learn audio programming. At that time I also learned about the Rust programming language, which eventually led me to find the Rust Audio Discord server. From there I realized that it could actually be possible to create a DAW in Rust. And not just any DAW, one that I actually wanted to use myself.

I knew going in that it was going to take a *long* time to develop a DAW, but I wanted a career I was passionate about.

But now I've been working on Meadowlark for over 2 years now and starting to question things.

---

# Scope

I'm worried that I may have bitten off more than I can chew with the features I touted in [Meadowlark's design doc](https://github.com/MeadowlarkDAW/Meadowlark/blob/main/DESIGN_DOC.md). Even the MVP features alone are turning out to be quite difficult.

Currently I'm stuck between two options:

- Do I try and create a team of developers for this project (even going as far as possibly hiring developers)?
- Or do I limit the scope of Meadowlark and keep working on the majority of it alone?

I'm really not sure what I want to do. Obviously managing a team gives me a lot of anxiety thinking about it, but at the same time I want Meadowlark to be something that I actually want to use myself.

I've also come to realize that I tend to focus better and feel happier when working alone. That may sound odd to a lot of you, but it's how my brain works. But at the same time I also know that working alone isn't always an option. I'm still struggling to figure out what kind of career I want in life.

Someone in my Discord suggested perhaps creating something simpler like a tracker before creating a full-blown DAW? It's not a bad idea, and maybe I'll do that. But again I'm still trying to decide if I want to have a team of people work on it or if I want to do it mostly by myself.

---

# Architectural Problems

My original plan for Meadowlark was to just wing it and develop nearly everything myself for the MVP (minimum viable product) release for Meadowlark, and then go from there. This however didn't turn out as well as I hoped. The code began to get really messy and hard to follow with a lot of interconnected parts.

I realize that I need to come up with a better plan for the overall architecture of Meadowlark. Not just for my own sanity, but also for other people who want to chip into the project. Replanning and reworking everything is going to take some time, which is partly why I put this project in hiatus.

If I do decide to have a team of developers, I need to spend time creating an actual design document this time around. Not just explaining the goals of the project, but actual technical plans on how things will work and fit together.

---

# Managing Repositories

Because of how easy Rust makes it to make modular code, wherever it made sense I split the code into reusable crates such as [dropseed](https://github.com/MeadowlarkDAW/dropseed), [rainout](https://github.com/MeadowlarkDAW/rainout), [creek](https://github.com/MeadowlarkDAW/creek), [pcm-loader](https://github.com/MeadowlarkDAW/pcm-loader), [meadowlark-plugins](https://github.com/MeadowlarkDAW/meadowlark-plugins), [meadowlark-factory-library](https://github.com/BillyDM/meadowlark-factory-library), [audio-waveform-mipmap](https://github.com/MeadowlarkDAW/audio-waveform-mipmap), [Meadowlark's website](https://github.com/MeadowlarkDAW/MeadowlarkDAW.github.io)  etc. There are also several forks such as [audio-graph](https://github.com/MeadowlarkDAW/audio-graph), [clack](https://github.com/MeadowlarkDAW/clack), [samplerate-rs](https://github.com/MeadowlarkDAW/samplerate-rs), and [rust-libsamplerate](https://github.com/MeadowlarkDAW/rust-libsamplerate).

I also have several crates outside of the Meadowlark project such as [Awesome-Audio-DSP](https://github.com/BillyDM/Awesome-Audio-DSP), [iced_audio](https://github.com/iced-rs/iced_audio) [iced_baseview](https://github.com/BillyDM/iced_baseview), [egui-baseview](https://github.com/BillyDM/egui-baseview), [imgui-baseview](https://github.com/BillyDM/imgui-baseview), [bit-mask-ring-buf](https://github.com/BillyDM/bit_mask_ring_buf), [slice-ring-buf](https://github.com/BillyDM/slice_ring_buf).

But now I find that managing so many repositories is a headache. Every time I see an issue or a PR, I get anxious about having to now spend my time responding to it, which also seems to disrupt my flow for the rest of the day.

---

# Rust vs C++

A part of me kind of regrets choosing Rust when it comes to those sub-projects I mentioned above. If I used C++, I could have just used JUCE and/or other well-established C++ libraries for some of those things and saved myself a lot of time.

Another thing I kind of regret is needing to translate existing open source plugin DSP from C++ to Rust. It's yet another time sink.

But don't get me wrong, I still think Rust is the future. Especially when you have a team of developers, Rust's strictness and safety guarantees are invaluable in making sure all that code works together.

It's just that if I do decide to keep working on my own, I'm finding it harder and harder to justify the extra work needed just to be able to use Rust. I'm still deciding on it, and I also don't want to let my other fellow Rust audio programmers down.

---

# GUI Problems

I didn't anticipate just how hard of a technical problem the GUI would be. I went into more detail in my recent blog [DAW Frontend Development Struggles](../daw-frontend-development-struggles). There are some things I might have gotten wrong in that article such as the importance of damage tracking for rendering (I'm still on the fence on whether it's important or not), but the GUI is still a problem I'm not sure how to overcome.

Right now I'm looking into a couple of new contenders for a GUI library such as [Flutter](https://flutter.dev/) and [Dioxus](https://dioxuslabs.com/). [GTK4](https://www.gtk.org/) is also still a strong contender. And if I decide to go with C++, JUCE is an obvious choice.

Something to keep in mind is that commercial DAWs like Bitwig and Ableton Live have decided to develop their own in-house GUI solutions. Of course they were started at a time when the general-purpose GUI library landscape was much different, but the point still stands that they calculated that it was worth creating an in-house solution over working around a 3rd-party general-purpose GUI library. Of course these companies also have the luxury of more developers, so I'm not sure if this is a good idea for Meadowlark even if I do decide to hire a team of developers myself.

Another thing I could do is not worry about performance or looks at all for MVP and use something like [egui](https://github.com/emilk/egui). But I'll still need to switch to a better GUI library at some point, and switching GUI libraries after the fact isn't trivial, especially if you have a lot of custom widgets.

---

# Publicity

I'm not entirely sure why, but I've started feeling anxious about all the publicity I'm getting.

I've gotten a *lot* of people joining my Discord server excited about the project. A lot of them are even eager to help contribute code. While this sounds like a great thing, in reality I've been really struggling to figure out what these volunteer developers can actually do to help. It also doesn't help that there isn't much documentation and that the architecture is currently a mess and needs reworking.

But even if I did have a sound architecture with lots of documentation, I'm still wary about relying on volunteer effort. For one it makes me anxious thinking about having to create, manage, and coordinate all of those tasks. And two, (I'm not trying to sound condescending to anyway wanting to help), but I can hardly think of any "good first issue" tasks in something as complex as a DAW (at least for the initial bulk development).

Most of the eager volunteers I get say they are new to Rust and/or they want to help as a way to learn how DSP or audio software works. Again I don't mean to sound condescending (I was in the same boat myself at one point), but there's definitely some prerequisite knowledge required in making a complex DAW codebase. I would have to spend my time getting every volunteer up to speed, which again makes me feel anxious thinking about it.

If I do decide to have a team of developers, I think it would be better to have only a few people (maybe even just one or two) who can dedicate their full time (or maybe part time) to the project. But if I do go that route, I would have to go through the trouble of finding these people and possibly hiring them. Oh and not to mention actually raising the money to hire someone. (Kickstarter maybe?)

Developers aside, all these people being excited and optimistic about this project has also put a lot of pressure on me to perform and deliver something. I thought I would be ok with it, but it's starting to get to me.

---

# Conclusion

In my venture to create a DAW I went down many rabbit holes such as DSP, DSP optimizations, audio graphs, plugin hosting, loading and playing audio files, connecting to system audio and MIDI devices, GUI development, GUI performance, etc. In my years of programming I also have experience in things like GPU programming, scripting languages, and game development. I've gotten to know several DAWs very well such as FL Studio, Logic, and Bitwig. I also have an interest in drawing, graphic design, and UX design.

So I feel like if anyone in the world can make this project happen, I have a great shot at it. I want to do something meaningful with all the things I've learned.

But on the other hand do I want to become a project lead or even a CEO of a nonprofit to make that happen? I just don't know. I'm really questioning if I'm enough of a social-oriented person to do that or be happy doing that.

Maybe I would be happier if I kept working on Meadowlark alone? I'm just not sure. It seems to have worked out for the developers of projects like [Vital](https://github.com/mtytel/vital) and [Bespoke Synth](https://github.com/BespokeSynth/BespokeSynth).

And at the same time I'm still quite passionate about things like game development and music production. Maybe I would be happier doing that? I don't know. Point is this is a journey in my life that I'm still trying to figure out.

I'm also seeking professional help by seeing my therapist and my psychiatrist again. (Still have to wait a while to get an opening for an appointment though.)

In the end I really just need to take some time off and figure things out. I'm lucky to have a very supportive and loving family to help get me through this. Thanks for bearing with me!