+++
title = "RootVG"
date = 2024-04-06
[taxonomies]
categories = ["GUI"]
tags = ["GUI"]
+++

---

# Preface

A big piece of the puzzle for the GUI library I'm creating is more or less complete! It's a hardware-accelerated vector graphics rendering library for [wgpu](https://wgpu.rs/), which I call [RootVG](https://github.com/MeadowlarkDAW/rootvg). It can render quads, triangle meshes, images, text, and even custom primitives with custom pipelines, all using the amazing wgpu library.

> See the [demo example](https://github.com/MeadowlarkDAW/rootvg/blob/main/examples/demo.rs) for a quick overview of how the API works.

*The logo I made for RootVG :)*
<img src="/images/rootvg/rootvg-logo.svg" alt="RootVG logo" width="256px" height="256px"/>

---

# Why create RootVG?

There are four reasons. One, I wanted to create a rendering engine that is specifically tailored towards GUI. I wanted something that is lightweight* and has very little overhead (both for the CPU and GPU). Unlike other vector graphic rendering libraries like [NanoVG](https://github.com/inniyah/nanovg), [FemtoVG](https://github.com/femtovg/femtovg), [vger](https://github.com/audulus/vger-rs), or [Skia](https://skia.org/) which have a streaming drawing API similar to that of the [HTML5 Canvas API](https://www.w3schools.com/jsref/api_canvas.asp), users of RootVG construct reusable "primitives" that can be cheaply placed at any z index and inside of any scissoring rectangle in any order, and then be rendered using as few draw calls as possible. I'll explain how this works later on below.

Two, I wanted first-class support for custom shaders, as those will be very useful for creating efficient visualizers such as waveform displays, oscilloscopes, and spectrometers. WGPU makes it surprisingly simple to mix multiple pipelines together into a single render pass.

Third, I wanted to avoid targeting only OpenGL. Apple has been trying to depreciate OpenGL in MacOS, and I'd like to stay ahead of the curve. Additionally, on my Linux system specifically, I find that wgpu-based applications seem to have lower latency than OpenGL-based ones. Less latency in a GUI is always a win in my book.

And fourth, I wanted to take a deep dive into graphics programming in wgpu. I find graphics programming fun, and I also wanted the experience for potential fun side projects in the future (like a simple Minecraft-ish clone from scratch that I've always wanted to create). Wgpu has been a joy to work with and learn about.

> * While wgpu itself is relatively lightweight compared to something like [Skia](https://skia.org/), it's still not quite as lightweight as I would have hoped. On release mode with LTO enabled and debug symbols stripped, [cargo-bloat](https://github.com/RazrFalcon/cargo-bloat) shows wgpu itself taking up about 6MB of the binary. Oh well, it's good enough for now. I'd still rather use wgpu over interfacing with OpenGL, Vulkan, and Metal directly. Hopefully at some point wgpu gains the ability to omit including the shader compiler [naga](https://github.com/gfx-rs/wgpu/tree/trunk/naga) into the binary and instead [load in precompiled shaders](https://github.com/gfx-rs/wgpu/issues/3103). Using precompiled shaders should also improve startup times.

---

# The Primitive Types

RootVG comes with six built-in primitive types:

## SolidQuad and GradientQuad

![quad primitives](/images/rootvg/quad-primitives.png)

I've taken the quad primitives from the [Iced](https://github.com/iced-rs/iced) GUI library and modified them to be more performant for my use case.

A "quad" is essentially a rectangle with optional rounded corners and an optional outline with a given thickness. Each of the four corners can have a different amount of rounding. A circle can be made if the corners have a large enough rounding.

The solid variant has a background filled with a solid color, and the gradient variant has a background filled with a linear gradient (I might add support for a radial gradient in the future).

Like Iced, the gradient variant uses the [OkLab](https://bottosson.github.io/posts/oklab/) color space. It looks *oh so* nice 👌.

There's no support for applying a gradient to the outline though. It would make things a lot more complicated and I personally don't think it's necessary.

Also like Iced, the solid variant has support for an optional drop-shadow. I might add drop-shadow support the the gradient variant as well if I find a need for it or if there's demand for it.

## SolidMesh and GradientMesh

![mesh primitives](/images/rootvg/mesh-primitives.png)

I also borrowed these from the [Iced](https://github.com/iced-rs/iced) GUI library. Again I've modified them to be more performant for my use case.

These primitives simply render a collection of triangles. As the name suggests, the solid variant fills them with a solid color, and the gradient variant fills them with a linear gradient (again I might add support for radial gradients in the future). Like the `GradientQuad`, the `GradientMesh` also makes use of the [OkLab](https://bottosson.github.io/posts/oklab/) color space.

[MSAA](https://therealmjp.github.io/posts/msaa-overview/) anti-aliasing is used to smooth out meshes. Though unlike how Iced does it, in RootVG all primitives of all types are rendered in a single render pass. This can be a lot more efficient than constantly switching render passes for different types of primitives, especially on mobile hardware.

[Lyon](https://github.com/nical/lyon) can be used to generate meshes using a streaming drawing API. This is the intended way to create more complex shapes such as arcs and bezier curves. I've also borrowed some of the helper drawing methods from Iced.

## Text

![text primitive](/images/rootvg/text-primitive.png)

Text primitives are rendered using [glyphon](https://github.com/grovesNL/glyphon). Not much to say about it, just that I'm thankful the work has already been done for me here.

>Currently glyphon is not as optimized as it could be, especially for my use case. I might end up doing a soft-fork with RootVG-tailored optimizations.
>
>The rendering quality is also not quite as sharp as I would like it. Hopefully this will improve once glyphon gets sub-pixel support.

## Image

![image primitive](/images/rootvg/image-primitive.png)

This primitive simply renders an srgba image. Image primitives can also be scaled and rotated.

RootVG also supports using the output of a previous render pass as the source of an image primitive, as shown in the [prepass_texture example](https://github.com/MeadowlarkDAW/rootvg/blob/main/examples/prepass_texture.rs). This allows for tricks such as rendering an expensive waveform display to a texture using a wgpu shader, and then drawing that texture with a regular RootVG `ImagePrimitive` later. If the waveform is usually static, then this method is far more efficient than re-rendering the waveform in the shader every frame.

## Custom Primitives

RootVG also supports custom primitives rendered using custom pipelines and shaders. This will be especially useful for creating efficient visualizers like spectrometers and oscilloscopes.

The API for this is a bit different than the API for built-in primitives. Take a look at the [custom_primitive example](https://github.com/MeadowlarkDAW/rootvg/blob/main/examples/custom_primitive.rs) for how this works.

Unlike the built-in primitives, custom primitives aren't batch optimized, meaning that each custom primitive instance will need its own draw command. Though in practice this shouldn't be a problem since there is only going to be one or a handful of these custom primitives in a typical plugin/app.

---

# How Primitives are Stored

In order for RootVG to have low CPU overhead, primitives must be cheap to clone and upload to the GPU.

For quad primitives this is simple. They are stored in a packed format that can be directly copied to the GPU's vertex buffer. For example the `SolidQuackPrimitive` is stored like this, which matches the struct in the shader one-to-one:

```rust
#[repr(C)]
#[derive(Clone, Copy, Debug, PartialEq, bytemuck::Pod, bytemuck::Zeroable)]
pub struct SolidQuadPrimitive {
    pub color: PackedSrgb,
    pub position: [f32; 2],
    pub size: [f32; 2],
    pub border_color: PackedSrgb,
    pub border_radius: [f32; 4],
    pub border_width: f32,
    pub shadow_color: PackedSrgb,
    pub shadow_offset: [f32; 2],
    pub shadow_blur_radius: f32,
}
```

The other primitive types rely on some sort of heap-allocated data such as a `Vec` of vertices for meshes, a `Vec` of glyph vertices for text, and a texture for images. These primitive types wrap their heap-allocated part in an `Rc` pointer to allow them to be cheaply cloned and diffed.

For example, a `SolidMeshPrimitive` is stored like this (not exactly this, it's been simplified a bit for this example):

```rust
pub struct SolidMeshPrimitive {
    // This part is copied to the batch's vertex and index buffer.
    pub mesh: Rc<SolidMesh>,

    // This part is copied to the batch's uniform/push constants buffer.
    pub uniform: MeshUniforms,
}

pub struct SolidMesh {
    pub vertices: Vec<SolidVertex2D>,
    pub indices: Vec<u32>,
}

#[repr(C)]
#[derive(Copy, Clone, Debug, PartialEq, bytemuck::Pod, bytemuck::Zeroable)]
pub struct SolidVertex2D {
    pub position: [f32; 2],
    pub color: PackedSrgb,
}

#[repr(C)]
#[derive(Debug, Clone, Copy, PartialEq, bytemuck::Pod, bytemuck::Zeroable)]
pub struct MeshUniforms {
    pub offset: [f32; 2],

    /// A 2d transformation represented by a column-major 3 by 3
    /// matrix, compressed down to 3 by 2.
    pub transform: [f32; 6],

    /// Whether or not to apply the `transform` matrix. This is used
    /// to optimize meshes with no transformations.
    ///
    /// By default this is set to `0` (false).
    pub has_transform: u32,
}

```

Doing this also allows multiple primitives to share the same buffer of data in the GPU. For example, you can have multiple knob widgets share the same index and vertex buffer to an arc mesh, with only offset/rotation/scale being copied to a uniform buffer for each instance. (Though for now I haven't actually made this optimization in RootVG yet. More testing is needed to see if this will actually improve performance in practice.)

---

# How Primitives are Sorted

To improve performance on the GPU side, the number of draw calls should be reduced as much as possible.

For example, say you have many knob widgets in an application. And say that each knob renders multiple parts: an image as the knob's body, a rectangle on top as the "notch" on the knob, an arc around the knob showing its modulation range, and a label of text below it with a background.

If you try to render each knob widget one by one in order, you would be doing a bunch of operations on the GPU:
1. switch to image pipeline
2. draw call to render the image of the knob's body
3. switch to quad pipeline
4. draw call to render the knob's "notch"
5. switch to mesh pipeline
6. draw call to render the arc showing the modulation range
7. switch to quad pipeline
8. draw call to render the quad underneath the label
9. switch to text pipeline
10. draw call to render the text of the label
11. repeat steps 1-10 for every knob widget

Ouch.

Then how about we try rendering all primitives of the same type all at once in a single draw call? The problem is that can make things not be drawn in the order the user expected. Primitives that were supposed to be rendered on top of other primitives could be rendered below them.

The solution is to have the user specify the z index for each primitive in the API. Primitives that have a higher z index are guaranteed to be drawn after ones that have a lower z index. Consequently primitives with the same z index are not guaranteed to be drawn in the same order they were added.

As a bonus, this means that all primitives in the entire app that have the same z index can be batched together, not just for widgets of the same type!

Though there is one caveat. Since this is meant to be used in a GUI library, we also need widgets themselves to have a z index. Some widgets obviously need to be rendered on top of others in the correct order. The solution is to have a "widget z index" that takes the upper part of the final z index and the "internal primitive z index" take the lower part of the final z index. Widgets provide the GUI library with a list of primitives via the [`PrimitiveGroup*`](https://github.com/MeadowlarkDAW/rootvg/blob/main/src/primitive_group.rs) struct, and then later when the GUI library adds it to the canvas, RootVG automatically applies the widget's z index to each primitive's z index (as well as also applying the widget's offset to each primitive).

> * I haven't made an example using the `PrimitiveGroup` yet. I'm going to wait until I've worked out the kinks from using it in my own GUI library first.

```rust
fn z_index(widget_z_index: u16, primitive_z_index: u16) -> u32 {
    (widget_z_index as u32) << 16 | primitive_z_index as u32
}
```

Now how does RootVG actually use the z index to sort primitives? It simply stores batches into a hash map using the z index as the key.

```rust
#[derive(Clone, Copy, PartialEq, Eq, Hash)]
struct BatchKey {
    z_index: u32,
}

struct BatchEntry {
    solid_meshes: Vec<SolidMeshPrimitive>,
    gradient_meshes: Vec<GradientMeshPrimitive>,
    solid_quads: Vec<SolidQuadPrimitive>,
    gradient_quads: Vec<GradientQuadPrimitive>,
    text: Vec<TextPrimitive>,
    images: Vec<ImagePrimitive>,
    // (optional custom primitives) ...
}

pub struct Canvas {
    batches: FxHashMap<BatchKey, BatchEntry>,

    // ...
}
```

When it comes time to render, RootVG collects all the keys in the hashmap into a `Vec`, and then does an unstable quick sort to sort the keys.

```rust
// Collect all keys from the hash map into a Vec.
let mut keys: Vec<BatchKey> = batches.keys().map(|k| *k).collect();

// Sort keys by z index
keys.sort_unstable_by(|a, b| a.z_index.cmp(&b.z_index));
```

Then it simply renders each batch in order.

```rust
for key in keys.iter() {
    if let Some(batch_entry) = batches.get_mut(key) {
        // render batch
    }
}
```

*(Note RootVG does not actually render the batches from the hashmap directly, this is just to explain the concept. It actually first prepares the batch entries into the various pipelines and stores the result of the prepare pass into a `CanvasOutput` struct. The `CanvasOutput` is then what actually gets rendered.)*

---

# Scissoring Rectangles

![scissoring rectangle](/images/rootvg/scissoring-rectangle.png)

The last piece of the puzzle to make RootVG viable for GUIs is the ability to "scissor" parts of primitives that lie outside a given rectangle.

Luckily GPUs have a built-in method for this, all we need to do is call a method in wgpu. The one caveat is this method must be called between draw calls, meaning that we have to split batches into multiple draw calls for each scissoring rect.

In our case this is easy to do, as all we need is to add the scissoring rectangle as part of the key in our hashmap, and it just works.

```rust
#[derive(Clone, Copy, PartialEq, Eq, Hash)]
struct BatchKey {
    scissor_rect: RectI32,
    z_index: u32,
}
```

---

# Conclusion

Could I have made [NanoVG](https://github.com/inniyah/nanovg), [FemtoVG](https://github.com/femtovg/femtovg), [vger](https://github.com/audulus/vger-rs), [Vello](https://github.com/linebender/vello/tree/main), or [Skia](https://skia.org/) work for my GUI library? Honestly, probably.

But since I was already deep in the weeds of GUI, and since a deeper understanding of graphics programming is something I've wanted to learn anyway, I'm glad I did it. I also now have a library that works really nicely with how my GUI library works, meaning I don't need to add extra complexity to my GUI library to optimize rendering.

I also would have never attempted this if awesome libraries like [wgpu](https://wgpu.rs/), [Iced](https://github.com/iced-rs/iced), [glyphon](https://github.com/grovesNL/glyphon), [Cosmic Text](https://github.com/pop-os/cosmic-text) and [Lyon](https://github.com/nical/lyon) haven't already done most of the heavy lifting for me. So special thanks to them!

*And in case you're wondering, no I haven't spent this entire time on just RootVG. I've made substantial progress on my GUI library as well. Stay tuned for that! :)*

