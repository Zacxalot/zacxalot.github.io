+++
title = "Getting Started With Vulkan"
date = "2023-01-26"
transparent = true

[taxonomies]
categories = ["Flat-Blend"]
tags = ["rust","graphics"]
+++

Before I get into anything else, I just want to make it clear that this is my first blog post and the first time since my dissertation that I've written anything longer than a few sentences, so go easy on me.

# The Project - Flat-Blend
Every time I use any vector editing software, I am frustrated by the limitations of what it can do compared to Blender. So, I’m trying to make a spinoff of Blender, but it’s 2D and it’s written in Rust.

Key things I want out of this:
- Modifiers: Arrays, mirroring, skinning, boolean, all of the fun stuff that would apply to 2D
- Edit Mode: No messing about with different coloured cursors, select an object, edit it, put it back
- Shortcuts & Controls: I'm a big fan of how Blender controls, big bonus for me
- SVG Export

For a first goal, I can’t tell if these are too adventurous or not. The biggest challenge I’m expecting to face is working with Vulkan. I’ve used a little bit of OpenGL in the past but I’m having a go of Vulkan this time around and it has a bit of a steeper learning curve. From what I’ve heard though, the difficulty if offset by the room for efficiency that Vulkan offers. 

Besides the goals of the end product, I have some meta goals to go along-side:
-	Large Scale Project: Besides my final year university project, I have never done anything personal with objectives as large as this before, so I’d like to “finish” something finally
-	Writing a Blog: I’ve always liked the idea of a blog, even if nobody reads it I feel it’s a nice way of reflecting on what you have achieved and looking at what you can do next. I also plan for this blog to be an encouragement to myself to carry on with the project to completion
-	Learning Vulkan: My job title is 3D Software Engineer, so to try and edge myself away from the ever tightening grasp of imposter syndrome, I’d like to finally learn a graphics API through and through
 
# Code
As I've mentioned, I intend to build this project using Rust. Besides me just enjoying using it, it seems like a good fit for what I want to do. It's capable of great performance and has a super handy wrapper around Vulkan called [Vulkano](https://lib.rs/crates/vulkano)

I jumped into startng a couple weeks ago and got some basic stuff going. Really just got a rendering pipeline in there at the moment, managed to render a square and a circle with orthographic projection. I'm not going into to much detail on what code I've got just yet, I'll save that for a later post. I can however [give you a link to the repo](https://github.com/Zacxalot/flat-blend/) and you can look yourself. It's a mess in there at time of writing, don't say I didn't warn you.

Here are some screenshots!
{{image(src="square.png", alt="Square rendered in a window, rotated 45 degrees", caption="Thought I'd mix it up and render a square rather than a triangle")}}
{{image(src="circle.png", alt="Circle rendered in a window", caption="72 vertices worth of smooth circly goodness")}}


# Summary
tl;dr I'm making blender but flat.

This might be kind of short in terms of blogs, but it's been a long day and I don't really know what I'm doing 😅. Thanks for reading though!