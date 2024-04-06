+++
title = "Safe (and Fast) Audio Buffer API in Rust"
date = 2022-03-31
[taxonomies]
categories = ["CLAP", "Audio Graph", "DAW Engine"]
tags = ["CLAP", "Audio Graph", "DAW Engine"]
+++

> UPDATE: I'm not really happy with this article anymore. I've ended up not actually using this code, and there are many things I would change about it today (including removing the unsafe). I may update this at some point, but I'm busy with other things right now.

---

While writing my DAW engine in Rust, I've came across a very Rust-specific problem when it comes to audio buffers for plugins.

I plan on having first-class support for the awesome new open-source [CLAP] plugin spec (although this problem isn't specific to that spec). In it, the host sends the audio buffers to the plugin in a couple of small and tidy structs:
```c
// C

typedef struct clap_process {
   // ...

   // Audio buffers, they must have the same count as specified
   // by clap_plugin_audio_ports->get_count().
   // The index maps to clap_plugin_audio_ports->get_info().
   //
   // If a plugin does not implement clap_plugin_audio_ports,
   // then it gets a default stereo input and output.
   const clap_audio_buffer_t *audio_inputs;
   clap_audio_buffer_t       *audio_outputs;
   uint32_t                   audio_inputs_count;
   uint32_t                   audio_outputs_count;

   // ...
} clap_process_t;

typedef struct clap_audio_buffer {
   // Either data32 or data64 pointer will be set.
   float  **data32;
   double **data64;
   uint32_t channel_count;
   uint32_t latency;       // latency from/to the audio interface
   uint64_t constant_mask; // mask & (1 << N) to test if channel N is
                           // constant
} clap_audio_buffer_t;
```

All fine and dandy, but a couple conundrums arise with the how the plugin is allowed to define what buffers it wants for each of its audio ports:

```c
// C

enum {
   // This port is the main audio input or output.
   // There can be only one main input and main output.
   // Main port must be at index 0.
   CLAP_AUDIO_PORT_IS_MAIN = 1 << 0,

   // The prefers 64 bits audio with this port.
   CLAP_AUDIO_PORTS_PREFERS_64BITS = 1 << 1,  // <----------------- 1
};

typedef struct clap_audio_port_info {
   // ...

   uint32_t flags;
   uint32_t channel_count;

   // ...

   // in-place processing: allow the host to use the same buffer for
   // input and output if supported set the pair port id.
   // if not supported set to CLAP_INVALID_ID
   clap_id in_place_pair;  // <----------------------------------- 2
} clap_audio_port_info_t;
```

1. Each audio port can choose whether it wants to use 32 bit or 64 bit buffers. This means you can have plugins that requests 32 bit buffers for some ports and 64 bit buffers for others. To make matters more complicated, this only hints to the host that the plugin *wants* 64 bit buffers, but the host may still give it 32 bit buffers regardless. Luckily on the flipside the host may not send 64 bit buffers to plugins that haven't requested it.
2. Any pair of input and output ports may be bounded together into their own "in_place_pair". This tells the host that it may send a single buffer for both ports in that pair to save CPU resources, akin to `process_replacing()` in VST.

# Dealing with mixed 32 bit and 64 bit buffers

---

To start with the first problem, we can use an enum for `clap_audio_buffer_t` that can either be 32 bit or 64 bit buffers:
```rust
// Rust

pub enum AudioBuffer {
    F32(Vec<Vec<f32>>),
    F64(Vec<Vec<f64>>),
}
```

Fine enough for plugins that requested 64 bit buffers, but for plugins that haven't made that request, we are requiring them to essentially add a runtime check for something that is guaranteed to be the 32 bit variant. This runtime check may not be a big deal for most plugins, but I would like to have the ability to avoid it if possible. The solution I came up with is to use the `unwrap_unchecked` option built into Rust's `Option` type:
```rust
// Rust

pub struct AudioBuffer {
    pub float: Option<Vec<Vec<f32>>>,
    pub double: Option<Vec<Vec<f64>>>,

    // Ensure that this struct can only be initialized with
    // `new_f32()` or `new_f64()`.
    _private: (),
}

impl AudioBuffer {
    // These initializers ensure that either `float` or `double` will
    // always be `Some`.
    pub fn new_f32(float: Vec<Vec<f32>>) -> Self {
        AudioBuffer { float: Some(float), double: None, _private: () }
    }
    pub fn new_f64(double: Vec<Vec<f64>>) -> Self {
        AudioBuffer { float: None, double: Some(double), _private: () }
    }
}

// --------------------------------------------------------------------

// Now the plugin that requested 64 bit buffers can retrieve them like
// this:

if let Some(b) = buffer.double.as_mut() {

    // 64 bit dsp stuff

} else {
    // The user can safely use `unwrap_unchecked()` here because if
    // `double` is `None` then `float` must be `Some`.
    //
    // Or they can still use the `if let Some()` trick if they don't
    // want any unsafe in their code, at the cost of an extra runtime
    // check.
    let b = unsafe { buffer.float.as_mut().unwrap_unchecked() };

    // 32 bit dsp stuff

}

// --------------------------------------------------------------------

// And plugins that have not requested 64 bit buffers can safely
// assume that the buffer is always 32 bit:
//
// Or they can still use the `if let Some()` trick if they don't
// want any unsafe in their code, at the cost of a runtime check.

let b = unsafe { buffer.float.as_mut().unwrap_unchecked() };

```

# Dealing with aliasing pointers

---

In C and C++, the host sends a single buffer for an "in_place_pair" of ports by storing the same pointer in both slots. For example, the host can send a single stereo buffer to a plugin with a single input/output like this:
```c
// C

float left_buffer[MAX_FRAMES];
float right_buffer[MAX_FRAMES];
float* stereo_buffer[2] = {&left_buffer, &right_buffer};

clap_audio_buffer_t input_buffer = {
    .data32 = &stereo_buffer,

    // ... initialize other stuff
};

clap_audio_buffer_t output_buffer = {
    .data32 = &stereo_buffer,

    // ... initialize other stuff
};

clap_process_t proc = {
    .audio_inputs = &input_buffer,
    .audio_outputs = &output_buffer,
    .audio_inputs_count = 1,
    .audio_outputs_count = 1,

    // ... initialize other stuff
}
```

This creates an aliased pointer to mutable data (the same pointer appears twice within the same struct). C and C++ are all hunky-dory with this, but Rust is not.

So how do we fix it in Rust?

After a lot of head-scratching and rewrites, I eventually came to this solution. Wrap all the input ports in an `Option`. If an input port is `None`, then it means that the host has given a single buffer for that input/output port pair. If that is the case then the shared buffer will live in the corresponding output port.

```rust
// Rust

pub struct ProcAudioBuffers {
    pub inputs: Vec<Option<AudioBuffer>>,
    pub outputs: Vec<AudioBuffer>,
}

// --------------------------------------------------------------------

// Now a plugin that has requested in-place can check if a
// particular pair is in-place or not:

// As a side note, the user may also safely use `get_unchecked`
// for indexing into the array of buffers if the plugin has
// defined an input/output port at that index.
let output = &mut proc_buffers.outputs[0];

if let Some(input) = &proc_buffers.inputs[0] {
    // Non-in-place DSP stuff
} else {
    // In-place DSP stuff
}

// --------------------------------------------------------------------

// If a plugin has not put an input port in an "in_place_pair",
// then the user may safely use `unwrap_unchecked()` to get that
// input port buffer without any checks at runtime.

let sidechain_in = unsafe {
    proc_buffers.inputs.as_ref().get_unchecked(1).unwrap_unchecked()
};
```

This may seem fairly simple in hindsight, but I only came to this solution once I accepted there is no clean way to get around making the user use `unsafe` to achieve the least possible amount of runtime checks. I've tried things like [complex enums] that tried to boil everything down into a single runtime check (switching on the enum), but it got ugly fast.

# Actual implementation

---

One more note before I wrap this up. You might have noticed that the user has mutable access to the input buffers in `ProcAudioBuffers`.

First off, I should note that the buffers in my DAW engine aren't just `Vec`'s, they actually look like this:

```rust
// Rust

#[derive(Clone)]
pub(crate) struct SharedAudioBuffer<T>
where
    T: Sized + Copy + Clone + Send + Default + 'static
{
    buffer: Shared<(UnsafeCell<Vec<T>>, UniqueBufferID)>,
}

impl<T> SharedAudioBuffer<T>
where
    T: Sized + Copy + Clone + Send + Default + 'static
{
    // ...

    #[inline]
    fn borrow(&self, proc_info: &ProcInfo) -> &[T] {
        unsafe {
            std::slice::from_raw_parts(
                (*self.buffer.0.get()).as_slice().as_ptr(),
                proc_info.frames
            )
        }
    }

    #[inline]
    fn borrow_mut(&self, proc_info: &ProcInfo) -> &mut [T] {
        unsafe {
            std::slice::from_raw_parts_mut(
                (*self.buffer.0.get()).as_mut_slice().as_mut_ptr(),
                proc_info.frames,
            )
        }
    }
}
```

Yes, that is an `UnsafeCell`. I won't get into detail an what this is all doing for now as I'll probably make another post about it in the near future, but for now you can read the [safety note] in the repo.

Anyway, you can see that instead of passing around owned `Vec`'s as our buffers, we are passing them around via a smart pointer (in this case [basedrop]'s `Shared` smart pointer).

