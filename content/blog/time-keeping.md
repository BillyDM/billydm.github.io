+++
title = "Accurate Timekeeping in a DAW"
date = 2022-11-21
[taxonomies]
categories = ["DAW Engine"]
tags = ["DAW Engine"]
+++

Among the many things a DAW needs to do, keeping track of when events (i.e. midi notes, automation nodes, the start and end of audio clips, etc.) should occur is one of them. I want to share the solution I've come up with for my DAW engine. It was partly inspired by [this article](https://ardour.org/timing.html) by the Ardour team, as well as [their source code](https://github.com/Ardour/ardour/tree/master/libs/temporal) if you want some further reading.

# The Problem

---

In short, the problem is that there isn't just one type of time we're dealing with. There are actually four distinct "time domains":

* Musical Time - Time measured in musical units such as beats and measures. This is the time domain that sheet music deals with, and is what the user usually thinks of when they compose music.
* Sample Time - Time measured in the number of samples (for a single channel of audio). This time domain is usually used for audio clips and when the user is editing a project in a non-musical context (i.e. editing a podcast or writing a movie score). This time domain is also used by your sound card and audio plugins.
* "Real Time" (Seconds) - Time measured in seconds. This time domain is used more as an intermediate step for conversions (as well as a visual aid for the user) than it is used as an actual format to store events in.
* Pixels - Okay, this one might be a bit of a stretch. But it still stands that we need to convert whatever time format we use into the actual pixel position to draw something on the screen.

> I've actually discovered a fifth time domain which I probably need to add support for eventually, which is timecodes in video formats. These are usually in units of frames, and framerates differs depending on the video format.

Now the equations to convert between these time domains may seem pretty simple on first inspection, for example:
* `MusicalBeats = Seconds * BPM / 60.0`
* `Seconds = SampleTime / SampleRate`
* `Pixels = MusicalBeats * PixelsPerBeat * ZoomFactor`

But things become immediately harder once you want to support automated tempo in a project. Now we're no longer dealing with simple ratios, we are now dealing with a complicated piece-wise function.

Although if that was the only problem, then I wouldn't be writing this article. There is bigger hidden problem, which is that ratio calculations on computers are imprecise due to the nature of floating point numbers. If you go willy-nilly with how you store time data and constantly convert back-and forth between time domains, things can start to get out of sync. Not only that, but depending on the time domain you originally stored the information in, if you change the tempo or the sample rate of the project, your events can now be occurring at slightly different times. If you try to change it back, it still may not be the original value we started with, and thus we have lost information.

# The Solution

---

## Part 1: The "Source of Truth"

The first part of the solution is to define what your "source of truth" is. By this I mean that all conversions should be performed from the "source of truth" time domain to whatever time domain we need, and never the other way around (whenever possible).

In a DAW, there are generally two sources of truth being used: musical time and sample time.

### Musical Time

For the majority of events in a DAW, musical time is the best source of truth. This is because the information stored in musical time is independent of both tempo and sample rate, so we won't lose any information when changing the BPM or sample rate.

If we want to support events with sample-accurate precision, we can still use musical time as long as our format has enough precision.

For example, these types of events will be stored in musical time:
* The start and end positions of piano roll (MIDI) notes in a piano roll clip.
* The position of automation nodes in an automation clip.
* The start and end positions of piano roll clips and automation clips on the timeline.
* The start position of audio clips on the timeline.

### Sample Time
This is the best format to use when dealing with audio clips. This is because these types of events deal with actual samples in the audio clip, and so unlike musical time these events *are* dependent on the sample rate and the tempo. For example, you don't want a change in the tempo to change the sample on which the event occurs.

This format is also more reliable when editing in a non-musical context (i.e. editing podcasts or editing the sound of a video), so in those cases certain timeline events should be stored in sample time instead.

> It would actually be better to use video timecodes as the source of truth while editing the sound of videos, so that'll be a third one I'll add to my code later.

## Part 2: The Actual Format

Now that we've decided our sources of truth, what format should we use to actually store them? If we used floats, we would run into the same imprecision errors we mentioned above. So my solution uses fixed point. But not just any fixed point, fixed point with a specific modulus.

### Musical Time

Musical Time is stored in units of musical beats, where the fractional part uses a modulus of `1,476,034,560` (In other words, one unit in the fractional part is equal to exactly `1 / 1,476,034,560` of a musical beat).

