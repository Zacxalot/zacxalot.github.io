+++
title = "Renderdoc, Wireframes and Refactoring"
date = "2023-03-01"
transparent = true

[taxonomies]
categories = ["Flat-Blend"]
tags = ["rust","blender","renderdoc","vulkan","refactoring"]
+++

Now that I've got my `BMesh` rendering, I think it would be nice to try and replicate some of the different viewport options Blender is capable of. More specifically, I'd like to implement a wireframe view.
Before jumping straight to how it's done in Blender, I've tried getting it going myself.

With a little bit of Duckduckgo-ing, I bumped into this, [Vulkan Polygon Mode](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkPolygonMode.html). By setting this value to `VK_POLYGON_MODE_LINE` in the pipeline, instead of rendering the whole surface of each triangle, only the edges are drawn. I found the value in the Vulkano docs [here](https://docs.rs/vulkano/latest/vulkano/pipeline/graphics/rasterization/enum.PolygonMode.html) and set it in my pipeline definition like so:

```rust
let rasterization_state = RasterizationState {
    cull_mode: StateMode::Fixed(CullMode::None),
    polygon_mode: PolygonMode::Line,
    ..Default::default()
};
```

Running with this change presented this: 🥳

{{image(src="wireframe.png", alt="A wireframe showing the triangles that make up the square", caption="I wish everything I've done so far on this project was this easy")}}

You may notice a bit of an issue with this, however. Looking at the wireframe of the same mesh in Blender, it is only rendering the edges of BMesh, not the edges of the triangles.

{{image(src="blender-wireframe.png", alt="A wireframe showing the just the BMesh edges of a square in Blender", caption="")}}

This is how we want it, because this is how the `BMesh` data is stored, and this is all the user cares about whilst interacting with it.

## Renderdoc

Again, before looking at how Blender does it, I tried searching the internet some more but practically everything I could find was about using `VK_POLYGON_MODE_LINE`. So I gave in, and loaded up one of my favorite programs, [Renderdoc](https://renderdoc.org/). If you've never come across it, Renderdoc is a graphics debugging tool that lets you see step by step how the GPU has processed a frame.

Renderdoc has an overwhelming interface (looks like Java Swing but it's probably Qt), but I've managed to find all of the information I've needed from it so far. Importantly, I spotted that the pipeline is using the primitive topology of `Line List`, which I didn't even know existed! From here, I can also look at the shader code that produces the wireframe. I had a peek, and it seemed a bit overcomplicated for what I need, so I'll stick with what I've got for now.

{{image(src="renderdoc-1.png", alt="Render doc showing the pipeline state", caption="")}}

The input into the vertex shader in Renderdoc was just a list of vertices, paired up to make a line. To get a line list from my `BMesh`, I used Rust's wonderful iterators to make cheeky function for it:
```rust
pub fn bm_edge_list(bmesh: &mut BMesh) -> Vec<Vertex> {
    unsafe {
        bmesh
            .edges
            .iter()
            .flat_map(|(_, edge)| [(*edge.v0).vertex, (*edge.v1).vertex])
            .collect::<Vec<Vertex>>()
    }
}
```

Then, by setting the topology mode to Line List, and passing in only the vertices we get this: 🥳

{{image(src="bmesh-wireframe.png", alt="A wireframe showing the just the BMesh edges of a square", caption="")}}

## Refactoring

By implementing the new wireframe display, I had to change the pipeline that renders our shapes to remove the input of indices. The `LineList` topology doesn't accept these (Or maybe it does, and I've just not made use of them).
So that has pushed me into doing some refactoring.
I knew it was coming, the base code I was using from the [Vulkano example triangle](https://github.com/vulkano-rs/vulkano/blob/0.32.X/examples/src/bin/triangle-v1_3.rs) was understandably designed around just drawing a triangle.
But that's just not going to cut it anymore!
'
I've probably done a terrible job of this, but without having written anything in Vulkan before, it's fair to accept it won't be even close to perfect on the first try.
I've done two big things really.

Firstly, I've moved all of the initialisation of Vulkan into its own initialisation function, which returns a `VulkanState`. This struct looks like this and will likely be subject to heavy change when I realise my refactoring is bad.

```rust
pub struct VulkanState {
    pub device: Arc<Device>,
    pub surface: Arc<Surface>,
    pub descriptor_set_allocator: StandardDescriptorSetAllocator,
    pub command_buffer_allocator: StandardCommandBufferAllocator,
    pub recreate_swapchain: bool,
    pub previous_frame_end: Option<Box<dyn GpuFuture>>,
    pub queue: Arc<Queue>,
    pub vertex_buffers: VertexBuffers,
    pub index_buffers: IndexBuffers,
    pub shaders: Arc<LoadedShaders>,
    pub swapchain: Arc<Swapchain>,
    pub swapchain_images: Vec<Arc<SwapchainImage>>,
    pub viewport: Viewport,
    pub attachment_images: Arc<AttachmentImageMap>,
    pub memory_allocator: Arc<GenericMemoryAllocator<Arc<FreeListAllocator>>>,
    pub render_passes: Arc<RenderPasses>,
    pub pipelines: Arc<Pipelines>,
    pub frame_buffers: Arc<FrameBufferMap>,
    pub uniform_buffer: Arc<CpuBufferPool<Data>>,
}
```
`Arc`'s for days!
I think this is pretty cool though, instead of having to pass everything around individually we can now pass in an `Arc` to the `VulkanState` and let the function grab what it needs.

The second thing I did was to move the creation of shaders, render passes, pipelines, frame buffers and attachment images into their own files and have loaders for each that return an [EnumMap](https://crates.io/crates/enum-map) for each.
This makes keeping track of what what each resource is tied to and for really easy.

I also experimented with wrapping the values of the `EnumMap`'s with `Option` for the `VertexBuffers` and `IndexBuffers`.
This is important for these buffers as making an empty vertex buffer is apparently very naughty (`'main' panicked at 'assertion failed: size != 0'`), so unless we have some vertices ready for each of our vertex buffers when creating our Vulkan environment, we can't make them.

Here is the type definition and initialisation of the vertex buffers:
```rust
// buffers.rs, terrible name
pub type VertexBuffers = EnumMap<VertexBufferKey, Option<Arc<CpuAccessibleBuffer<[Vertex]>>>>;

//init.rs
vertex_buffers: enum_map! {_ => None}
```

And here is the render pass definition `EnumMap`. The rest of the `EnumMap`'s are similar to this.
```rust
// render_pass_loader.rs
#[derive(Enum)]
pub enum RenderPassKeys {
    Solid,
}

pub type RenderPasses = EnumMap<RenderPassKeys, Arc<RenderPass>>;

pub fn load_render_passes(device: Arc<Device>, format: Format) -> Arc<RenderPasses> {
    Arc::new(enum_map! {
        RenderPassKeys::Solid => solid_draw_pass(device.clone(), format).unwrap()
    })
}
```

To top it all off, I've cleared all of the warnings ⚠️🥳

Doing all of this refactoring took hours and I'm glad it's done for now. Plenty of room for progress now!

Next up, a grid!

---

I'm not all that into `Radiohead` but I like `Sit Down. Stand Up`, it's just 3 minutes of buildup and then Thom Yorke expressing peak lyricism.
{{spotify(src="https://open.spotify.com/embed/track/6MKWCO8g2W05UcaFyfQ6Cl?utm_source=generator")}}
