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

| Render Mode               |   API Required              |   Status          |   Performance  |  HW Support       |
|--------------------------:|:---------------------------:|:-----------------:|:--------------:|:-----------------:|
| Draw-Adjust-Draw          | Any                         | Planned for FL93  |   good         |  Almost All       |
| Geometry Shader Redirect  | D3D11 FL10+                 | Implemented       |   better       |  FL10+            |
| Vertex Shader Redirect    | D3D11.3 /w VPRTfromVS Cap** | Implemented       |   best         |  Subset of FL10+  |

** Support is exposed through an optional D3D 11.3 flag that isn't supported by all hardware/drivers. More info on [MSDN](https://msdn.microsoft.com/en-us/library/windows/desktop/dn933226(v=vs.85).aspx).

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

# Setup

### Prototype setup:
1.	Clone and build the [MultiviewPrototype ANGLE branch](https://github.com/shahmeeresmail/angle/tree/MultiviewPrototype)
2.	Clone the modified [Stereo three.js samples](https://github.com/shahmeeresmail/three.js/tree/StereoMultiview)
3.	Download chrome canary, build chromium from source or use another browser that will allow cmd stream pass through, draft extensions and local file access

### Workaround flags
Since Multiview isn't an official extension, its interfaces have not been exposed. Futher, the it currently does not specify any way to do side-by-side rendering. Therefore, workaround flags have been put in the ANGLE MultiviewPrototype branch to allow multiview to operate in some form. They are currently all enabled by default but this will probably change when Webgl_Multiview becomes officially supported. The flags are:
- multiviewEnabledWithViewIDUsage
    * Multiview is enabled and disabled depending on whether or not a shader is bound that uses multiview. This is needed until an extension interface is exposed that allows enabling/disabling of multiview.
- forceMultiviewSbs
    * When multiview is enabled, use SBS mode only. Needed until modes other than SBS are supported.
- multiviewStereoViews
    * When multiview enabled, force the view index to start at zero and the number of views to always be two. Needed until extension interfaces exposed to set the start offset and view count.
- autoCreateSbsViewsForMultiview
    * When multiview is enabled in SBS mode, generate Side-by-Side views by horizontally dividing the current viewport rect and scissor rect by the number of views. Needed until multiple viewports and scissors can be set.
    
### To Enable Multiview usage in samples:
1.	Do a Find/replace in the three.js samples html/js files:
a.	Find: var useMultiviewExtension = false;
b.	Replace: var useMultiviewExtension = true;
2.	Save all

### To Disable Multiview usage in samples:
1.	Do a Find/replace in the three.js samples html/js files:
a.	Find: var useMultiviewExtension = true;
b.	Replace: var useMultiviewExtension = false;
2.	Save all

### Running samples:
1.	Enable/Disable Multiview usage
2.	Close any browsers running samples (for now this is necessary to ensure shaders are properly recompiled when toggling Multiview on or off)
3.	Launch a browser with cmd/shader passthrough enabled, draft/unknown extensions enabled and local file access
a.	On chrome the command line is:
chrome --use-passthrough-cmd-decoder --allow-insecure-localhost --enable-webgl-draft-extensions --allow-file-access-from-files
b.	For firefox, edge and others, I am not sure what the command line parameters are. It may require custom builds
4.	Open index.html from the cloned samples using the previously launched browser instance. This will launch the sample browser where individual samples can be launched. Samples using Multiview should render two images side-by-side

### Notes:
-	Not all samples work yet. Some need additional shader or code modifications in order to work and others just won’t work well with multiview. 
-	Working samples should display rendered output side by side regardless of whether or not the Multiview extension is enabled. 
-	GPU bound samples will probably show little to no performance increase with Multiview whereas CPU bound ones can show significant gains.
-	If you are using chrome/chromium and would like to debug ANGLE while running the samples, add “--gpu-startup-dialog” to the command line parameters. This will  output the GPU process ID into a message box and pause execution until the message box is closed. Once the message box shows up, you can attach to the GPU process ID listed using Visual Studio and then close the message box to continue execution when you are ready. 


# More Info
* Get info on [VP/RT indices output support using a VS](https://msdn.microsoft.com/en-us/library/windows/desktop/dn933226(v=vs.85).aspx) (required for Vertex Shader Redirect)
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
