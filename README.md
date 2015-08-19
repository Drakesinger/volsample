
# volsample

Research on sampling methods for real-time volume rendering

![Teaser](huwb.github.com/volsample/img/teaser.jpg)

## Intro

This is the source code for the course titled *A Novel Sampling Algorithm for Fast and Stable Real-Time Volume Rendering*, in the *Advances in Real-Time Rendering in Games* course at SIGGRAPH 2015. The full presentation PPT is available for download from the course page [here][ADVANCES2015] - this is the best place to start for an introduction to this project.

This research introduces three main volume sampling techniques which are described below - Forward Pinning, General Pinning and Adaptive Sampling.

There are aspects of these that are far from perfect - see notes below. We'd love to get your help to improve the techniques and the implementation!


## Running

This is implemented as a Unity 5 project and should "just work" without requiring any set up steps. This is a very convenient framework for doing this research. It provides the tweaking UI for free and affords very fast iteration.

Upon opening scene Clouds in the project, you should immediately be able to move the camera around in the editor and see the effect on the render. If you play the project, the Animator component on the camera will play our test animation.

There are a few small gotchas that can arise - see the Troubleshooting section below.


## Algorithm

Most of the work is performed by scripts on the *Main Camera* game object.

### Forward Pinning

This is the first part of the presentation and involves shifting the samples to compensate for camera forward motion. The forward motion is integrated in the variable `m_distTravelledForward` in *CloudsBase.cs*. Note that since the rays are scaled, this has to be taken into account when offsetting the ray steps, so the integration code takes this into account.

This is then passed into the clouds shader in *CloudsRayScales.cs*, which then shifts the raymarch step position.

### General Pinning

The core of this is an advection process used to keep sample slices stationary. The two sample slices are drawn in the Editor view if `AdvectedScalesSettings.debugDrawAdvection` is true, so you can verify that they are stationary. These debug draws do not use Forward Pinning, so they will only appear stationary if you rotate and strafe the camera only.

The advection is implemented in *AdvectedScales.cs*. See the inline code comments for details. The general idea is that that both sample slices are kept as close to stationary as possible when the camera moves. We know how the camera has moved each frame, and therefore can work out the new angle for each sample slice point. We use Fixed Point Iteration (FPI) to invert this - for a particular angle in the final camera position (depending on which ray scale we are updating), it will provide an angle to a point on the sample slice which can then be used to compute a new scale.

Special handling is required for new scales (i.e. to extend the sample slice at the sides when the frustum moves). Ideally the sample slice would be extended along the motion path of the point at the right of the frustum. If this extension could be provided to FPI then FPI would probably yield the full, final solution. However I'm not sure if this is possible. Instead we detect if the iteration has landed off the slice and then manually extend the slice. This is done as a linear approximation to the extension - i.e. 1 line segment. If you e.g. spin the camera very fast you will see that the slice is extended using line segments. However for normal motion it is not visible and it seems to be pretty stable so this is probably good enough for our needs.

As far as we can tell Unity doesn't support uploading floats to FP32 textures, so the ray scales are written onto geometry which is then rendered into a floating point texture. See *UploadRayScales.cs*. I couldn't easily get it to work with a N x 1 texture, so I'm using a N x N texture instead.


### Adaptive Sampling

The sample layout overlay (white lines on top of render) is generated by *DiagramOverlay.cs*. This script contains a C# implementation of our adaptive sampling algorithm, described in our publication and illustrated [here](https://www.shadertoy.com/view/llXSD7 "Adaptive Sampling Diagram"). If you have the *Draw* and *Adaptive* options selected on this script, you should see it in action in the overlay.

Unfortunately Unity doesn't support passing arrays into shaders. For now we just compute the adaptive sampling directly in shader. It would be more efficient to upload these to a texture and read them from there, but like the ray scales this would probably have to be done through geometry and I haven't tried this. We could probably pack these into a non-FP32 texture format instead.


## Troubleshooting

You may run into the following:

* If you see just the standard sky box render in the camera view, re-enable the *CloudsRayScales* script on the camera. if it auto disables itself when you enable it, it is likely because the shader is not building. look in the console log for shader build errors. if you don't see any you may have already cleared them - perhaps try reopening Unity.
* You may notice the sample layout changes shape slightly when the camera moves forwards/backwards. This is actually by design and is happening in the advection code (if `advectionCompensatesForwardPin` is true), and compensates for the non-trivial motion of samples when we forward pin them. Look for comments around this variable and usages of the variable in the code.

## Bugs and improvement directions

There are many directions for improving this work

* Gradient relaxation is a little complicated at the moment, doing multiple passes in different directions from different starting points. I believe with experimentation this could be simplified. Also, it currently doesn't always work well enough - you can sometimes still see pinching. We may want to define a maximum gradient that is never exceeded (a hard limit instead of the current soft process).
* Generalisation from flatland 2D to full 3D motions. Ray scales are currently a 1D array of floats, indexed by ray angle on the xz plane. This would have to generalise to a 2D array. The advection procedure generalise easily. The gradient relaxation may need a bit of thinking to generalise, and might need to be a GPU process.
* The ray scale clamping works well most of the time but when transitioning straight from a strafing layout to a rotating layout, some of the ray scales get clamped and is causing some aliasing. I'm not sure if this is a bug or a limitation of the current clamping scheme.
* The adaptive sampling distances and weights are currently dynamically computed in *Clouds.shader*. It would be more efficient to pass them in to the shader - see notes above.
* The render generally breaks down when the camera is raised above the clouds etc. It would be valuable to polish this and make it work for all camera angles.

[ADVANCES2015]: http://advances.realtimerendering.com/s2015/index.html "Advances in Real-Time Rendering - SIGGRAPH 2015"
