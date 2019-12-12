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
suffice. Pretty soon, however, I got tired of looking at the graphics, so I inexplicably decided
to implement simple versions of shadow mapping[^1] and deferred shading[^2]. Later on, I added
support for FXAA [^3] and a glow effect.

It was important to me to allow turning off each of the rendering effects separately, so that 
development would be possible on my puny little laptop. Each combination of rendering effects needs 
a different shader, so you get a bit of a combinatorial explosion[^4].  At first, I handled this by
manually splicing together shader source fragments, adding a few lines here and there if some effect
was enabled and so on. This worked fine for a while. Then, however,
[@Vollkornaffe](https://github.com/Vollkornaffe) came up with a great idea for a spiral effect,
which would require doing some transformations in the vertex and fragment shader. I loved the idea,
but then it hit me: since the core of the shaders would be different, I would need to once again
implement support for all of the combinations of rendering effects around that!

Assumably, there are industry-proven solutions to this problem, but after some deliberation I came
up with a somewhat overengineered (and yet hacky) method of successively _transforming_ shaders.
This method later turned into the core of Rendology.

# Shader Cores and their Transformations
Rendology defines a `shader::Core<P, I, V>` as consisting of a vertex shader
`shader::VertexCore<P, I, V>` and a fragment shader `shader::FragmentCore<P>`. The type parameters
define data that has to be provided from the CPU side when drawing with the shader:
- `P` is per-draw-call uniform data,
- `I` is per-instance data, and
- `V` is per-vertex data.

The rendology shader types store GLSL shaders in a decomposed form. Both vertex and fragment cores
consist of output variable declarations, a body, and a list of output expression. The fragment core
additionally has input declarations for varying variables coming from the vertex shader. This
decomposed form allows successive transformations to be applied to shaders.

Before we can look at an example, we need to define the data that flows into the shader:
```rust
use nalgebra as na;

struct Params {
    projection_matrix: na::Matrix4<f32>,
    view_matrix: na::Matrix4<f32>,
    light_pos: na::Vector3<f32>,
}

struct Instance {
    instance_matrix: na::Matrix4<f32>,
}

struct Vertex {
    vertex_pos: [f32; 3],
    vertex_normal: [f32; 3],
}

// (Skipping some trait implementations here.)
```
Given these definitions, we can define a simple shader:
```rust
use rendology::shader;

fn scene_core() -> shader::Core<Params, Instance, Vertex> {
    let vertex = shader::VertexCore::empty()
        .with_body(
            "mat4 normal_matrix = transpose(inverse(mat3(instance_matrix)));"
        )
        .with_out(
            shader::defs::V_WORLD_POS,
            "instance_matrix * vec4(vertex_pos, 1)",
        )
        .with_out(
            shader::defs::V_WORLD_NORMAL,
            "normalize(normal_matrix * vertex_normal)",
        )
        .with_out(
            shader::defs::V_POS,
            "projection_matrix * view_matrix * v_world_pos",
        );

    let fragment = shader::FragmentCore::empty()
        .with_out(shader::defs::F_COLOR, "vec4(1, 0, 0, 1)");

    shader::Core { vertex, fragment }
}
```
A `shader::Core` can be compiled into raw GLSL code. Declarations for input
data types (e.g. `Params`) are generated implicitly.

You may notice that this shader is not very _shady_ --- it just outputs flat red colors.
Let's define a shader core transformation, i.e. a function that takes a shader, modifies it in
some way, and produces a new shader. In this case, we will use the `v_world_pos` and
`v_world_normal` outputs of the above shader to calculate diffuse lighting.
```rust
fn diffuse_transform<I, V>(
    core: shader::Core<Params, I, V>,
) -> shader::Core<Params, I, V> {
    let fragment = core.fragment
        .with_in_def(shader::defs::V_WORLD_POS)
        .with_in_def(shader::defs::V_WORLD_NORMAL)
        .with_body(
            "
            float diffuse = max(
                0.0,
                dot(v_world_normal, normalize(light_pos - v_world_pos.xyz))
            );
            "
        )
        .with_out_expr("f_color", "diffuse * f_color");

    shader::Core {
        vertex: core.vertex,
        fragment,
    }
}
```
The fragment shader is transformed such that it now takes `v_world_pos` and `v_world_normal` as
varying input. The given vertex shader is left unmodified. Note that `diffuse_transform` is generic
in the instance data `I` as well as the vertex data `V`; all that is required is that `Params` is
given, so that we have access to `light_pos`. Thus, the same transformation can be applied to
different kinds of shaders.

While this is a simple example, the same principles are applied in the implementation of Rendology
multiple times. A scene shader needs to be defined only once. Depending on the configuration of the
pipeline, the shader then undergoes various transformations, by which support for shadow mapping, 
deferred shading and other effects may be added successively.

# Rendering Pipeline
Rendology's pipeline ensures at compile time that the necessary data for running your scene shader
is given when drawing. One only needs to implement the `SceneCore` trait for the scene shader.
For a full example, see
[`examples/custom_scene_core.rs`](https://github.com/leod/rendology/blob/master/examples/custom_scene_core.rs),
where texturing is implemented. Then, it is possible to create shadow passes, shaded scene passes
and plain scene passes for your `SceneCore` implementation. As an example, the pipeline's `draw` 
function for a shaded scene pass is declared as follows:
```rust
pub fn draw<C, D, P>(
    self,
    pass: &ShadedScenePass<C>,
    drawable: &D,
    params: &P,
    draw_params: &glium::DrawParameters,
) -> Result<Self, DrawError>
where
    C: SceneCore,
    D: Drawable<C::Instance, C::Vertex>,
    P: shader::input::CompatibleWith<C::Params>,
```
Here, `C::Params` are the uniforms for `C`, `C::Instance` is the per-instance data and `C::Vertex`
is the mesh's vertex type. `drawable` holds instances as well as the mesh that is to be drawn.

There is a certain order of operations that must be respected when drawing a frame. Consider for
example the case of using shadow mapping and deferred shading. A typical frame may look like this:
1. Clear buffers.
2. Draw the scene from the light's perspective, creating a shadow texture.
3. Draw the scene from the camera's perspective, creating albedo and normal textures.
4. Calculate light in another texture by making use of the normal texture.
5. Compose by multiplying light and albedo texture.
6. Draw plain and/or translucent objects.
7. Apply postprocessing and present the frame.

Of course, there are many ways of messing up this order; for example, it would not make sense to
create the shadow texture _after_ step five. Rendology enforces this order of operations by defining
a series of types that represent a finite-state automaton. Each operation takes `self` by-move and
returns an instance of a type with the legal follow-up operations. Furthermore, the types are
annotated with `#[must_use]`. Taken together, these definitions ensure that whenever you start a
frame, you will follow a path through the automaton until the result is presented to the user.

# Appendix
The following diagram shows the paths that are currently possible when drawing a frame:
![drawing a frame](/assets/rendology_pipeline.png)

# Footnotes
[^1]: Glium's [shadow mapping example](https://github.com/glium/glium/blob/master/examples/shadow_mapping.rs) was of great help in this.
[^2]: Again, Glium's [deferred shading example](https://github.com/glium/glium/blob/master/examples/deferred.rs) was helpful.
[^3]: Or something resembling FXAA somewhat, hopefully.
[^4]: For example, the shaders for deferred lighting change slightly if you want to support shadow mapping, and then they change again if you want to add the glow effect into the pipeline.
