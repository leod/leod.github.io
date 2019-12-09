---
layout: post
title:  "Introduction to Rendology"
date:   2019-12-09 13:05:53 +0100
categories: rust gamedev rendering rendology 
---

[_Rendology_](https://github.com/leod/rendology) is a 3D rendering pipeline based on
[Glium](https://github.com/glium/glium) and written in Rust. It features basic implementations of
[shadow mapping](https://learnopengl.com/Advanced-Lighting/Shadows/Shadow-Mapping),
[deferred shading](https://learnopengl.com/Advanced-Lighting/Deferred-Shading),
[a glow effect](https://learnopengl.com/Advanced-Lighting/Bloom),
[FXAA](https://en.wikipedia.org/wiki/Fast_approximate_anti-aliasing)
and instanced rendering. In this blog post, I'll outline some of the concepts of Rendology and
describe how they came to be this way. 

Note that this is written from the perspective of an amateur game developer, so take everything
with two grains of salt. Also, please keep in mind that Rendology is unstable, undocumented and not 
ready for general usage yet.

# Background
Rendology was split off from my puzzle game project
[_Ultimate Scale_](https://github.com/leod/ultimate-scale). When I started this project, I wanted
to focus on the game concept --- certainly, I thought, drawing some plain cubes would more than
suffice. Pretty soon, however, I got tired of looking at the graphics, so I implemented simple
verions of shadow mapping[^1] and deferred shading[^2]. Later on, I added support for FXAA [^3] and
a glow effect.

It was important to me that all of these rendering effects could be turned off separately, so that
development would be possible on my puny little laptop. Each combination of rendering effects needs
a different shader, so you get a bit of a combinatorial explosion[^4].  At first, I handled this 
by manually splicing together shader source fragments, adding a few lines here and there if some
effect was enabled and so on. This worked fine for a while. Then, however,
[@Vollkornaffe](https://github.com/Vollkornaffe) came up with a great idea for a spiral effect,
which would require doing some transformations in the vertex and fragment shader. I loved the idea,
but then it hit me: since the core of the shaders would be different, I would need to once again
implement support for all of the combinations of rendering effects around that!

Assumably, there are industry-proven solutions to this problem, but after some deliberation I came
up with a somewhat overengineered (and yet hacky) method of successively _transforming_ shaders.
This method later turned into the core of Rendology.

# Shader Cores and their Transformations

# Footnotes
[^1]: Glium's [shadow mapping example](https://github.com/glium/glium/blob/master/examples/shadow_mapping.rs) was of great help in this.
[^2]: Again, Glium's [deferred shading example](https://github.com/glium/glium/blob/master/examples/deferred.rs) was helpful.
[^3]: Or something resembling FXAA somewhat, hopefully.
[^4]: For example, the shaders for deferred lighting change slightly if you want to support shadow mapping, and then they change again if you want to add the glow effect into the pipeline.
