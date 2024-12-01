+++
title = "VitaliumVerb 1.2.0"
date = 2024-12-01
[taxonomies]
categories = ["Plugins", "GUI"]
tags = ["Plugins", "GUI"]
+++

---

It's been a while since I posted anything. I should get better at staying on top of that.

I've of course been working on my main projects (and I will maybe post an update on those soon), but I figured I would announce a small update I made to my reverb plugin port [VitaliumVerb](https://github.com/BillyDM/vitalium-verb) I talked about in my previous blogpost [Porting a Reverb](../porting-a-reverb).

# It Has a GUI! 🚀
---

I went ahead and made a quick n' dirty (but still visually pleasing) GUI in the [Vizia](https://github.com/vizia/vizia) GUI framework!

![VitaliumVerb screenshot](/images/vitaliumverb-update/VitaliumVerb-screenshot.png)

> Pre-built binaries for Linux, Mac, and Windows can be downloaded from the [Releases](https://github.com/BillyDM/vitalium-verb/releases) tab in the GitHub repository!

So why did I choose Vizia and not my own GUI framework that I've been working on? Well for one, my GUI framework is not finished yet 😅, and two, [nih-plug](https://github.com/robbert-vdh/nih-plug) has a ready-made, easy-to-use slider widget for Vizia.

While I would prefer knobs and an eq widget like the original Vital synth had, this will do for now. Maybe at some point I will create a fancier GUI in my own GUI framework.

# Other Changes
---

Another small (but breaking) change I made is that the stereo width parameter now ranges from `0.0%` to `200.0%` instead of `-100%` to `100%` like a stereo width parameter should. This should make it clearer what the parameter does.

# Addendum
---

Someone was asking what the "minor improvements and optimizations added" were as mentioned in the readme, and I realized I forgot to summarize this in my previous blogpost.

Essentially it's just these three things:
* A stereo width parameter was added to the wet signal.
* Parameter curves were tweaked to focus better on the sweet spots (and because I wanted to use nih-plug's built-in parameter curves as much as possible).
* Runtime-evaluated constants like filter coefficients, gain amplitudes, chorus phase increments, and allpass matrices are only recalculated when their respective parameters have been changed (the original recalculated these every process cycle).
