+++
title = "Mesh Data Structures?"
date = "2023-01-28"
transparent = true

[taxonomies]
categories = ["Flat-Blend"]
tags = ["rust","graphics"]
+++

At the moment, we're generating vertices using the [Lyon](https://lib.rs/crates/lyon) crate and sticking them right into a vertex buffer for Vulkan to use.

```rust
let mut geometry: VertexBuffers<Point, u16> = VertexBuffers::new();
let mut geometry_builder = simple_builder(&mut geometry);

let options = FillOptions::tolerance(0.001);
let mut tessellator = FillTessellator::new();

let mut builder = tessellator.builder(&options, &mut geometry_builder);

builder.add_circle([0.0, 0.0].into(), 1.0, Winding::Positive);

builder.build().unwrap();
```

The geometry generation is fine, I plan to use Lyon for this purpose for as long as I can. I'm still thinking about it, but I think my problem is how do I store this data for several meshes and how do I edit the vertex data while still keeping track of where faces are.

In seek of answers, I turned to the Blender development wiki and found [this page](https://wiki.blender.org/wiki/Source/Objects/Mesh), which reveals that Blender uses to methods of structuring mesh data. `Mesh`, which is how meshes are stored when they're not being modified, and `BMesh` which is how they're stored when they are being modified. I'm not sure if either of these are relevant for what I'm trying to achieve, or if I need a similarly separated system for static and editable mesh data, so I'm going to look into both and see.

# A few hours later...
This is a few hours later and my search hasn't gone too well. From what I could find, the Blender documentation for `Mesh` starts and ends on the page I just linked. There's not a lot to go off here, but it does mention that the Mesh structure is based on struct of arrays. This page also links off to a ticket [here](https://developer.blender.org/T95965) which reveals a bit more about what is stored. For `BMesh` there's a [separate page](https://wiki.blender.org/wiki/Source/Modeling/BMesh/Design#Mesh_Editing_API) with a lot more to it, but it assumes a lot of understanding that I just don't have.

So I'm kind of back to square one with this. I'm just going to implement how I think it would work and make the mistakes I was hoping to avoid by doing it the tried and tested way.