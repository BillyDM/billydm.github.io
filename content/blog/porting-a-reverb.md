+++
title = "Porting a Reverb"
date = 2024-05-02
[taxonomies]
categories = ["DSP", "Plugins"]
tags = ["DSP", "Plugins"]
+++

---

I know I should be working on finishing the GUI library, but for the past couple weeks I've been a bit obsessed over a side project 😅.

I quite like the sound of the reverb module from the [Vital](https://github.com/mtytel/vital) synth, and I've been wanting to port it to a standalone effect plugin. I'd also like to potentially add it as one of the selectable algorithms in Meadowlark's future built-in reverb plugin.

Of course being a 🦀-y guy, I wanted to try porting it to idiomatic Rust. Vital's codebase is fairly complicated since it uses lots of SIMD intrinsics and some raw pointers, but I was up to the challenge. (Although it ended up being a lot tougher than I thought.)

You can get the finished reverb plugin and view the code at [https://github.com/BillyDM/vitalium-verb](https://github.com/BillyDM/vitalium-verb). (Note there is no GUI for it yet at the time of this writing.)

# Parsing Vital's Code
---

> I'm going to link to the fully open source fork of Vital called [Vitalium](https://github.com/DISTRHO/DISTRHO-Ports/tree/master/ports-juce6.0/vitalium) since that's what I referenced when porting the code. It probably doesn't matter, but I just wanted to be extra sure I was only copying GPLv3 code.

The main bulk of the reverb DSP code is in [reverb.h](https://github.com/DISTRHO/DISTRHO-Ports/blob/31afd943cfa93da7f0193b6db7ba275ff810e5a8/ports-juce6.0/vitalium/source/synthesis/effects/reverb.h) and [reverb.cpp](https://github.com/DISTRHO/DISTRHO-Ports/blob/31afd943cfa93da7f0193b6db7ba275ff810e5a8/ports-juce6.0/vitalium/source/synthesis/effects/reverb.cpp).

The codebase contains its own cross-platform SIMD abstractions located in [poly_values.h](https://github.com/DISTRHO/DISTRHO-Ports/blob/31afd943cfa93da7f0193b6db7ba275ff810e5a8/ports-juce6.0/vitalium/source/synthesis/framework/poly_values.h) and [poly_utils.h](https://github.com/DISTRHO/DISTRHO-Ports/blob/31afd943cfa93da7f0193b6db7ba275ff810e5a8/ports-juce6.0/vitalium/source/synthesis/framework/poly_utils.h#L119), along with some algorithms for common mathematical operations located in [futils.h](https://github.com/DISTRHO/DISTRHO-Ports/blob/31afd943cfa93da7f0193b6db7ba275ff810e5a8/ports-juce6.0/vitalium/source/synthesis/framework/futils.h).

From what I can gather with my novice-level understanding of DSP (sorry if I get some of the terminology wrong here), the reverb is composed of the following parts:

* Wet/dry mix amount. It uses an [equal power fade](https://github.com/DISTRHO/DISTRHO-Ports/blob/31afd943cfa93da7f0193b6db7ba275ff810e5a8/ports-juce6.0/vitalium/source/synthesis/framework/futils.h#L346) function to turn that into amplitudes for the wet and dry signals.
* Four simple one-pole filters. Two are used to apply low-pass/high-pass to the dry signal before it is sent to the reverb tank. The other two are the low-shelf/high-shelf filters that dampen the feedback signal in the reverb tank. The code for the filter design is located in [one-pole-filter.h](https://github.com/DISTRHO/DISTRHO-Ports/blob/31afd943cfa93da7f0193b6db7ba275ff810e5a8/ports-juce6.0/vitalium/source/synthesis/filters/one_pole_filter.h).
* A chorus effect that is applied to the feedback signal in the reverb tank. Luckily for me it doesn't use the DSP from the chorus module, instead it seems to use a much simpler chorusing algorithm that is implemented inline.
* Three (I think) stages of allpass filters that are applied to the feedback signal in the reverb tank. Each allpass stage is implemented as a 4x4 matrix (I think). 
The first two stages are feedback filters (I think) and the last stage is a feed-forward filter (I think). *(I don't know how reverbs work tbh, I just know that allpass filters are involved somehow)*.
* A ring buffer used to delay the wet output. It contains a Catmull sub-sample interpolator implemented as a 4x4 matrix. The ring buffer and its interpolation algorithm are defined in [memory.h](https://github.com/DISTRHO/DISTRHO-Ports/blob/31afd943cfa93da7f0193b6db7ba275ff810e5a8/ports-juce6.0/vitalium/source/synthesis/lookups/memory.h#L136), the the code to construct the Catmull matrix and the value matrix live here in [poly_utils.h](https://github.com/DISTRHO/DISTRHO-Ports/blob/31afd943cfa93da7f0193b6db7ba275ff810e5a8/ports-juce6.0/vitalium/source/synthesis/framework/poly_utils.h#L138), and the matrix struct itself lives in [matrix.h](https://github.com/DISTRHO/DISTRHO-Ports/blob/31afd943cfa93da7f0193b6db7ba275ff810e5a8/ports-juce6.0/vitalium/source/synthesis/framework/matrix.h).
* Multiple ring buffers used to hold the feedback memory. A polynomial sub-sample interpolator is used to read from this memory. Like the delay ring buffer's interpolator, this interpolator is implemented as a 4x4 matrix, with the code to construct this matrix here in [polyutils.h](https://github.com/DISTRHO/DISTRHO-Ports/blob/31afd943cfa93da7f0193b6db7ba275ff810e5a8/ports-juce6.0/vitalium/source/synthesis/framework/poly_utils.h#L119), and the matrix struct itself being the same as above.
* And finally, multiple ring buffers used to hold the state of the allpass filters. The reverb does not use sub-sample interpolation to read from these.

Special notes:

* All parameters are linearly smoothed by calculating a delta amount and adding that delta to the parameter every frame.
* All ring buffers have a size equal to a power of 2. This allows indices to be cheaply constrained to a valid range by bitwise and-ing them with a mask.
* The pointers to the feedback and delay memory ring buffers are offset by 1. This is needed because the sub-sample interpolators read one sample in the past, which would cause an out-of-bounds read if there wasn't that offset.
* At the top of each call to process, the last 4 samples in each feedback memory ring buffer are wrapped to the front of the buffer. I'm not sure exactly why this is done, but I think it has something to do with how the sub-sample interpolator works.

# Initial Attempt
---

I used the great [nih-plug](https://github.com/robbert-vdh/nih-plug) framework for my plugin.

I started out with making my own simple SIMD abstractions similar to how Vital does it, instead of using the nightly-only [portable-simd](https://github.com/rust-lang/portable-simd) library. I wanted to follow Vital's code as closely as possible. (Although I later regretted this decision and switched to using `portable-simd` as you will see later on).

I then added the simple stuff like the wet/dry gain and the lowpass/highpass filters that are applied to the dry signal before it is sent to the reverb tank. I also decided to go with separate wet/dry parameters instead of a single "mix" parameter like the original C++ code had. I encountered a small bug with the gain dipping right before 0 decibels (I put a parenthesis in the wrong place), but other than that it was smooth sailing and I quickly got it working.

After that, I wasn't really sure how to break down the algorithm any further since I don't really understand how reverbs work that well. So I just went for a hail mary and implemented all the rest of the reverb. Now compile it, load it into a DAW, *and*...

Nothing except a constant DC offset on the output. Oh boy, where do I start debugging this?

# Debugging Adventure
---

Since I don't have any equations to reference from, I decided to isolate all the relevant C++ code into a small test project where I could run a sine wav through it and record the values at every single step in the process. I did the same in Rust and compared them to see where things went wrong.

>Yes I could have used a debugger to step through it instead of printing every variable to the terminal, but I'm lazy and didn't feel like learning how to get the C++ debugger working with Linux and [VSCodium](https://vscodium.com/). Yes I'm sure it's easy, but I'm just used to debugging this way. Plus I'd rather have a single long list of values to reference from to more easily see where things go wrong.

Oh boy were there a lot of problems.

The first things I caught were some parenthesis in the wrong place and some multiply asterisks that were supposed to be plus signs. There were also a few places where I got the member variables mixed up with local variables that had the same name but without an underscore at the end (ugh, I'm so thankful Rust makes you use `self.` to access member variables).

The next thing that took me forever to figure out what was wrong is in the calculation of the allpass offsets. Here you can see what the C code was generating and what my rust code was generating:
```
// c++ ----------------------------------
allpass_offset1: [6395, 8012, 7009, 7466]
allpass_offset2: [6459, 7164, 6825, 7258]
allpass_offset3: [8155, 7660, 4537, 5690]
allpass_offset4: [6235, 6668, 7977, 5306]

// Rust --------------------------------------
allpass_offsets = [
    [3, 8012, 1, 7466],
    [3, 7164, 1, 7258],
    [3, 7660, 1, 5690],
    [3, 6668, 1, 5306],
]
```
Weird, some of the numbers are correct but some of them aren't. I tried looking at every step in the process, trying to see where my code was wrong. Eventually I finally found the culprit. Apparently the `_mm_mul_epi32` intrinsic I was using to multiply i32 vectors is SSE4.1, not SSE2, and so it was causing funky behavior on my AMD CPU. I then looked at how Vital implements its multiply function for i32 vectors:

```c++
static force_inline simd_type vector_call mul(simd_type one, simd_type two) {
#if VITAL_AVX2
      return _mm256_mul_epi32(one, two);
#elif VITAL_SSE2
      simd_type mul0_2 = _mm_mul_epu32(one, two);
      simd_type mul1_3 = _mm_mul_epu32(_mm_shuffle_epi32(one, _MM_SHUFFLE(2, 3, 0, 1)),
                                       _mm_shuffle_epi32(two, _MM_SHUFFLE(2, 3, 0, 1)));
      return _mm_unpacklo_epi32(_mm_shuffle_epi32(mul0_2, _MM_SHUFFLE (0, 0, 2, 0)),
                                _mm_shuffle_epi32(mul1_3, _MM_SHUFFLE (0, 0, 2, 0)));
#elif VITAL_NEON
      return vmulq_u32(one, two);
#endif
    }
```
Apparently it's kind of complicated to multiply i32 vectors in SSE2, but once I implemented this in my code I was finally getting the correct values for allpass_offsets.

After some more debugging I eventually got it to the point where I was getting the same values up to the first pass of the processing loop. I felt like I was finally on the home stretch, but when I loaded it into the DAW, nothing. There was no wet output, only the dry signal playing through.

I then discovered that I needed to look much further in the processing loop before the first non-zero value is read from the feedback and allpass memories. That makes sense, a reverb is essentially a bunch of echos that take time to be reflected back. I found that with the parameters I was using, the first frame in the C++ code where the allpass reads a non-zero is frame 1136, and the first frame where the feedback reads a non-zero is frame 2930. 

Looking at the output of the Rust code, it was indeed showing that it was always reading zeros even after these frames. I checked and double-checked the complex interpolation algorithms to see if there was a mistake. Then I thought to see if maybe it was the memory ring buffers themselves that were all zeros, and sure enough, they were.

This was strange because there didn't seem to be anything wrong with the code for writing to these ring buffers:

```rust
// in the PolyF32 struct

#[inline(always)]
pub fn store_into_slice(&self, a: &mut [f32; 4]) {
    unsafe {
        _mm_storeu_ps(a.as_mut_ptr(), self.0);
    }
}

// in the reverb code

(scaled_input + delay_input).store_into_slice(
    &mut allpass_lookup[allpass_write_index..allpass_write_index + 4]
    .try_into()
    .unwrap()
);
```

Eventually I figured out that using `.try_into().unwrap()` to turn a slice of variable length into a slice of constant length doesn't actually work. What it actually does is clone the slice into a temporary array, which then just gets discarded after the function is over.

Maybe there's a way to idiomatically convert a slice of variable size to a slice of constant length, but at this point I figured it was probably better to just use the `portable-simd` library in Rust since it already has all these kinds of quirks figured out. As a bonus this will also automatically let this code work for other processor architectures as well.

After fixing a few bugs from refactoring it to use the `portable-simd` library, it was finally time to load it into a DAW to see what will happen, *and*...

SUCCESS! (mostly) With the chorusing effect on, the wet signal was very noisy. But with it off, the reverb seemed to be working fine.

So I spent a while longer figuring out where my chorus code went wrong, and I just couldn't find it.

Then I finally figured it out, the code wasn't wrong, Vital just has the chorus parameters on a logarithmic curve that is heavily weighted towards the smaller values. When I loaded up Vitalium and turned the chorus knobs all the way up, sure enough there was that same noisiness. So adding a similar curve to my parameters finally fixed it!

# Improvements
---

Now I need to tweak the parameter curves until they feel right. Although I couldn't find a setting with nih-plug's built-in curves that made the "decay" parameter feel right. The decay parameter goes from 0.1 seconds all the way up to a crazy 64 seconds for creating those ambient drone effects. I want the majority of the parameter to take up the typical range of 0.1 - 5 seconds, and then have the all the rest of the values take up a small portion of the parameter. So I ended up making my own custom [piece-wise function mapping](https://github.com/BillyDM/vitalium-verb/blob/1d3fe0d80939ecd3f308d2a3f0c20c3b57d322d3/src/lib.rs#L349) for it.

I also decided to switch to using a single "mix" parameter like what Vital has instead of separate wet/dry parameters. After trying both methods I like the single mix parameter better.

I also decided to add a stereo width parameter to the wet output since the [algorithm](https://www.musicdsp.org/en/latest/Effects/256-stereo-width-control-obtained-via-transfromation-matrix.html?highlight=width) for this is pretty simple. I actually think the reverb sounds even better with the width parameter set to around -5%, so I'm really glad I did that!

I also made some optimizations. The original code calculated the values/deltas for every parameter on every call to `process()`, so I added code to only recalculate values for parameters that have changed.

# Conclusion
---

After this experience, I think it could be worth looking into compiling complex C++ DSP into a static library that can be used by Meadowlark and its plugins, instead of going through the painstaking process of translating it all to Rust. I have some experience doing this before with my [RtAudio bindings](https://github.com/BillyDM/rtaudio-sys).

And if the DSP is written in C, I can even consider transpiling it using [C2Rust](https://github.com/immunant/c2rust) so there isn't anything special needed in the build system. The [rust-libsamplerate](https://github.com/RamiHg/rust-libsamplerate) crate is a great example that uses this technique.

Still, for simpler DSP, especially ones that don't use lots of SIMD intrinsics or 3rd party libraries, I'd still prefer to have those ported those to Rust.