Why `1,476,034,560`? Because it's the least common multiple of `2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 18, 20, 24, 32, 64, 128, 256, 512, 1,024, 2,048, 4,096, 8,192, 16,384, and 32,768`. This allows us to create those subdivisions of musical beats with *exact* precision. For example, because it's a multiple of 3, we can represent a triplet (`1/3`) as exactly `1,476,034,560 / 3 = 492,011,520`, as apposed to the infinitely repeating digits that occur in both decimal and binary.

Using a fixed point modulus is not a new concept in the audio world, in fact MIDI devices have been using `1920` as their modulus since the early days. This number neatly covers half-notes, triplets, quarter-notes, 5th-notes (whatever those are called), 6th-notes, eight-notes, 10th-notes, 12th-notes, 15th-notes, 16th-notes, 32nd-notes, 64th-notes, and 128th-notes (and some others).

My number goes further by also covering 7th-notes, 9th-notes, 11th-notes, 13th-notes, 14th-notes, 256th-notes, 512th-notes, 1,024th-notes, 2,048th-notes, 4,096th-notes, 8,192th-notes, 16,284th-notes and 32,768th-notes.

But more importantly my number is much larger than all the common sample rates (`22,050, 24,000, 44,100, 48,000, 88,200, 96,000, 176,400, 192,000, 352,800, and 384,000`). This ensures that we have enough precision for sample-accurate events, even at very high sample rates and at very low BPMs.

Also this number is just under the 32-bit limit of `4,294,967,296`, allowing us to make the most use of the 32-bit unsigned integer we use for the fractional part of our format, while also giving us a little bit of buffer room to avoid overflow when adding fractional parts together.

Now that leaves us with another 32-bit unsigned integer for the non-fractional part. This gives us a maximum project length of `4,294,967,296` musical beats. Even at a ludicrous tempo of 300 BPM, that still gives us a maximum time of `9,942` days. So nothing to worry about. (And for you speedcore fans, that's still 29.8 days at 100,000 BPM).

> Note, I opted for unsigned integers instead of signed ones because fixed-point becomes far more complicated when negatives are involved. For pre-rolls, I'm just going to offset the start of the song.

Here is my implementation of `MusicalTime` [in code](https://github.com/MeadowlarkDAW/meadowlark-core-types/blob/main/src/time/musical_time.rs). Note I'm calling this special fraction of a beat a `Tick`.

### Superclock Time

For sample time I'm doing something similar, but with a modulus of `282,240,000` (In other words, one unit in the fractional part is equal to exactly `1 / 282,240,000` of a second). This number is divisible by all the common sample rates `22,050, 24,000, 44,100, 48,000, 88,200, 96,000, 176,400, 192,000, 352,800, and 384,000`. This allows us to change the sample rate without losing any information.

I also chose this modulus because it's also the number that Ardour uses, so perhaps that could lead to better compatibility between Meadowlark and Ardour project files.

I'm calling this special format `SuperclockTime` to avoid confusion with normal sample time.

Here is my implementation of `SuperclockTime` [in code](https://github.com/MeadowlarkDAW/meadowlark-core-types/blob/main/src/time/superclock_time.rs). Note I'm also calling this special fraction of a second a `Tick`.

## Part 3: The Tempo Map

Now we need to handle automated tempo. This requires a piece-wise function where each section between automation nodes is a segment in this function. In my code, I'm calling the struct that stores the automation information for the project tempo and which does all the calculations the `TempoMap`.

I would give more details on how this will actually work in practice, but I haven't actually implemented it yet. I just know that it *should* be possible, and right now I'm focused on just getting a DAW with static tempo working before focusing on adding support for automated tempo. The `TempoMap` is there in my code, but right now it's just a placeholder that simply uses static tempo. However, I do know that the final implementation will probably employ some kind of binary search or binary tree to help speed up calculations. Once I have implemented it I might make an article about it.

What I can say is how this tempo map will be used by the rest of my code. Every time the tempo map is changed (i.e. when the user changes the static tempo or adds/removes/moves and automation node in the tempo automation lane), the tempo map converts every single event in the project from our special format to its corresponding time in samples, and then it sends the result to the audio thread.

While not the most efficient way to do things, it is the easiest and least error-prone. Doing it this way also follows our "source of truth" philosophy where conversions only ever happen from the source of truth formats to the other formats. Plus, changing the BPM in a project is a fairly rare operation in the full production lifecycle, so I don't think having performance possibly chug a little bit while moving a node in the tempo automation lane is a real problem.

Oh yeah, not only am I planning on supporting automated tempo with this tempo map, but also various groove and swing parameters. (Jazz, GlitchHop, and Electro Swing fans rejoice!)