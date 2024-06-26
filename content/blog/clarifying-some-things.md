+++
title = "Clarifying Some Things, and What I'm Doing Now"
date = 2023-05-16
[taxonomies]
categories = ["Misc"]
tags = []
+++

# Preface

---

Welp, apparently one of my previous posts [DAW Frontend Development Struggles](../daw-frontend-development-struggles) is trending on Hacker News and the Rust subreddit. Couldn't have come at a more awkward time.

I want to take this time to clarify some things, as well as what my current plans for Meadowlark are. Some of my opinions have changed somewhat since that post.

# Clarifications

---

I want to clarify that I do think a fully-featured GUI library in Rust is possible, it's just that it's still a few years away.

Also I want to state that my complications have more to do with the fact that I'm writing a complicated DAW GUI and not a typical application. Existing Rust GUI libraries are already (mostly) competent at this.

And yeah, I was maybe a bit quick to dismiss [slint](https://slint-ui.com/) and [makepad](https://github.com/makepad/makepad) in my post. I now have a different reason for not going with those which I'll explain below, but I don't want to downplay the potential of these projects (and also projects like [vizia](https://github.com/vizia/vizia) and [iced](https://github.com/iced-rs/iced)).

I now think declarative is probably possible to use for something as complex as a DAW (although it's still more cumbersome IMO). Immediate mode might also work if executed correctly.

The [xilem](https://github.com/linebender/xilem) architecture also seems very promising in solving a lot of the problems that a GUI library in Rust has. But of course it's just an experiment at the moment.

Also my opinion on the importance of damage-tracking has changed somewhat. I no longer think it's *that* important in the advent of modern GPUs. But I do still think that more consideration should be given to minimizing the work being done on the GPU.

# Sparking Discussion on Rust GUIs

---

That being said, I think the original opinions I had in my previous post illustrate an important point.

It goes to show that a lot of work and careful considerations are needed if we ever want a truly general purpose Rust GUI toolkit to take over. I'm sure I'm not the only one who has been frustrated by the state of the Rust GUI ecosystem.

# GUI is *Hard*

---

I can't stress this point enough. There's so much more to GUI than meets the eye.

### Edit:

Now while that point still stands, I think I was being unfair with my previous edit.

There's a lot of "one man" Rust GUI libraries out there. There's nothing wrong with that, and they can already be used for a wide variety of applications. I think most if not all these projects aren't actually trying to directly compete with the likes of QT, JUCE, GTK, Flutter, Electron, etc. They are aiming to be a solution for a different problem, and I believe they are achieving that very well.

My personal favorite is [egui](https://github.com/emilk/egui), it makes it so easy to whip up prototypes, and the performance isn't bad either.

With that out of the way, I still want to give you my list from the previous edit. The point I'm now making is that if the goal of a particular Rust GUI library *is* to compete in the big leagues, there are a lot of complicated hidden details that must be addressed. While not every one of these features need to be included, I think it gives a good overall idea of the complexity we're dealing with here.

Some of these features include:

* Extensive documentation, examples, and tutorials (hello worlds and 7GUIs alone aren't enough)
* Accessibility features. This one is hard to get right.
* Proper unicode text support. Even better if it has support for inline styling. This one is *very* hard to get right.
* Text that looks sharp even at small sizes. I say this because a lot of Rust GUI libraries currently use GPU-based text rendering which doesn't look very good.
* Built-in features to help with localization
* Support to easily use custom icons
* Proper multi-window support (this is harder than you might think)
* Proper drag-and-drop support. This includes both internal drag-and-drop, and drag-and-drop from the OS.
* Proper touch support. Even if you aren't targeting mobile, users with touchscreen laptops and tablets will still expect touch to work.
* Flexible state management and widget composition. Real-world applications commonly have unusual requirements and edge cases, and so the user shouldn't be locked into one way of doing things. This one is more a matter of opinion though, and it may be possible to create an API that covers most if not all use cases.
* Features that allow the user to better optimize the performance of their UI. Don't leave optimizations decisions purely up to the UI engine.
* Proper support for drawing custom widgets. This includes the ability to draw more complicated shapes like arcs and Bézier curves. These complex shapes don't always need to be GPU-accelerated, but they need to be possible. A common use case to be aware of is interactive graphs and plots.
* Support for custom shaders (along with the ability to draw widgets on top of the shader area). People will want this. This is also now more complicated because Apple is getting rid of support for OpenGL. You now need to support Metal as well. This will also be needed if you want to support video playback.
* If using immediate mode, make sure to use a lot of caching techniques to meet performance demands when scaled to large project sizes (or even use retained-mode under the hood).
* If not using damage tracking for rendering, give careful consideration on how to optimize, minimize, and cache the commands that are sent to the GPU.
* If using a declarative API, make sure that it's possible to declare the UI in code rather than a markup language (and have documentation and examples for how to do this). Not everyone likes using the markup language approach.
* Fast incremental compiles (or even better, hot-reloading)
* Support for [pointer locking](https://developer.mozilla.org/en-US/docs/Web/API/Pointer_Lock_API). Well ok, this is a pretty niche feature, but let me tell you if you do need it it really sucks if the UI library doesn't support it.
* Bindings to other languages (especially a scripting language like Python or ~~Javascript~~ Typescript). This alone will make the project accessible to a *lot* more people.
* Detailed unicode text editing (single-line and multi-line). This is *very* hard to get right.
* Ability to click links in user-generated text
* Support for a variety of image formats, including SVG and GIF
* Loading placeholders for images and other content that is being downloaded
* Ability to play sound effects (although this one could be handled by a separate crate)
* Ability to playback videos (offline and streaming). This should also include control overlays.
* Integration with native menubars and other window controls
* Support for borderless windows
* Integration with native dialogs such as file dialogs and print dialogs
* Integration with OS notifications and media controls
* Proper password input, as well as integration with the OS's keychain
* Support for custom keyboard shortcuts, along with the ability to change those shortcuts at runtime
* Kinetic scrolling for touch screens (although this probably isn't that necessary unless you're targeting mobile)
* Infinitely scrolling lists (very hard to get right)
* Ability to move panels using drag-and-drop, as well as the ability to pop-out panels into a floating window
* Support for custom layouts for things like node editors
* Animation support
* Nested drop-down menus, as well as the ability to scroll drop-downs that are taller than the screen.
* Proper nested tree widget
* Proper table widget
* Some more advanced widgets like calendars, color selectors, and emoji input dialogs (although these could be handled by third-party extensions)
* Also don't forget to include any of the essential widgets. These lists of built-in widgets in [GTk3](https://docs.gtk.org/gtk3/visual_index.html) and [GTK4](https://docs.gtk.org/gtk4/visual_index.html) can give you a good idea.

> JUCE doesn't actually have a lot of these features. Its focus is on audio software, and not general use. Existing Rust libraries like [vizia](https://github.com/vizia/vizia), [iced](https://github.com/iced-rs/iced), and [egui](https://github.com/emilk/egui) are already quite competent at the audio plugin use case (which is the majority of JUCE's user base).
>
> The [rui](https://github.com/audulus/rui) library also seems promising for this use case since it is specifically being used for a commercial audio application. Though I'm not sure anyone has actually tried to make an audio plugin with it yet.

# What I Learned About Open Source

---

I learned a very important lesson from my struggles. Not just my struggles with the UI, but with Meadowlark itself.

You can't rely on volunteers to build key components of a large open source project. Unless you have found someone else who is as passionate as you are, has the same vision as you do, and has as much free time as you do (or unless you're running a business with employees), you must be prepared that you will be working on it alone for a *long* time. You must be prepared that you will do the vast majority of the work yourself. You also need to understand your limits as a solo developer (which is something I previously grossly miscalculated).

# What I'm Doing Now

---

I learned that I simply cannot make both a DAW engine and a DAW frontend on my own. I severely underestimated the frontend. I'm only one person.

But that's ok, because I actually have found someone else who is working on their own DAW frontend, and they are very good at it. By coincidence (or maybe it's fate?), they share a similar vision to what I had with Meadowlark's UI. So we decided to team up, with them working on the frontend and me working on the backend engine.

They're using Flutter, and it's impressive what they were able to do with it. So Meadowlark (or whatever we end up calling it) will use Flutter for mainly that reason.

They were originally going to use the [Tracktion Engine](https://www.tracktion.com/develop/tracktion-engine) for the backend, but they agreed that having a new open source engine would be beneficial.

And so here's what I'm doing now: I'm creating my own DAW engine like Tracktion's, but with these notable advantages:

* Modular ecosystem: you only need to include the parts of the engine you use
* Good documentation
* Written in Rust, with all the safety advantages that brings
* Zero dependencies on JUCE
* Full first-class support for all of [CLAP](https://github.com/free-audio/clap)'s features, allowing for some exciting new ways to use plugins
* Better control over the engine: it doesn't force you to use a certain workflow
* Additional bindings to C and C++
* *Maybe* MIT license? I haven't decided on that yet.

Maybe I might even try monetizing the engine in some way. Whether that be dual-licensing or sponsorships, I haven't decided yet.

This DAW engine will be called [Dropseed](https://github.com/MeadowlarkDAW/dropseed) (following the naming scheme I have of native fauna and flora in Kansas. Dropseed is the name of a grass).

I'm also creating an FL Studio Patcher-like plugin as a testbed for my engine. I eventually want to have a Patcher-like plugin in Meadowlark anyway, so it's still a good use of my time.

> BTW, I'm not going to mention who this person is or what project they were working on yet. I don't want to drive unwanted attention towards them.