I then make them accessible to the user via this safe wrapper (which is not in the repo yet btw):

```rust
// Rust

pub struct AudioBufferFormat<T>
where
    T: Sized + Copy + Clone + Send + Default + 'static
{
    // Make this private so the user doesn't have direct access to this
    // "unsafe" buffer.
    rc_buffers: SmallVec<[SharedAudioBuffer<T>; 2]>,
}

impl<T> AudioBufferFormat<T>
where
    T: Sized + Copy + Clone + Send + Default + 'static
{
    // ...
    
    /// Immutably borrow a channel.
    ///
    /// This will return `None` if the channel with the given index does
    /// not exist.
    #[inline]
    pub fn channel(&self, channel: usize, proc_info: &ProcInfo)
        -> Option<&[T]>
    {
        self.rc_buffers.get(channel).map(|b| b.borrow() })
    }

    /// Mutably borrow a channel.
    ///
    /// This will return `None` if the channel with the given index does
    /// not exist.
    #[inline]
    pub fn channel_mut(&mut self, channel: usize, proc_info: &ProcInfo)
        -> Option<&mut [T]>
    {
        // Safety: Mutability rules are upheld because this method borrows
        // `self` as mutable, preventing the user from borrowing the same
        // buffer twice.
        self.rc_buffers.get(channel).map(|b| b.borrow_mut() })
    }

    // ... more methods for borrowing more than one channel at once
}

pub struct AudioBuffer {
    pub float: Option<AudioBufferFormat<f32>>,
    pub double: Option<AudioBufferFormat<f64>>,

    // Make these private so the user can't modify them.
    latency: u32,     // latency from/to the audio interface
    silent_mask: u64, // mask & (1 << N) to test if channel N is silent
    channel_count: usize,
}

impl AudioBuffer {
    pub fn latency(&self) -> u32 {
        self.latency
    }

    pub fn silent_mask(&self) -> u64 {
        self.silent_mask
    }

    pub fn channel_count(&self) -> usize {
        self.channel_count
    }
}

// And finally, this is what gets passed into the plugin's `process()`
// method:

pub struct ProcAudioBuffers<'a> {
    pub inputs: &'a [Option<AudioBuffer>>],
    pub outputs: &'a mut [AudioBuffer],
}
```

# To be continued

---

This post showed how you can pass audio buffers from a Rust host to an internal Rust plugin. Next up I'll show how this audio buffer API can be used to make safe [CLAP] plugins (and possibly other plugin formats) in Rust, making it so you can use the same code for both an internal plugin in my DAW engine as well as an external CLAP plugin.

# Addendum

---

I've learned that writing blogs really helps with working through a problem. I'm excited to do this more often!

[CLAP]: https://github.com/free-audio/clap
[complex enums]: https://github.com/RustyDAW/rusty-daw-engine/blob/ddf260123ec69b41ef92e184f94ebfb8d42ce231/src/process_info/proc_audio_buffers.rs
[safety note]: https://github.com/RustyDAW/rusty-daw-engine/blob/main/src/audio_buffer.rs#L10
[basedrop]: https://github.com/glowcoil/basedrop