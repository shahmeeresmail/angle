# Multiview on ANGLE

The goal of multiview is to provide a fast rendering path for simultaneous rendering of geometry 
to multiple targets or regions. This has the potential to improve performance on VR and cube map
rendering scenarios.

At the moment, the focus is on VR rendering scenarios but we would like to leave the door open
to support cube map rendering and other scenarios in the future.

On desktop GL, the multiview extension was used to render to Texture Arrays. However, on WebGL and
WebVR the left and right images could exist in Texture Array or in a Side-by-Side (SBS) swap chains 
so support for side-by-side rendering needs to be added.

Since the WebVR spec is still evolving as is the Multiview spec, this is just a prototype for proof
of concept. As the WebGL_Multiview and WebVR specs evolve, this implementation should as well if it
is to eventually make its way to the master ANGLE branch.

### Level of WebGL_Multiview (prototype) support for different rendering modes

| Output Resource  |  Direct3D 9   |  Direct3D 11     |   Desktop GL   |    GL ES      |    Vulkan     |
|------------------|:-------------:|:----------------:|:--------------:|:-------------:|:-------------:|
| TextureArray     | not started   |  in progress     |  not started   |  not started  |  not started  |
| Side-by-side     | not started   |    complete      |  not started   |  not started  |  not started  |

### Multiview Rendering Modes

| Render Mode               |   API Required             |   Status          |   Performance  |
|--------------------------:|:--------------------------:|:-----------------:|:--------------:|
| Draw-Adjust-Draw          | Any                        | Planned for FL93  |   good         |
| Geometry Shader Redirect  | D3D11 FL10+                | Implemented       |   better       |
| Vertex Shader Redirect    | D3D11.3 /w VPRTfromVS Cap  | Implemented       |   best         |


#### Draw-Adjust-Draw
For each draw, loop through all views and in each iteration:
- Set the RTV or Viewport/Scissor for the view
- Set a uniform indicating the view ID
- Issue the draw

#### Geometry Shader Redirect
For each draw:
- Multiply the instance count by the number of views. If it isn't an instanced draw already, turn
it into one and make the instance count equal to the number of views
- In the VS preamble, set 
    <p><code>gl_InstanceID = input.gl_InstanceID / numViews;<br>
    gl_ViewID = input.gl_InstanceID % numViews;</code></p>
- Pass gl_ViewID to the GS stage
- Create a GS which passes through all VS attributes and outputs gl_ViewID to either 
SV_ViewportArrayIndex (if rendering SBS) or 
SV_RenderTargetArrayIndex (if rendering to a texture array)

#### Vertex Shader Redirect
For each draw:
- Multiply the instance count by the number of views. If it isn't an instanced draw already, turn
it into one and make the instance count equal to the number of views
- In the VS preamble, set 
     gl_InstanceID = input.gl_InstanceID / numViews;
     gl_ViewID = input.gl_InstanceID % numViews;
- Outputs gl_ViewID to either 
SV_ViewportArrayIndex (if rendering SBS) or 
SV_RenderTargetArrayIndex (if rendering to a texture array)

# More Info
* Read ANGLE development [documentation](doc).
* Look at [pending](https://chromium-review.googlesource.com/#/q/project:angle/angle+status:open)
  and [merged](https://chromium-review.googlesource.com/#/q/project:angle/angle+status:merged) changes.
* Become a [code contributor](doc/ContributingCode.md).
* Use ANGLE's [coding standard](doc/CodingStandard.md).
* Learn how to [build ANGLE for Chromium development](doc/BuildingAngleForChromiumDevelopment.md).
* Get help on [debugging ANGLE](doc/DebuggingTips.md).


* Read about WebGL on the [Khronos WebGL Wiki](http://khronos.org/webgl/wiki/Main_Page).
* Learn about implementation details in the [OpenGL Insights chapter on ANGLE](http://www.seas.upenn.edu/~pcozzi/OpenGLInsights/OpenGLInsights-ANGLE.pdf) and this [ANGLE presentation](https://drive.google.com/file/d/0Bw29oYeC09QbbHoxNE5EUFh0RGs/view?usp=sharing).
* Learn about the past, present, and future of the ANGLE implementation in [this recent presentation](https://docs.google.com/presentation/d/1CucIsdGVDmdTWRUbg68IxLE5jXwCb2y1E9YVhQo0thg/pub?start=false&loop=false).
* If you use ANGLE in your own project, we'd love to hear about it!
