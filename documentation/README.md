# neuVid Documentation

## Quick Start

The following tutorial videos walk through the basics of using neuVid:
* [Making Neuron Videos with neuPrint and neuVid](https://www.youtube.com/watch?v=bxbW0cumPPQ)
* [Camera Motion in Neuron Videos with neuPrint and neuVid](https://www.youtube.com/watch?v=P3VbpETjCjY)

## Introduction

The goal of `neuVid` is to simplify production of the rather stereotyped videos common in neuroscience research on the *Drosophila* fruit fly.  An important aspect of these videos is that they have simple camera motion: the camera usually starts looking at the fly brain face on (i.e., as if the fly were looking straight at the camera with its right on the camera's left), and translates freely but changes its viewing direction only by rotating around the  vertical axis.  The absence of free-form camera motion helps viewers stay oriented and understand what they are seeing.  It also keeps `neuVid` simple, with simpler commands to specify the animation, and a simpler implementation.

This documentation focuses on the use of `neuVid` with data sets having explicit segmentations of neurons.  It is also possible to use `neuVid` with some volumetric data sets lacking a segmentation, as described in this [other documentation](README_VVD.md).

## The Simplest Video

Probably the simplest video shows a single neuron spinning around the vertical axis.  With `neuVid`, such a video can be defined with twelve lines of text, nicely formatted.

The input to `neuVid` is [JSON](https://en.wikipedia.org/wiki/JSON), a simple structured filed format.  Its most important elements are:
- *arrays*, which are lists of values, delineated by `[` and `]`;
- *objects*, which are dictionaries or maps from keys to values, delineated by `{` and `}`.

Here is a `neuVid` input file for the simplest video:

```json
{
  "neurons" : {
    "anchor" : [
      508689623
    ],
    "source" : "https://hemibrain-dvid.janelia.org/api/node/52a13/segmentation_meshes"
  },
  "animation" : [
    [ "frameCamera", { "bound" : "neurons.anchor" } ],
    [ "orbitCamera", { "duration" : 12.0 } ]
  ]
}
```

The video then would be generated using these steps, described in more detail in the [Basic Usage](../README.md#basic-usage) documentation:

```
blender --background --python neuVid/importMeshes.py -- -i /tmp/simplest.json -o /tmp/simplest.blend
blender --background --python neuVid/addAnimation.py -- -i /tmp/simplest.json -ib /tmp/simplest.blend -o /tmp/simplestAnim.blend
blender --background --python neuVid/render.py -- -ib /tmp/simplestAnim.blend -o /tmp/framesFinal
blender --background --python neuVid/assembleFrames.py -- -i /tmp/framesFinal -o /tmp
```

The `-o` and `-ib` arguments may be omitted with their values being inferred from `-i /tmp/simplest.json`, allowing this more concise version:

```
blender --background --python neuVid/importMeshes.py -- -i /tmp/simplest.json
blender --background --python neuVid/addAnimation.py -- -i /tmp/simplest.json
blender --background --python neuVid/render.py -- -i /tmp/simplest.json -o /tmp/framesFinal
blender --background --python neuVid/assembleFrames.py -- -i /tmp/framesFinal -o /tmp
```

It is easiest to edit a JSON file in a modern text editor that understands and visually highlights the JSON file format; a good candidate that works on most platforms is [Visual Studio Code](https://code.visualstudio.com).  By default, JSON does not allow comments.  In `neuVid`, comment lines starting with `#` or `//` are allowed, as they are stripped before further JSON processing.  To prevent Visual Studio Code from getting confused by such comments, use the "Select Language Mode" button at the bottom of the window to switch from "JSON" to "JSON with Comments".

Be careful not to add a comma after the last item in a JSON block, like an array (e.g., use `[1, 2, 3]`, not `[1, 2, 3,]`).  This common mistake produces an error when `neuVid` parses the JSON file, but usually `neuVid` is able to give an error message indicating the location of the extraneous comma.

## The Simplest Video, Deconstructed

The top-level JSON object in the input file has two major sections.

The `neurons` key starts the *definitions section*.  This section associates *names* with data that will be imported as polygonal meshes that Blender can render.  The `neurons` key's value is an object, and its `anchor` key is an example of a name.  The `anchor` name is associated with that key's value, an array containing the identifier number `508689623`.  Any use of the name `neurons.anchor` thus refers to the neuron with that identifier.

Implicitly, all the named neurons are rendered as meshes.  Where these meshes come from is described by the `neuron` object's `source` key.  In this case, its value states that the meshes come from a [DVID server](https://hemibrain-dvid.janelia.org), specifically, its `segmentation_meshes` data instance.

The first explicit use of the `neurons.anchor` name occurs in the other major section, the *animation section*.  In the top-level object, the `animation` key's value is an array of *commands* that describe the animation that Blender will render.  The first command, `frameCamera`, tells Blender how to position the camera: the bounding box for `neurons.anchor` should mostly fill the camera frame, with the camera looking at the fly's brain face on.  Note that this useful camera composition is specified symbolically, without the use of absolute coordinates.  Minimizing the need for absolute coordinates is one of the guiding principles of `neuVid`.

The other command in the animation section, `orbitCamera`, describes the first actual animation, or change over time.  This command makes the camera take 12 seconds to rotate 360° around the center of `neurons.anchor`'s bounding box, with the axis of rotation being the vertical axis.  While the length of the animation is specified explicitly with the `duration` argument, the other details are defined implicitly: the center of rotation comes from `neurons.anchor`'s  bounding box because that was the target for the previous camera command, `frameCamera`, and the final rotation angle of 360° makes sense because full rotation back to the starting point is a common choice.
This implicit behavior can be changed with optional arguments to `orbitCamera`, but having usable defaults is another guiding principle of `neuVid`.

When the input file is edited, some `neuVid` scripts need to be rerun, but which scripts depends on which sections were changed.  Edits to the definitions section require rerunning the `importMeshes.py` script and then the `addAnimation.py` script.  (If synapses are changed, then the `buildSynapses.py` script must be run before `importMeshes.py`; see the [Synapses](#synapses) documentation.)  Edits to the animation section require rerunning only the `addAnimation.py` script.

## Previewing

For videos more complex than the simplest one, it is useful to see a preview before taking the time to render all the frames.  The simplest way to do so is to run the interactive Blender application and load the file produced by `addAnimation.py`. Press the "Play Animation" button by the timeline to see a lower-quality rendering of the animation.  (Note that with Blender 2.79, this interactive rendering of silhouette shading and transparency is rather bad, but the situation is significantly better for Blender 2.80 and later.)

To preview the real rendering, another approach is to render only every `n` frames (e.g., `n` of 24 would render on frame per second of final animation, for the default 24 frames per second).  The `render.py` script supports this approach with the `-j n` (or `--frame-jump n`) argument.

To render only a single frame, `f`, use `render.py`'s `-s f -e f` (or `--frame-start f --frame-end f`) arguments.

## Categories

The `neurons` key in the simplest video is one example of a *category*.  Names associated with data, like `anchor`, are defined within a category, and the category determines how that data will be rendered by Blender.  

There are four categories in `neuVid`, each with its own key in the definitions section:

- `neurons`: Neurons are rendered as colored surfaces with shading and shadows (which look better with a path-tracing renderer like Cycles or Octane).
- `rois`: ROIs are rendered as white silhouettes.
- `synapses`: Synapses are rendered as colored balls, with a bit of extra brightness suggesting that they emit light.
- `grayscales`: This rather specialized category is for 2D images, with the typical usage being to show the grayscale images of the original electron microscopy (EM) data.  These images are rendered as a special "picture in picture" element on in front of the 3D rendering.

Each category is optional.  If a video shows only neurons, for example, it would not need the keys for `rois`, `synapses`, and `grayscales`.

Here is an example that adds the `rois` category to the previous example with `neurons`:

```json
{
  "neurons" : {
    "anchor" : [
      508689623
    ],
    "source" : "https://hemibrain-dvid.janelia.org/api/node/52a13/segmentation_meshes"
  },
  "rois" : {
    "source" : "https://hemibrain-dvid.janelia.org/api/node/52a13/roisSmoothedDecimated",
    "step1" : [
      "MB(R)", "LAL(R)"
    ]
  },
  "animation" : [
    [ "setValue", { "meshes" : "rois.step1", "alpha" : 0.05 } ],
    [ "frameCamera", { "bound" : "neurons.anchor" } ],
    [ "orbitCamera", { "duration" : 12.0 } ]
  ]
}
```

The `rois` category in this example defines a name, `rois.step1` that is associated with *two* ROIs.  The `source` key specifies that the meshes for these ROIs are loaded from [another data instance of the DVID server](https://hemibrain-dvid.janelia.org), `roisSmoothedDecimated`.  The actual ROI names should follow the standards of DVID as presented by the [`neuPrintExplorer`](https://neuprint.janelia.org/?dataset=hemibrain%3Av1.0.1) user interface (e.g., `aL(R)`, `b'L(R)`, `EB`, etc.).  Note that the order of keys in an object is irrelevant, and the `rois` category has the `source` key first.

Note also that the same name can be reused in different categories (e.g., `rois.step1` could have been `rois.anchor` without conflicting with `neurons.anchor`), and the names in the categories need not match.

The animation section in this example has a new command, `setValue`.  This command changes the alpha (transparency) value for the meshes associated with the `rois.step1` name.  This one command affects all that name's meshes, so it affects both `MB(R)` and `LAL(R)`.  The [Transparency and Color](#transparency-and-color) documentation, below, discusses alpha in more detail.  Using names as command arguments provides *indirection*, so an animation for a neuron can be defined without explicit mention of that neuron's identifier, and can be reused with a different neuron simply by changing the name definition.

Both `source` values are DVID URLs in this example.  Using such a URL, `neuVid` fetches the appropriate meshes and saves them as [OBJ files](https://en.wikipedia.org/wiki/Wavefront_.obj_file) in local directories to be used by Blender.  For `neurons` the directory is `neuVidNeuronMeshes`, and for `rois` it is `neuVidRoiMeshes`, with both directories created in the same directory as the input JSON file.  As an alternative, the `source` value can specify the directory Blender should use for the OBJ files.  For `neurons`, OBJ files in this directory should have names like `508689623.obj`.  For `rois`, it does not work to use the standard ROI names, as most platforms do not allow parentheses so file names like `MB(R).obj` are not valid.  A solution is to use names like `MBR.obj` and `LALR.obj`, making sure that the `rois` category matches:

```json
{
  "neurons" : {
    "anchor" : [
      508689623
    ],
    "source" : "https://hemibrain-dvid.janelia.org/api/node/52a13/segmentation_meshes"
  },
  "rois" : {
    "source" : "/tmp/localMeshDirectory/",
    "step1" : [
      "MBR", "LALR"
    ]
  },
  "animation" : [
    [ "setValue", { "meshes" : "rois.step1", "alpha" : 0.05 } ],
    [ "frameCamera", { "bound" : "neurons.anchor" } ],
    [ "orbitCamera", { "duration" : 12.0 } ]
  ]
}
```

## Time

The simplest video uses the `orbitCamera` command with a `duration` argument to make animated camera motion start at the beginning of the video and end 12 seconds later.  To specify changes that start at any time after the beginning of the video, use the `advanceTime` command.  Here is an example:

```json
{
  "rois" : {
    "source" : "https://hemibrain-dvid.janelia.org/api/node/52a13/roisSmoothedDecimated",
    "step1" : [
      "MB(R)"
    ],
    "step2" : [
      "LAL(R)"
    ]
  },
  "neurons" : {
    "anchor" : [
      508689623
    ],
    "source" : "https://hemibrain-dvid.janelia.org/api/node/52a13/segmentation_meshes"
  },
  "animation" : [
    [ "frameCamera", { "bound" : "rois.step1" } ],

    [ "advanceTime", { "by" : 1.0 } ],

    [ "frameCamera", { "bound" : "rois.step2", "duration" : 2.0 } ],
    [ "advanceTime", { "by" : 2.0 } ],

    [ "advanceTime", { "by" : 1.0 } ],

    [ "frameCamera", { "bound" : "rois.step1 + rois.step2", "duration" : 2.0 } ],
    [ "advanceTime", { "by" : 2.0 } ],

    [ "advanceTime", { "by" : 1.0 } ]
  ]
}
```

The animation opens with the camera framed on the bounds for `rois.step1`. The next command is `advanceTime` `by` `1.0` seconds (omitting the extra JSON syntax).  The result is that the subsequent `frameCamera` command with `duration` `2.0` causes the camera to start moving one second into the video and end three seconds in.

Note that even though that `frameCamera` command specifies `duration` `2.0`, the time at which any subsequent command would start remains at one second into the video.  Thus, additional `advanceTime` commands are necessary to make the subsequent commands start later.

Note also the use of *two* `advanceTime` commands, one with `duration` `2.0` and then another with `duration` `1.0`.  The same effect would be possible with one `advanceTime` command of `duration` `3.0`, but using two commands reinforces the idea that there is time advancement to match the `duration` of the `frameCamera` command for `rois.step2`, and then additional time advancement to add a pause before the start of the next command.  This approach makes it a bit simpler to edit the input to change the length of the pause.

The next command is a third `frameCamera`.  It specifies that the framing should be on the bounds of `rois.step1 + rois.step2`.  The effect is to make the camera frame on the union of the two bounds.  No set operators other than `+` are supported for the `bound` argument, but the next section discusses a more powerful case for other arguments of other commands.

A variety of interesting animations are possible by combining commands like `frameCamera`, `orbitCamera`, and `advanceTime`, but not all combinations are supported.  It does not work to "nest" a `frameCamera` command in the middle of the duration of an `orbitCamera` command, for example.  A solution is to break the longer-duration `orbitCamera` into two shorter-duration commands, with the `frameCamera` in between them.

## Transparency and Color

The transparency of a mesh is controlled by the *alpha* value on the mesh's material.  The [Catetories](#categories) documentation showed how the `setValue` command can specify an alpha for a mesh for the duration of a video.

To vary the alpha, use the `fade` command.  It takes a `duration` argument so the alpha can change gradually over time, which looks more appealing than an abrupt change.  Here is an example:

```json
{
  "neurons" : {
    "source" : "https://hemibrain-dvid.janelia.org/api/node/52a13/segmentation_meshes",
    "anchor" : [
      508689623
    ],
    "connectedToAnchor" : [
      976506780, 763730414
    ]
  },
  "animation" : [
    [ "frameCamera", { "bound" : "neurons.anchor + neurons.outputsToAnchor" } ],
    [ "orbitCamera", { "duration" : 12.0 } ],

    [ "advanceTime", { "by" : 4.0 } ],

    [ "fade", { "meshes" : "neurons.connectedToAnchor", "startingAlpha" : 0.0, "endingAlpha" : 1.0, "duration" : 2.0 } ],
    [ "advanceTime", { "by" : 2.0 } ],

    [ "advanceTime", { "by" : 4.0 } ],

    [ "fade", { "meshes" : "neurons.connectedToAnchor", "startingAlpha" : 1.0, "endingAlpha" : 0.0, "duration" : 2.0 } ],
    [ "advanceTime", { "by" : 2.0 } ]
  ]
}
```

An alpha of zero makes a mesh completely transparent, while an alpha of one makes it completely opaque.  If no alpha is specified with a `setValue` or `fade` command, it is one by default, so `neurons.anchor` keeps an alpha of one for the whole video.

Note that the two `fade` commands have starting and ending times completely within the duration of the `orbitCamera` command.  It is fine to overlap animation commands in this way, to create more sophisticated effects.

Color is closely related to transparency.  By default, `neuVid` assigns colors to neuron meshes automatically, drawing from a six-color palette:

![neuVid's color palette](palette.png)

This palette is based on a common [color-blind-friendly palette](https://www.color-hex.com/color-palette/49436).  The colors have been changed slightly to make their [relative luminance](https://en.wikipedia.org/wiki/Relative_luminance) values (the percentages in the third row) more similar.  Relative luminance describes the brightness of the color as perceived by a human, and making these values more similar reduces the chance that some colors will appear more prominent.  The change to relative luminance made the palette somewhat less effective in color-blind conditions, but nevertheless it works relatively well for 3D renderings, where lighting also affects the colors.

It is possible to override the automatic assignment of colors to neuron meshes.  The `setValue` command with the argument `color` is a good way to make a change for the whole video, while `fade` can be used to interpolate between the `startingColor` and `endingColor` arguments over a `duration`.  The value of a `color`, `startingColor` or `endingColor` argument can take any of several forms, as illustrated in the following example:

```json
{
  "neurons" : {
    "source" : "https://hemibrain-dvid.janelia.org/api/node/52a13/segmentation_meshes",
    "main" : [
      508689623
    ],
    "partners" : [
      5813022375, 763730414
    ],
    "more" : [
      914778159, 976506780
    ],
    "special" : [
      763730414
    ]
  },
  "animation" : [
    [ "setValue", { "meshes" : "neurons.main", "color" : 3 } ],
    [ "setValue", { "meshes" : "neurons.special", "color" : "green" } ],
    [ "setValue", { "meshes" : "neurons.partners + neurons.more - neurons.special", "color" : "#ababab" } ],

    [ "frameCamera", { "bound" : "neurons.main" } ],
    [ "orbitCamera", { "duration" : 12.0 } ]
  ]
}
```

Each of the three `setValue` commands shows one of the ways of specifying colors:

- an index value between 0 and 5, to choose from the colors in the order of the image, above;
- a name of a color, from the choices in the bottom rows of the image, above;
- a custom color using the [CSS hex notation](https://www.w3schools.com/colors/colors_hexadecimal.asp), a string starting with `#` followed by two hexidecimal digits each for the red, green, and blue components.

Note that int the last `setValue` command, the `meshes` argument has the value `neurons.partners + neurons.more - neurons.special`.  The effect is to assign the `#ababab` color to all the neurons in `neurons.partners` and `neurons.more` except the neurons in `neurons.special`.  Any `meshes` argument can use set operations like these on names from the `neurons` and `rois` categories, but not from the `synapses` category.

The `rois` category renders meshes as silhouettes, with mesh faces becoming more transparent as they come closer to facing the camera.  To control how quickly this transition to transparency occurs, the `rois` category has an optional key, `exponents`.  The default value is five, and a *higher* value makes a mesh transition to transparency *faster*.  Here is an example assigning a value of eight to a group of ROIs that should be deemphasized:

```json
{
  "rois" : {
    "source" : "https://hemibrain-dvid.janelia.org/api/node/52a13/roisSmoothedDecimated",
    "deemphasized" : [
      "FB", "EB"
    ],
    "emphasized" : [
      "NO"
    ],
    "exponents" : {
      "deemphasized" : 8
    }
  }
}
```

In the `exponents` object, the special key `*` refers to all ROIs not explicitly mentioned in other keys.  See the [Tips](#tips) documentation, below, for some ideas about how to use `exponents` and `alpha`.

## Synapses

Synapses appear as little spheres, and are defined by names in the `synapses` category.  Here is a simple example:

```json
{
  "neurons" : {
    "source" : "https://hemibrain-dvid.janelia.org/api/node/52a13/segmentation_meshes",
    "anchor" : [
      508689623
    ]
  },
  "synapses" : {
    "source" : "https://neuprint.janelia.org/?dataset=hemibrain:v1.0.1",
    "anchorPSD" : {
      "neuron" : 508689623,
      "type" : "post"
    }
  },
  "animation" : [
    [ "setValue", { "meshes" : "synapses.anchorPSD", "color" : "#303030" } ],
    [ "frameCamera", { "bound" : "neurons.anchor" } ],

    [ "fade", { "meshes" : "neurons.anchor", "startingAlpha" : 1.0, "endingAlpha" : 0.8, "duration" : 2.0 } ],
    [ "fade", { "meshes" : "synapses.anchorPSD", "startingAlpha" : 0.0, "endingAlpha" : 1.0, "duration" : 2.0 } ],
    [ "advanceTime", { "by" : 2.0 } ],

    [ "pulse", { "meshes" : "synapses.anchorPSD", "toColor" : "#ffffff", "duration" : 4.0 } ],
    [ "advanceTime", { "by" : 4.0 } ]
  ]
}
```

In the `synapses` category, the `source` key must be a [`neuPrintExplorer`](https://neuprint.janelia.org/?dataset=hemibrain%3Av1.0.1) URL.  Each other key defines the name for a synapse set.  In this example, the `synapses.anchorPSD` name refers to to synapses from the neuron `508689623`, and the `type` key being `post` limits these synapses to the post-synaptic sites (PSDs).

PSDs have a gray color in the [`NeuTu` proofreading system](https://github.com/janelia-flyem/NeuTu), so the `setValue` command changes the meshes for `synapses.anchorPSD` to have a similar color.

Synapse locations can be slightly inside the branches of neurons, which makes the display of synapses and neurons as solid objects challenging.  It helps somewhat to make the neurons slightly transparent, so the first `fade` command does exactly that, making `neurons.anchor` become 80% opaque over a period of two seconds.  Simultaneously, the other `fade` command makes `synapses.anchorPSD` fully opaque.

Even with this fading, smaller synapse sets may be hard to notice, especially if they are colored gray.  The `pulse` command can help by gently (once per second) cycling from the current color to a highlighting color, which is white in this example.

As described in more detail in the [Usage with Synapses](../README.md#usage-with-synapses) documentation, animations with synapses require the running of an extra script, `buildSynapses.py`.  This script queries the synapse locations from the `neuPrint` server mentioned in the `source` URL.  The full set of steps to generate the animation are as follows:

```
python neuVid/buildSynapses.py -i /tmp/synapses.json
blender --background --python neuVid/importMeshes.py -- -i /tmp/synapses.json
blender --background --python neuVid/addAnimation.py -- -i /tmp/synapses.json
blender --background --python neuVid/render.py -- -i /tmp/synapses.json -o /tmp/framesFinal
blender --background --python neuVid/assembleFrames.py -- -i /tmp/framesFinal -o /tmp
```

The object for `anchorPSD` has two keys, but `neuVid` supports additional keys to fine tune a synapse set:

- The `type` value can be `pre` for pre-synaptic sites (T-bars) or `post` for post-synaptic sites (PSDs), or omitted for both.  For `pre`, use `setValue` `color` `yellow` to match the coloring of NeuTu.
- If `radius` specifies a number then the rendered sphere for each synapse has that radius.
- If `partner` specifies the identifier for another neuron, then synapses are limited to those shared with that neuron.
- If `roi` specifies the name of an ROI, then synapses are limited to those in that ROI.  Boolean operators are supported (e.g., `"roi" : "not EB and not FB"` for only the synapses outside of both `EB` and `FB`) but not grouping parentheses.
- The `includeWithin` key can specify a bounding-box object to put spatial limits on the synapses.  This object can have any combination of `xMin`, `xMax`, `yMin`, `yMax`, `zMin` or `zMax` as keys (e.g., `"includeOnly" : { zMax : 19000 }` omits any synapse with `z` coordinates larger than 19,000).

## Grayscale Images

*Incomplete*

- The `grayscales` category, names map to:
  - `source`: one image, or an .avi file for a "movie texture"
  - `position`
- Use `centerCamera` to get the position.
- Use `showPictureInPicture` to animate a rotating "card" that shows the image or movie texture
- Card animation makes sense for EM slices, which are small and perpendicular to the standard viewing angle

![Animation from showPictureInPicture](showPiP.png)

- Use `showSlice` to animate a "card" shwoing the image or movie texture without the initial and final rotation of `showPictureInPicture`
  - `source`: one image, or an .avi file for a "movie texture"
  - `bound`: the image will animate through this bounding box
  - `euler`: angles for the card, as an alternative to `bound`
    - `position`
    - `scale`
    - `distance`
  - `duration`
  - `delay`
  - `fade`

## Renderers

*Incomplete*

| Renderer | Quality | Speed | Capacity | Ease of use | Free |
| -------- | ------- | ----- | -------- | ----------- | ---- |
| Eevee    | *       | ***   | ***      | ***         | Yes  |
| Cycles   | ***     | *     | ***      | ***         | Yes  |
| Octane   | ***     | **    | *        | **          | No   |

- "Capacity" refers to the amount of data (i.e., neuron meshes, ROI meshes) that the renderer can handle.  Octane's capacity is limited by GPU memory.  Cycles can use either the CPU or the GPU, and the CPU often can handle more data.  On a machine with 32 GB of memory, for example, it has rendered a scene with almost one-quarter billion mesh faces.

- Rendering with Cycles:
  - The default choice.
  - If black spots appear on fading objects (i.e., objects with alpha less than one), try a `--transparent-max-bounces` (or `-tmb`) value greater than the default of 32.
  - Uses only the CPU by default.  To use the GPU, use one of the following flags, to choose the options [described in the Blender documantion](https://docs.blender.org/manual/en/latest/render/cycles/gpu_rendering.html):
    - `--optix` (or `-optix`)
    - `--cuda` (or `-cuda`)
    - `--hip` (or `-hip`)
    - `--metal` (or `-metal`)
  - Wastes time on a "Synchronizing object" step for every object at almost every frame.  Use `--persistent` (or `-p`) to avoid it, but note that this option causes a crash if the system does not have sufficent memory (RAM).

- Rendering with Octane:
  - `-oct` or `--octane` argument for `render.py`
  - `--roi` argument `render.py`
  - `compRoisAndNeurons.py` script

- Rendering with Eevee:
  - `-ee` or `--eevee` argument for `render.py`

## Neuroglancer

*Incomplete*

[Neuroglancer](https://github.com/google/neuroglancer) has its own way of rendering videos.  Documentation for using [Neuroglancer's own video-generation script](https://github.com/google/neuroglancer/blob/master/python/neuroglancer/tool/video_tool.py) consists of the comments at the top of the script.  To use the script, install the necessary support software with [Conda](https://docs.conda.io/en/latest/miniconda.html):

```
conda create -n ng -c conda-forge geckodriver pillow numpy requests tornado six google-apitools google-auth atomicwrites 
conda activate ng
pip install neuroglancer
```
Copy the Neuroglancer URLs for key moments and paste them into a text file, say, `/tmp/ng.txt`.  Separate these URL lines with timing lines, containing just the number of seconds to advance before applying the next URL.  Generate frames from `ng.txt` (using [Firefox]( https://www.mozilla.org/) to do the rendering in this example):
```
python -m neuroglancer.tool.video_tool render --browser firefox --hide-axis-lines --height=1080 --width=1920 /tmp/ng.txt /tmp/framesNg
```
Assemble the video:
```
blender --background --python neuVid/assembleFrames.py -- -i /tmp/framesNg -o /tmp
```
To use `neuVid` instead, convert `ng.txt` to a JSON file for `neuVid` input:
```
python neuVid/importNg.py -i /tmp/ng.txt -o /tmp/fromNg.json
```
Then proceed as usual.

Why convert to `neuVid`:
* Smoother animation transitions, with ["slow in and slow out"](https://en.wikipedia.org/wiki/Twelve_basic_principles_of_animation#Slow_in_and_slow_out)
* More believable rendering, with global illumination effects from Cycles or Octane
* Simpler editing

To see how `neuVid` simplifies editing, note that `neuVid`'s input is _commands_ to create animation rather than the _state_ at key moments in the animation as with Neuroglancer.  Many of the edits involved in refining a video are small when expressed as commands, but have effects on much or all of the state of the video, and so are large when expressed as changes to that state.  

Consider the following example:
1. Frame the camera on neuron _A_
2. Start orbiting the camera around _A_
3. Two seconds later, fade on neuron _B_
4. Two seconds later, fade on neuron _C_
5. Two seconds later, fade off neuron _A_
5. Two seconds later, stop orbiting the camera

State in the corresponding Neuroglancer URLs:
1. Camera facing _A_, camera rotation 0º, _A_ visible, _B_ not visible, _C_ not visible
2. Camera facing _A_, camera rotation 90º, _A_ visible, _B_ visible, _C_ not visible
3. Camera facing _A_, camera rotation 180º, _A_ visible, _B_ visible, _C_ visible
4. Camera facing _A_, camera rotation 270º, _A_ not visible, _B_ visible, _C_ visible
5. Camera facing _A_, camera rotation 360º, _A_ not visible, _B_ visible, _C_ visible

The edit is to make the camera frame on _B_ instead of _A_.

All the state URLs must be updated because every one encodes the camera facing _A_.

On the other hand, consider the corresponding `neuVid` input:
```
[ "frameCamera", { "bound" : "neurons.A" } ],
[ "orbitCamera", { "duration" : 4.0 } ],
[ "advanceTime", { "by" : 1.0 } ],
[ "fade", { "meshes" : "neurons.B", "startingAlpha" : 0.0, "endingAlpha" : 1.0, "duration" : 1.0 } ]
[ "advanceTime", { "by" : 1.0 } ],
[ "fade", { "meshes" : "neurons.C", "startingAlpha" : 0.0, "endingAlpha" : 1.0, "duration" : 1.0 } ]
[ "advanceTime", { "by" : 1.0 } ],
[ "fade", { "meshes" : "neurons.A", "startingAlpha" : 1.0, "endingAlpha" : 0.0, "duration" : 1.0 } ]
[ "advanceTime", { "by" : 1.0 } ]

```
The edit in this case involves merely changing the first command's `"neurons.A"` argument to `"neurons.B"`.

Another example: change the camera orbiting to total 180º instead of 360º.

This kind of editing is common during iterative refinement to reach a desired final video.  So the best use of Neuroglancer might be to set up the initial, rough version of a video, with `neuVid` being used for the iterations of editing to reach the final result.

Current limitations of importing Neuroglancer into `neuVid`:
* Only segmentation meshes (e.g., neurons, ROIs) are imported; other data (grayscale images, synapses) are ignored.
* Only meshes in OBJ format or the [Neuroglancer legacy single-resolution format](https://github.com/google/neuroglancer/blob/master/src/neuroglancer/datasource/precomputed/meshes.md#legacy-single-resolution-mesh-format) are supported.
* Only one camera framing is created automatically: an initial framing, on a prominent body chosen by heuristics.  Once the translation has produced a `neuVid` input JSON file, though, it is easy to add additional `frameCamera` commands as needed.
* Only camera orbiting around principal axes (_X_, _Y_, or _Z_) is supported.

## SWC Files

*Incomplete*

Example input JSON fragment:
```json
{
  "neurons": {
    "source": "./directory-of-swc-files",
    "cerebellum": ["AA0431.swc", "AA0964.swc"]
  },
  ...
}
```

The `.swc` file extension can be omitted:
```json
{
  "neurons": {
    "source": "./directory-of-swc-files",
    "cerebellum": ["AA0431", "AA0964"]
  },
  ...
}
```

A [SWC file](https://neuroinformatics.nl/swcPlus/) contains a sequence of nodes, with parent relationships defining a hierarchy or skeleton.  Each segment of the skeleton is represented in `neuVid` as a truncated cone, which is a cylinder if the radii at the ends (parent node, child node) are equal.

Optional aguments to `importMeshes.py` to control the generation of OBJ files from SWC files:
* `--swcvc` [default value: 12]: the vertex count for a cross section of the cones (cylinders).  For example, a value of 12 means the cones have 12 facets.  The default values works well for large collections of neurons. A larger value (e.g, 16 or 32) might improve the visual quality for videos with close-up views of smaller collections of neurons.
* `--swcar` [default value: 10]: a multiplicative factor for the radii of axonal segments (SWC type 2).  For example, SWC files from the [Janelia MouseLight project](https://www.janelia.org/project-team/mouselight) have all radii set to 1, and they appear too thin without being multiplied by some factor.
* `--swcdr` [default value: 15]: a multiplicative factor for the radii of dendritic segments (SWC type 3).  A convention of the MouseLight project is to make dendrites appear slightly fatter than axons.

An orientation correction is helpful with some SWC files, like those from the [Janelia MouseLight project](https://www.janelia.org/project-team/mouselight). Neurons from this project can be searched and downloaded from the [Mouse Light Neuron Browser](https://ml-neuronbrowser.janelia.org). For theses neurons, an initial `orbitCamera` command will set the default `neuVid` camera to look directly at the mouse's face, and a `lightRotationX` statement will make the lighting look more appealing:
```json
{
  "rois": {
    "source": "neuVid/test/test-roi-source",
    "shell": ["brain-shell-997"]
  },
  "neurons": {
    ...
  },
  "lightRotationX": 20,
  "animation": [
    ["orbitCamera", {"axis": "x", "endingRelativeAngle": -90, "duration": 0}],
    ...
  ]
}
```
Note also that a properly oriented mesh for the overall brain shell is available as [`test/test-roi-source/brain-shell-997.obj` from this repo](https://github.com/connectome-neuprint/neuVid/blob/master/test/test-roi-source/brain-shell-997.obj).


## Axes

*Incomplete*

In this example, the biological axes are rotated slightly from the Cartesinan axes, by angles -10 degrees around _X_ and 20 degrees around _Z_.  The rotated _Y_ axis is labeled as the anterior-posterior axis, and the rotated _Z_ axis is labeled as the ventral-dorsal axis.  The rotated _X_ axis is unlabeled.  These biological axes start out invisibile and then fade on partway through the animation.
```json
{
  "neurons" {
    ...
  },
  "rois" {
    ...
  },
  "axes" {
    "main": {
      "labels": {"+y": "A", "-y": "P", "+z": "V", "-z": "D"},
      "rotation": [-10, 0, 20]
    }
  }
  "animation" [
    ...
    ["fade", {"meshes": "axes.main", "startingAlpha": 0, "endingAlpha": 1, "duration": 1}],
    ...
  ]
}
```

Additional options for entries in `"axes"`:
* `"position"` (default [0.945, 0.099], in the bottom-right): normalized horizontal and vertical, between 0 and 1, with [0, 0] at bottom-left
* `"size"` (default 0.0245): normalized height, between 0 and 1

## Natural Language Input and Generative AI

*Incomplete*

Do not type any sensitive personal information into `generate`, because this information will be sent to the LLM model host (OpenAI).

To change the OpenAI or Anthropic API key, use the "Settings/API key..." menu item on the main menu bar.

The "conditioning" data that is included in the generative AI prompts is in the file [`documentation/training.md`](training.md).  The examples in this file are helpful for a human learning to use the system, too.

The generative AI works pretty well with the most powerful models: `gpt-4-turbo-preview` from OpenAI, or `claude-3-opus-20240229` from Anthropic. The less expensive models (e.g., `gpt-3.5-turbo` or `claude-3-sonnet-20240229`) may generate acceptable results for simpler descriptions but do not work as well in general. Even the powerful models can make mistakes.  More common mistakes include:
* omitting the `"advanceTime"` command necessary to let another command finish
* omitting the initial light rotation and `"orbitCamera"` necessary to put the FlyEM MANC and FlyWire data sets in the proper orientation
* not understanding multiplication (e.g., "Give the synapses the default radius * 2")

Sometimes a description that generates the wrong JSON one time will generate the right JSON the next time.

The `generate` application calls the `generate()` function from `gen.py`.  That function can be run from the command line, too, as in this example:

```
$ cd neuVid/neuVid
$ export OPENAI_API_KEY=sk-a...9O2U
$ python gen.py -o /tmp/example.json "Frame on IN00A001s 10477 and 10977 from the MANC. Orbit 30 degrees taking 3 seconds."

```

There is a suite of tests that can be run as follows:
```
$ cd neuVid/test
$ export OPENAI_API_KEY=sk-a...9O2U
$ python test-generate.py test-generate-input.txt
```
The environment variable for the Anthropic API key is `ANTHROPIC_API_KEY`.
By default, the results go to a file with a path like `/tmp/test-generate-results_gpt-4_2023-12-08_10:20:30.txt`.  To check the tests, compare this file to `test-generate-expected.txt`.

Note that running the tests is slow (to avoid OpenAI throttling) and costs around $7 with the current OpenAI pricing.


## Compute Cluster Usage

*Incomplete*

Additional arguments to `clusterRender.py`:
* `--cluster` [`-cl`] [optional, default value: `"gpu_rtx8000"`]: the name of the cluster to use. Note that for best performance, if the cluster uses GPUs (e.g., at Janelia, the cluster name has the `"gpu_"` prefix) then the additional arguments to the `clusterRender.py` script should tell Blender to use a GPU (e.g., `--optix` or `--cuda` on Linux, which is the typical operating system for a compute cluster).
* `--slots` [`-n`] [optional, default value: 32]: the number of slots (cores) to be used for the job. Note that for best peformance, Blender must know this value and use it for its thread count. To this end, `clusterRender.py` automatically passes this value as the `--threads` argument to `render.py`, so do not explicitly add another `--threads` argument.
* `--log` [`-l`] [optional, default value: a file having the same name as the input JSON file and the suffix `_log_` plus a timestamp]: the log file to contain the output of the `bsub` command.
* `--async` [`-as`] [optional, default value: `False`]: run `clusterRender.py` asynchronously, returning immediately instead of waiting for the job to come off the "pending" queue and run to completion.
* `--split` [optional, default value: no splitting]: splits the frames _within one video_ across cluster nodes, by setting the  `--frame-start` (`-s`) and `--frame-end` (`-e`) arguments of the `render.py` calls.
  - `--split `_n_ : creates _n_ jobs dividing the video frames evenly
  - `--split` :  uses `["fade", {"startingAlpha: 0, ...}]` (i.e., fade start) commands as boundaries for splitting the frames

Parallelizing `importMeshes.py` with `clusterImportMeshes.py`:
* Works only if `"neurons"` contains `"separate": true`, to make separate Blender files for the neurons with different sources, as [described in the next section](#advanced).
* Submits a separate `importNeurons.py` job for each separate Blender file (i.e., each item in the `"neurons"` `"source"` array).
* If there are _M_ separate files, then the job for file _i_ of _M_ has `--split `_i_ _M_ as and additional argument to `importNeurons.py`
* Blender uses only one thread (CPU core) when importing, so the jobs have `-n 1`.
* Given that many modern desktop computers have more than one core, importing can be parallelized on a single such computer (if no cluster is available) by manually using `--split `_i_ _M_ as additional arguments to `importNeurons.py`. Try a value of _M_ = 4 (regardless of the number of separate files) as a starting point, and check the resulting speedup before trying a larger _M_.
* A final `importMeshes.py` with `--skipExisting` is needed to build the overall Blender file that references the separate files.  This run also imports the ROIs and synapses (if present).

For clarity, `clusterRender.py` and `clusterImportMeshes.py` echo the actual `bsub` commands they will use to submit jobs before peforming the submission.

Some details of the `bsub` command may be specific to the cluster at [Janelia](https://www.janelia.org).

## Advanced

*Incomplete*

- Runtime arguments to `importMeshes.py`:
 - `--skipExisting` (`-sk`): do not download existing neuron/ROI/synapse meshes, which have been converted to OBJ files by earlier sessions

- Input JSON arguments for `render.py`:
  - `fps`
  - `lightPowerScale`
  - `lightSizeScale`
  - `lightDistanceScale`
  - `lightColor`
  - `lightRotationX` in degrees
  - `lightRotationY` in degrees
  - `lightRotationZ` in degrees
  - `useShadows`
  - `useSpecular`
  - `fovHorizontal` or `fovVertical` in degrees (also used by `addAnimation.py`, for interactive preview)

- Runtime arguments to `render.py`:
  - `--skipExisting` (`-sk`): do not rerender existing frames in the output directory, frames that have been rendered by earlier sessions

- Large segmentations:

  - If there are _N_ neurons and _N_ is large, try breaking them up into _M_ groups (e.g., by cell type) and show only one (or a few) groups at a time.  There is support in `neuVid` for making this approach easier.
  - In the JSON file's `"neurons"` category:
    - Add `"separate": true`.
    - Make `"source"` and array with _M_ elements (one per group).  The actual sources (array elements) do not need to differ, but there must be _M_ of them.
    - Make each `"neuron"` category key an object with keys `"ids"` and `"sourceIndex"`, the latter referring to an item in the `"source"` array.
  - Then `importMeshes.py` writes each of the _M_ groups in its own Blender file, with the suffix `_neurons_`_i_, where _i_ is from 0 to _M-1_.  Such a Blender file is loaded by `render.py` only when that group is visible.
  - The `--skipExisting` (`-sk`) argument to `importMeshes.py` will reuse all existing `_neurons_`_i_ files without rebuilding them, which can save considerable time if some unrelated part of the JSON file changed (e.g., the set of ROIs).
  - For an example, see `test/test-separate-files-hemi.json`.
  - Made the "Fly Hemibrain Overview" video possible, rendered with Octane

[![Watch the video](https://img.youtube.com/vi/PeyHKdmBpqY/maxresdefault.jpg)](https://www.youtube.com/watch?v=PeyHKdmBpqY)

## Tips

### Using `exponents` and `alpha` for ROIs

The visual appearance of ROIs is a matter of personal taste, but here are some ideas that have worked well in practice.

Under normal circumstances, the `rois` category's `exponents` key is not needed, and an `alpha` of `0.05` looks reasonable for all ROIs:

![Normal ROIs](roisNormal.png)

To emphasize one ROI, give it an `alpha` of `0.2` and deemphasize all the other ROIs by giving them `exponents` values of `8`:

![One emphasized ROI](roisOneEmphasized.png)

To give all ROIs a lighter look, use the `*` key in `exponents` to give everything a value of `7` or `8`.

### Getting `position` values

The `neuVid` input avoids literal positions as much as possible (e.g., `frameCamera` sets the camera position implicitly so some objects fill the view).  Nevertheless, literal positions sometimes are unavoidable (e.g., when the camera view should be filled with just part of an object, with `centerCamera`).  One way to get the coordinates of a literal position is to use [`neuPrintExplorer`](https://neuprint.janelia.org/?dataset=hemibrain%3Av1.0.1): in the skeleton view, shift-click on a particular position on a neuron body sets the camera target to that position.  All that is needed, then, is a way to get that target position from `neuPrintExplorer`.

For better or worse, `neuPrintExplorer`'s current user interface does not present that target position.  The best work-around for now is to get the target position from the URL `neuPrintExplorer` creates for the skeleton view.  Doing so requires a few particular steps, to force the URL to be updated accordingly:

1. Do a "Find Neurons" query for the neuron body in question.
2. Click on "eye" icon to split the query result panel, with the right half being a skeleton view showing the neuron.
3. Close the skeleton viewer in the right half.
4. Go to the "SKELETON" tab that still exists after the "NEUROGLANCER" tab.
5. Shift-click to set the camera target.
6. Go to the "NEUROGLANCER" tab, then immediately back to the "SKELETON" tab.
This switching of tabs makes the `neuPrintExplorer` URL contain a "coordinates" section near the end, with the camera position and target.
7. In a terminal shell run the script to parse that URL and extract the target.  Note: with most shells, the URL must be enclosed in single quotes(') to avoid problems due to the special characters in the URL.
```
python neuVid/parseTarget.py 'https://neuprint.janelia.org/results?dataset...%5D%5Bcoordinates%5D=31501.69753529602%2C63202.63782931245%2C23355.220703777315%2C22390.66247762118%2C24011.276508917697%2C31327.48613433571&tab=2&ftab='
```
9. The camera target position will be printed to the shell, in a form that can be used directly with, say, `centerCamera`.
```
[22391, 24011, 31327]
```
10. To get another target position, shift-click again, and then repeat steps 6 through 9 again.  It is necessary to go to the "NEUROGLANCER" tab and then back to the "SKELETON" tab to force the URL to update with the new camera target.

## Command Summary

*Incomplete*

General command syntax:
```
[ "commandName", { "argumentKey1" : argumentValue1, ..., "argumentKeyN" : argumentValueN }]
```

### `advanceTime`

Required arguments:
- `by`

### `centerCamera`

Required arguments:
- `position`

Optional arguments:
- `fraction` (default: 1)
- `duration` (default: 1)

### `fade`

Required arguments:
- `startingAlpha` or `startingColor`
- `endingAlpha` or `endingColor`
- `meshes`

Optional arguments:
- `duration` (default: 1)
- `stagger` (default: `false`) : `true` means meshes change alpha/color one by one at an accelerating rate; `"constant"` means they change one by one at a constant rate.

The `stagger` argument to `fade` makes meshes appear in the order they are listed.  In the last example, here, the meshes appear in the order their IDs are listed in the file `./ids.txt`:
```json
{
  "neurons": {
    "ascending": [424789697, 5813022341, 612371421, 673509195],
    "descending": [673509195, 612738462, 612371421, 487925037],
    "file": { "ids": "./ids.txt" },
    "source": "https://hemibrain-dvid.janelia.org/api/node/52a13/segmentation_meshes"
  },
  "animation": [
    [ "fade", { "meshes": "neurons.ascending", "startingAlpha": 0, "endingAlpha": 1, "duration": 1, "stagger": "constant" } ],
    [ "advanceTime", { "by": 1 } ],
    [ "fade", { "meshes": "neurons.descending", "startingAlpha": 0, "endingAlpha": 1, "duration": 1, "stagger": "constant" } ],
    [ "advanceTime", { "by": 1 } ],
    [ "fade", { "meshes": "neurons.file", "startingAlpha": 0, "endingAlpha": 1, "duration": 1, "stagger": "constant" } ],
    [ "advanceTime", { "by": 1 } ],
    [ "fade", { "meshes": "neurons", "startingAlpha": 0, "endingAlpha": 1, "duration": 1, "stagger": "constant" } ],
    [ "advanceTime", { "by": 1 } ]
  ]
}
```
Note that in the last `"fade"` command, which operates on all neurons by specifying a general `"meshes": "neurons"` (with no `.` and group name), the order of the neurons is undetermined.

The `sortByBbox.py` script is helpful for specifying a meaningful ordering.  It takes a list of mesh IDs and a directory of mesh files, and produces a list of IDs sorted by various properties of the meshes' bounding boxes.  Its arguments are:
* `-input` (`-i`): the path to the file with the unordered mesh IDs
* `-inputmeshes` (`-im`): the path to the directory with the mesh files
* `-output` (`-o`): the path to the resulting file of ordered mesh IDs
* `--sort min|max|mid|size`: the bounding box property to sort on, where `mid` means center, and `size` means volume
* `--axis 0|1|2`: the axis where 0 means _x_, 1 means _y_, 2 means _z_, to be used with `min`, `max`, or `mid`
* `--descending`: sort in descending order, instead of the default of ascending order

For example:
```
blender --background --python sortByBbox.py -- --input ids-original.txt --inputmeshes neuVidNeuronMeshes --output ids-sorted.txt --sort min --axis 2 --descending
```

### `frameCamera`

Required arguments:
- `bound`

Optional arguments:
- `duration` (default: 1)

### `label`

Required arguments:
- `text`
- `duration`

Optional arguments:
- `size` (default: 0.053): in normalized height units, between 0 and 1
- `position` (default: `"bottom"`): positional string, like `"top"` or `"bottom"`, or an `[x, y]` position, in normalized units, between 0 and 1
- `color` (default: `"white"`)

### `orbitCamera`

Optional arguments:
- `around` (default: the last camera target)
- `axis` (default: `z`): `x`, `y` or `z`
- `localAxis`: an alternative to `axis` giving rotation around the camera's local `x`, `y` or `z` (determined by previous orbits)
- `endingRelativeAngle` (default: -360)
- `duration` (default: 1)
- `scale` (default: 1) : non-default values give a "spiral" effect

### `poseCamera`

Required arguments:
- `target` : corresponds to Neuroglancer view state `position`
- `orientation` : corresponds to Neuroglancer view state `projectionOrientation`
- `distance` : corresonds to Neuroglancer view state `projectionScale`

Optional arguments:
- `duration` (default: 0) : note that animation to a pose, or between poses, can loop around in surprising ways

### `pulse`

Required arguments:
- `meshes`

Optional arguments:
- `toColor` (default: #ffffff)
- `duration` (default: 1)
- `rate` (default: 1)

### `setValue`

Required arguments:
- `meshes`
- `alpha` or `color` or `exponent` (for ROI silhouette shading) or `threshold` (for ROI transparent depth)

### `showPictureInPicture`

Required arguments:
- `source`

Optional arguments:
- `duration` (default: 3)

### `showSlice`

Required arguments:
- `source`
- `bound` or `euler`

 Optional arguments:
- `duration` (default: 3)
- `fade` (default: 0.5)
- `delay` (default: `fade` value)
