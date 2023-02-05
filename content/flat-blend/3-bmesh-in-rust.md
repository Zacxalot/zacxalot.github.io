+++
title = "BMesh in Rust"
date = "2023-02-05"
transparent = true

[taxonomies]
categories = ["Flat-Blend"]
tags = ["rust","bmesh","blender"]
+++

I'll start with trying to answer some of the questions I left off with in my last post.
- Do I need to implement this myself or is there a library that will do what I need

I found a package that might be heading in the right direction for what I need called [hedron](https://crates.io/crates/hedron), but the developer has stated that it's in "Pre Alpha" and a lot of the functionality I'd like to have such as boolean operations and subdivisions are marked as (TODO). Still, a very cool project! Has Bevy integration too!

There were a few other packages that I found but they were all quite old/not flexible enough.

---

- Is this even what I need, or is it too advanced/3D specific

Code now, ask questions later! I'm sure that won't come back to haunt me 😅

I don't care I'm having fun.

---

- How do I implement this in safe Rust, there's a lot of pointer usage going on here and I feel there might be some confusing lifetimes going on in there too

This is where it gets a bit rough. I've been hurting my brain trying to implement just the data structure in safe Rust and I've found it to be extremely difficult, at least when trying to translate from C++. The problem is the one I foresaw, a crazy amount of pointer usage, and those pointers go back and forth too. As you would expect, a `BMEdge` has a pointer to two vertices. But if vertex hasn't got any edges attached to it yet, the vertex gets a pointer to the edge that was just created.

This bi-directional referencing is used throught the BMesh structure (Diagram from the Blender wiki [here](https://wiki.blender.org/w/images/0/06/Dev-BMesh-Structures.png), bit of a hectic image), I believe it's integral for making the data easily traversable. 

I had a look at the source code for `LinkedList` in the Rust standard library for some potential hints on how to make something like this safe, but `LinkedList` isn't safe either.

```rust
// linked_list.rs Rust standard library

pub struct LinkedList<T> {
    head: Option<NonNull<Node<T>>>,
    tail: Option<NonNull<Node<T>>>,
    len: usize,
    marker: PhantomData<Box<Node<T>>>,
}

struct Node<T> {
    next: Option<NonNull<Node<T>>>,
    prev: Option<NonNull<Node<T>>>,
    element: T,
}
```

There's still a lot that I'm not sure about here. I think `PhantomData<Box<Node<T>>>` is there to tell the compiler that the struct owns some values of type `T`, ever though it doesn't really, it just owns some pointers. I'm not sure on that though, if that is what it's doing, I don't know why we need to do it. There's also the usage of `NotNull` for the pointers to Node. From what I can tell, it is just a wrapper around this: `pointer: *const T`, why we need that I have no idea. Both of these are covered by a nifty Rust book [Learning Rust With Entirely Too Many Linked Lists](https://rust-unofficial.github.io/too-many-lists/sixth-variance.html), which I've read a bit of but clearly I have not fully understood (I'm going to have another go later).

## Implemenatations
So! Over to my probably poor implementation of something similar. Similar in the sense that it's using raw pointers anyway.

```rust
// bm_vert.rs
pub type PBMVert = *mut ManuallyDrop<BMVert>;

#[derive(Debug)]
pub struct BMVert {
    pub edge: Option<PBMEdge>,
    pub vertex: Vertex,
}

// bm_edge.rs
pub type PBMEdge = *mut ManuallyDrop<BMEdge>;

pub struct BMEdge {
    pub v0: PBMVert,
    pub v1: PBMVert,
    pub r#loop: PBMLoop,
    pub v0_disk_link: BMDiskLink,
    pub v1_disk_link: BMDiskLink,
}
```

I've also implemented the vert and edge create Euler operations mentioned in the [BMesh design page ](https://wiki.blender.org/wiki/Source/Modeling/BMesh/Design#Low-level_API).

```rust
// bm_vert.rs
pub fn bm_vert_create(_bmesh: &mut BMesh) -> ManuallyDrop<BMVert> {
    ManuallyDrop::new(BMVert::from((0.0, 0.0)))
}

// bm_edge.rs
pub fn bm_edge_create(_bmesh: &mut BMesh, v0: PBMVert, v1: PBMVert) -> ManuallyDrop<BMEdge> {
    let mut e = ManuallyDrop::new(BMEdge {
        v0,
        v1,
        r#loop: null_mut(),
        v0_disk_link: BMDiskLink::new(),
        v1_disk_link: BMDiskLink::new(),
    });

    bmesh_disk_edge_append(&mut e, v0);
    bmesh_disk_edge_append(&mut e, v1);

    e
}
```

Comparing the two, there are a few things I'm doing differently and probably completely wrong. I'm using `ManuallyDrop` everywhere, this is because the BMesh struct we make doesn't actually own any of the Vertices, Edges, Faces... that it is made up of. So if we didn't stop it from dropping the values we create, we could get into a situation where we have dangling pointers. At least I think that's what I'm doing by wrapping everything in `ManuallyDrop`.

I implemented Drop on `BMVert` to see when its drop was being called and it stopped calling that as soon as I wrapped it in `ManuallyDrop` so I just went with it. Without it, the dangling pointers still pointed to the right data, but I think this data no longer had the guaratee of being what we expect it to be. Scary stuff!

So, I think this `ManuallyDrop` stuff is fine, but maybe not the best solution for this problem. It might be solved by using `PhantomData<Box<Node<T>>>`. Or maybe I do need it, I don't know. The Blender code uses `mempool`'s which look complicated too.

Another thing I'm doing differently is I'm using `*mut` rather than `NonNull`. Only because I don't know how `NonNull` works or why it's useful yet.

Finally, I noticed that I'm not using Box anywhere, so I am guessing everything I've been making so far has been going on the stack. 

## Difficult but fun
I really enjoy working with Rust and this is a side of it that I've never needed to touch on before. Knowing that standard library stuff is also unsafe makes me feel less nauseous about using it but I'm still well aware that there's a lot of pitfalls for me to jump into. I've got a lot left to learn but I'm getting there. I'm going to go and read some stuff and probably rewrite everything.

Thanks for reading!

---

I'm a bit too young to have experienced the music of the 90's, and I'm kind of glad because most of it sounds terrible compared to its bookending decades.
There's some alright stuff in there though, I just found `Come on over Baby (All I Want Is You) by Christina Aguilera`.
Give this a listen it's pretty sweet.

{{spotify(src="https://open.spotify.com/embed/track/4L8AtXJFgtX5E3Hr172uIg?utm_source=generator&theme=0")}}

I tried embedding the proper album version, but Spotify's embed functionality links you to the wrong song if it's a multi disker apparently 🙄.