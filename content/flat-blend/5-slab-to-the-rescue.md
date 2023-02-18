+++
title = "Slab to the rescue!"
date = "2023-02-14"
transparent = true

[taxonomies]
categories = ["Flat-Blend"]
tags = ["rust","bmesh","blender"]
+++

I'm back onto my flat-blend project and have continued implementing `BMesh` in Rust.
There's some stuff I'm still yet to understand the purpose of, `PhantomData` being a big one, but there's also quite a bit I think I've got right, for now.

I've gotten rid of all of the `ManuallyDrop` wrappers, which I had a feeling were the wrong thing to be using at the time I was writing them but I knew no better. Instead I'm using an allocator called [Slab](https://crates.io/crates/slab). With this you can create data of a single type, have raw pointers to the data without it moving and destroy the data when you're done with it.

So now my `BMesh` struct actually has ownership of the verts, edges, faces and loops, which is the way I think it should have been from the start.
```rust
pub struct BMesh {
    pub vertices: Slab<BMVert>,
    pub edges: Slab<BMEdge>,
    pub loops: Slab<BMLoop>,
    pub faces: Slab<BMFace>,
}
```

Here's how we create and kill verts now too.

```rust
pub fn bm_vert_create(bmesh: &mut BMesh) -> *mut BMVert {
    let v_index = bmesh.vertices.insert(BMVert::from((0.0, 0.0)));
    let v = bmesh.vertices.get_mut(v_index).unwrap();
    v.slab_index = v_index;

    v
}

pub fn bm_vert_kill(bmesh: &mut BMesh, vert: *mut BMVert) {
    unsafe {
        while let Some(edge) = (*vert).edge {
            bm_edge_kill(bmesh, edge);
        }

        bm_kill_only_vert(bmesh, vert);
    }
}

pub fn bm_kill_only_vert(bmesh: &mut BMesh, vert: *mut BMVert) {
    unsafe {
        bmesh.vertices.remove((*vert).slab_index);
    }
}
```

There's a couple of awkward things you might notice here. When creating a vert we insert it, but right after we get it out as mut to add its index to itself. This feels a bit icky to me but it's the only solution I could think of. Slab doesn't allow you to remove objects based on their raw pointer, which is the way I would have preferred. You'll also notice inside `bm_kill_only_vert`, I've had to wrap the dereference in brackets to be able to get `slab_index` out, this is kind of ugly to me but I think it's the only way.

Before settling on Slab, I did have a look around at some arena allocators. The two I tried out were [typed-arena](https://crates.io/crates/typed-arena) and [generational-arena](https://crates.io/crates/generational-arena). Typed-arena seems really cool, apparently very quick allocation, but the killer for me was that it doesn't allow the deletion of individual objects. If I understand this correctly, it would mean every mesh being edited would be a very slow memory leak. That might be a bit dramatic, I'm sure there would be ways of managing this behaviour so it isn't a problem.

Generational-arena seemed a bit more fitting to what I need. It allows the removal of objects and hands out `Index` objects which gives you a safe way of accessing the data you have inserted (Would also be a solution to my `slab_index` gripe). `Index` is quite annoying to work with though, having to have everything return a `Result` and use `arena.get_mut(vert_0)?` all the time rather than `(*vert_0)` would start to frustrate me after a while. If I was making BMesh from scratch, I think it would make sense to use `Result`'s, but I know the C++ implementation works, so I might as well take the performance benefit of raw pointers and keep rolling with it.
## Faces and loops

Besides switching to Slab, I have also implemented the create and kill functions for `BMFace` and `BMLoop` as well as the `disk link` creation for `BMEdge`. I'm getting into the swing of translating from C++ to Rust now so this didn't take too long. It also helps that my Slabs seem similar to memory pools in Blender, so there's even less for me to think about.

## Euler Operators

If you look at the [BMesh design page](https://wiki.blender.org/wiki/Source/Modeling/BMesh/Design) it has a list of the Euler operators that the higher level mesh editing functions use to do everything. So far, I've implemented these 6:
- bm_vert_create/kill
- bm_edge_create/kill
- bm_face_create/kill

There are a few more left to implement that the page lists as
- bmesh_kernel_split_edge_make_vert/join_edge_kill_vert
- bmesh_kernel_split_face_make_edge/join_face_kill_edge: Split Face, Make Edge and Join Face, Kill Edge
- bmesh_loop_reverse: Reverse the loop of a BMFace. Its own inverse

Before I jump in and implement these though, I'd like to bring things back to flat-blend and try to render an mesh made with BMesh. So, I need to have another dive into the Blender source code to try and find where/how it turns these BMesh's (and n-gons 😬) into something a GPU can consume. 

---

I listened to this album some time last year, great all round but this track is my favourite off it. Gets bonus points for having a wireframe sphere on the art.
{{spotify(src="https://open.spotify.com/embed/track/5XMJHuIuP9L8j2NEiEWRla?utm_source=generator")}}