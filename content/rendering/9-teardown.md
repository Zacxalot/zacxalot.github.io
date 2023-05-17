+++
title = "Teardown Render Techniques"
date = "2023-05-13"
transparent = true

[taxonomies]
categories = ["Rendering"]
tags = ["graphics"]
+++
<script type="module" src="https://unpkg.com/@google/model-viewer/dist/model-viewer.min.js"></script>

I've had a couple months off from the blog, but I'm going to try and get back into it again.

To jump back in, I'm going to try and figure out how the game [Teardown](https://store.steampowered.com/app/1167630/Teardown/) renders so many voxels and makes it look so pretty at the same time!

I'm going to carry on with flat blend at some point, it's in dire need of some restructuring to require less boilerplate when adding render passes. Adding that grid took way more effort than I think it should, sapping my motivation along with it.

Before anything, I just want to make it clear that I'll be doing a good amount of guessing here, and my guesses will be wrong in many cases, but hopefully some guesses will be right 😅.

# Teardown

Teardown is a destruction simulation sandbox game where you warp the environment to complete heists. If you haven't seen any footage of it yet, give some a watch, it's very impressive.

The frame I have captured for analysis doesn't do the games capabilities any justice, but for the sake of figuring out what was going on, I thought it would be best to keep simple. 

{{image(src="final-image.png", alt="The final rendered scene from teardown featuring a broken sign, some background structures, telephone wires, trees and a digger" caption="")}}

## Voxels

The first thing in the rendering job list (after clearing the screen, and doing something [with this texture](wtf.png)!?!?) is some voxels! [Voxels](https://en.wikipedia.org/wiki/Voxel) are basically just images, that are in 3D. Similarly to 2D images though, the data that is stored in them doesn't have to be colour values meant for human consumption. In the case of Teardown, the 3D textures 

<model-viewer 
    src="sign.glb"
    ios-src="sign.glb"
    alt=""
    rotation-per-second="32"
    shadow-intensity="1"
    camera-controls
    auto-rotate ar
    style="width:100%;height: 75vh; border-radius:15px;"
>