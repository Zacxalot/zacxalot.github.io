+++
title = "⌗⌗⌗⌗⌗⌗⌗⌗ Grids ⌗⌗⌗⌗⌗⌗⌗⌗"
date = "2023-03-16"
transparent = true

[taxonomies]
categories = ["Flat-Blend"]
tags = ["rust","blender","shadered","vulkan","shaders"]
+++

Blender has a grid, I want a grid!
Blender's grid shader is defined in `overlay_grid_frag.glsl`.
Whoever made this shader kindly left an explanation of what the code is achieving and a little bit about how it does it at the top, as well as a link to a chapter in [Nvidia's GPU Gems 2](https://developer.nvidia.com/gpugems/gpugems2/part-iii-high-quality-rendering/chapter-22-fast-prefiltered-lines) showing a different way to make pre-filtered lines. I had a read of both of these, as well as the shader code, and I was completely stumped!

So I decided to have a go of implementing it myself from scratch. Before messing about trying to add a new pipeline to the flat-blend code, I downloaded [SHADERed](https://shadered.org/) to let me iterate on my code quicker. SHADERed is like the desktop version of [Shadertoy](https://www.shadertoy.com/). I really don't like the code editor (weird auto brackets, no comment line shortcut and other annoyances), but the rest is extremely useful for trying different things out. There is a VSCode extension for it but that was annoying in a different way 🙃. 

The first part I got working was the large grid squares. I wrote this code to get whether a pixel was part of a grid or not. 
```glsl
float getGrid(vec2 uv,int size) {
	vec2 grid = mod((uv - (uResolution / 2)) - 0.5,size);
	return 1.0 - (clamp(min(grid.x, grid.y), 1.0, 2.0) - 1.0);
}
```
I'm not too proud of it as it seems a bit overcomplicated, but I'm new to writing shaders so it'll probably one I come back to in the future. I'll try and break it down though.

`(uv - (uResolution / 2)) - 0.5` gives us the coordinates of each pixel relative to the centre of the screen which, when projected onto the red and green output channels, looks like this:
{{image(src="centre.png", alt="A viewport divided into four sections, green, yellow, black and red", caption="")}}

Then we get the mod of this like so `mod((uv - (uResolution / 2)) - 0.5,size)`, which does a kind of sawtooth pass over our input, splitting it into squares of `size` width and height, which results in this:
{{image(src="big_grid.png", alt="A viewport divided into yellow squares", caption="")}}

You can see from this there are red lines going across the screen and green lines going up. To extract the grid from this we run this `1.0 - (clamp(min(grid.x, grid.y), 1.0, 2.0) - 1.0)` over it. First we get the `min(grid., grid.y)` which is where there is only red or only green (all of the yellow is where there is both), then clamp it between `1.0` and `2.0`. We then subtract `1.0` from this to make the actual range of values be between `0.0` and `1.0` (This seems a bit convoluted but looked better than other things I tried). Finally, we do `1.0 - black_grid` to get a white grid with a black background:
{{image(src="big.png", alt="A white grid on a black background", caption="")}}

I then run this same function with a size parameter half that of the larger grid to get a grid with squares half of the size. These are then combined like so `vec3 grid = vec3(max(big / 2, small / 8))` to produce this:
{{image(src="big_and_small.png", alt="A white grid with smaller less bold squares on a black background", caption="")}}

Now it's time for the axis! This is where I got stuck trying to re-use my `getGrid` function in a similar way to how Blender does it, but my implementation was too different, so I ended up making a new `getAxis` method.

```glsl
float getAxis(vec2 uv, int axis) {
	float line = abs(((uv[axis] + 0.5) - (uResolution[axis]/2))/5);
	return clamp(1.0 - line, 0.0, 1.0);
}
```

This operates on a similar principal to the get grid but operates on only one axis at a time, and instead of performing a `mod`, the output is divided and `abs`'ed to give a gradient along the axis like this:
{{image(src="y_axis.png", alt="A white line along the y axis", caption="")}}

If we run this for the other axis, and assign each to their respective colour channels like so `vec3 axis = vec3(xAxis, yAxis, 0.0);`, we get our axis!
{{image(src="axis.png", alt="Red and green x and y axis", caption="")}}

Then, all we do is combine that with the grid we had from before to get out final output 🥳
{{image(src="complete_grid.png", alt="A grid with all previously described elements on it", caption="")}}

Here is the final fragment shader
```glsl
#version 330

uniform vec2 uResolution;
out vec4 outColor;

int squareSize = 160;
int smallSquareSize = squareSize / 2;

float getGrid(vec2 uv,int size) {
	vec2 grid = mod((uv - (uResolution / 2)) - 0.5,size);
	return 1.0 - (clamp(min(grid.x, grid.y), 1.0, 2.0) - 1.0);
}

float getAxis(vec2 uv, int axis) {
	float line = abs(((uv[axis] + 0.5) - (uResolution[axis]/2))/4);
	return clamp(1.0 - line, 0.0, 1.0);
}

void main() {	
	float big = getGrid(gl_FragCoord.xy, squareSize);
	float small = getGrid(gl_FragCoord.xy, smallSquareSize);
	float xAxis = getAxis(gl_FragCoord.xy, 1);
	float yAxis = getAxis(gl_FragCoord.xy, 0);

	vec3 axis = vec3(xAxis, yAxis, 0.0);
	vec3 grid = vec3(max(big / 2, small / 8));

	vec3 gridCol = vec3(grid);

	float mask = max(axis.x, axis.y);

	outColor = vec4(mix(grid , axis, mask), 0.0);
}
```

## Implementing on flat-blend

Thanks to all of the refactoring I had done previously, adding in another render pass was pretty easy! Everything was pretty much a copy of the other render pass, besides using these vertices to cover the whole screen (and having nothing to do in the vertex shader).

```rust
let grid_vertices: Vec<Vertex> = vec![
    Vertex {
        position: [-1.0, -1.0],
    },
    Vertex {
        position: [1.0, -1.0],
    },
    Vertex {
        position: [-1.0, 1.0],
    },
    Vertex {
        position: [1.0, -1.0],
    },
    Vertex {
        position: [-1.0, 1.0],
    },
    Vertex {
        position: [1.0, 1.0],
    },
];
```

The other render pass also had to be changed to write over the grid, but all that took was changing a value from `Clear` to `Load`.

With these changes, we end up with our pink square on a grid 💃
{{image(src="flat-blend.png", alt="The same grid as before, now with a pink square on it", caption="")}}

This reveals a few issues:
1. The grid size doesn't mean anything in comparison to the size of the square because the square is being scaled but the size of the grid is fixed.
2. There's no way to shift the position of the "camera" in relation to the grid or the square.
3. The grid is also off centre anyway because I've not added a uniform to input the viewport resolution.

These are all things that need to be addressed. But I think the first I want to take care of is the camera positioning, which means it's time to start putting our meshes into "objects" 👻. That's for another time though, this has already taken ages for me to figure out!

---
`Movin' Out` by `Billy Joel` is today's song. My spreadsheets tell my I'll be able to afford a house by mid-2025, so I'll have to wait a bit to share the feeling.

{{spotify(src="https://open.spotify.com/embed/track/16GUMo6u3D2qo9a19AkYct?utm_source=generator")}}
