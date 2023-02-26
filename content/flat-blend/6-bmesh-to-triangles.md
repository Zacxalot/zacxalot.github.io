+++
title = "Rendering BMesh"
date = "2023-02-26"
transparent = true

[taxonomies]
categories = ["Flat-Blend"]
tags = ["rust","bmesh","blender"]
+++

Another short post from me this week. I've made some progress on getting `BMesh` converted into a list of vertices and triangles for Vulkan to eat up.

Here is the function I've made for it.
``` rust
pub fn bm_triangulate(bmesh: &mut BMesh) -> (Vec<Vertex>, Vec<u32>) {
    let mut all_bm_vertices: Vec<*mut BMVert> = vec![];
    let mut all_indices: Vec<u32> = vec![];

    for (_, face) in &bmesh.faces {
        unsafe {
            let vertices = BMLoopIterator::new(face.loop_start.unwrap())
                .map(|l| (*l).vertex)
                .collect::<Vec<*mut BMVert>>();

            let flattened_verts = vertices
                .iter()
                .flat_map(|v| (**v).vertex.position)
                .collect::<Vec<f32>>();

            let indices = earcutr::earcut(&flattened_verts, &[], 2).unwrap();

            for index in indices {
                if let Some(position) = all_bm_vertices
                    .iter()
                    .position(|val| val == &vertices[index])
                {
                    all_indices.push(position as u32);
                } else {
                    all_bm_vertices.push(vertices[index]);
                    all_indices.push((all_bm_vertices.len() - 1) as u32);
                }
            }
        }
    }

    unsafe {
        let all_vertices = all_bm_vertices
            .iter()
            .map(|v| (*(*v)).vertex)
            .collect::<Vec<Vertex>>();
        (all_vertices, all_indices)
    }
}
```

Per face (not always a triangle), it runs the ear cutting algorithm from the [earcutr](https://crates.io/crates/earcutr).
It then combines the results of all of these into a `Vec` of `Vertex`'s and a `Vec` of indices.
These can then be put directly into Vertex and Index buffers.

I had a look through the Blender code to try and find out how they do it, but I couldn't find it. I did however see on the [triangulate modifier docs](https://docs.blender.org/manual/en/latest/modeling/modifiers/generate/triangulate.html) that the clip method uses the ear clipping algorithm, which it mentions "gives similar results to the tessellation used for the viewport rendering". Perfect!

Thankfully the `earcutr` crate exists so I don't have to re-invent any wheels, but I had a look around for how the algorithm works anyway. I found this very [old but endearing website](https://www.personal.kent.edu/~rmuhamma/Compgeometry/MyCG/TwoEar/two-ear.htm) which explains a bit, the [github repo for earcutr](https://github.com/donbright/earcutr) which have some good explainations and fun ascii diagrams, and [this page here](https://twohiccups.github.io/Ear-Clipping/) which provides a lovely visualisation of what's going on. I'll be honest, I'm still not 100% sure how it works, but it does, and that's all I need.


Here is the fruit of my labour, another square 🙃. I've changed the colours to be a little more on brand at least.
{{image(src="screen.png", alt="A pink square in a window", caption="")}}

## BMesh in Motion

To make it feel like I've actually done something, I have made it so I can change the position of the bottom left vertex using the mouse.
I've probably done this in a way that is a crime against Vulkan, but I can worry about a proper implementation later. Here it is:

```rust
let window = surface.object().unwrap().downcast_ref::<Window>().unwrap();
let width = window.inner_size().width as f64;
let height = window.inner_size().height as f64;

let rel_x = 6.0 * ((position.x / width) * 2.0 - 1.0);
let rel_y = 6.0 * (1.0 - (position.y / height) * 2.0);

square_mesh
    .vertices
    .iter_mut()
    .next()
    .unwrap()
    .1
    .vertex
    .position = [rel_x as f32, rel_y as f32];

let (vertices, indices) = bm_triangulate(&mut square_mesh);

vertex_buffer = CpuAccessibleBuffer::from_iter(
    &memory_allocator,
    BufferUsage {
        vertex_buffer: true,
        ..BufferUsage::empty()
    },
    false,
    vertices,
)
.unwrap();

index_buffer = CpuAccessibleBuffer::from_iter(
    &memory_allocator,
    BufferUsage {
        index_buffer: true,
        ..BufferUsage::empty()
    },
    false,
    indices,
)
.unwrap();
```

Got some magic numbers in there (6.0) to scale it to be almost right 🤮.

This at least gave me a chance to look at the [winit](https://crates.io/crates/winit) crate's event handling. It's pretty cool!
Everything is in Enums so it's really nice to work with. Getting the position of the cursor after it's been moved is as simple as this:

```rust
Event::WindowEvent {
    event: WindowEvent::CursorMoved { position, .. },
    ..
} => {
    println!("{:?}", position)
}
```

That's all for now, here's a short video of the moving vertex 👀.
{{video(src="video.webm", alt="A pink square in a window with it's bottom left vertex moving")}}

---



I've been really getting into `Faith No More` recently. `Falling To Pieces` is one of my favourites. If you don't like this, listen to anything else off `The Real Thing`, there's some top stuff in there.
{{spotify(src="https://open.spotify.com/embed/track/20nb0Wl1yqoEERbUSILuG1?utm_source=generator")}}