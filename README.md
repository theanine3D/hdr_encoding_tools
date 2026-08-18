# HDR Encoding Tools

<img width="243" height="233" alt="image" src="https://github.com/user-attachments/assets/2a7f37b2-484d-47c6-9181-fcee7563bfc9" />...<img width="242" height="422" alt="image" src="https://github.com/user-attachments/assets/19b6b423-835e-447c-876b-6418b69164be" />


HDR Encoding Tools is a Blender 5.x addon that can prepare baked HDR light for use in game engines like Unity. It can convert an .HDR or .EXR image to a PNG with RGBM or dLDR encoding. It can also compress and clean up HDR vertex colors.

RGBM and dLDR encodings are used by game developers to reduce file size of lightmaps. RGBM can cut the file size of a HDR lightmap by at least half, and dLDR can cut it down even further (albeit with further lossiness.) You can read more about these encodings in [Unity's documentation](https://docs.unity3d.com/Manual/Lightmaps-TechnicalInformation.html).

HDR vertex colors have a significantly lower memory footprint than HDR images. However, Unity normally will clamp vertex colors to the 0.0 - 1.0 range, discarding all values above 1.0.  This addon's Compress feature will divide all light values by a specific compression factor, which then allows their safe export to Unity where they can be re-multiplied back to their intended value via a shader.

Finding this addon useful? Please consider starring it ⭐, or [donating](https://ko-fi.com/theanine3d) 🙂<br>

## Installation
1. Press the big green Code button above and choose "Download ZIP"
2. Open Blender Preferences and click on the "Addons" tab
3. Click on the "install" button and select your newly downloaded ZIP

---

## How to Use (Image-based)
First follow Blender's usual steps for baking light to an HDR image format (.hdr / .exr). Once you've saved your HDR lightmap file, follow these steps:

- Go into the UV/Image Editor and open the right sidebar (ie. press N)
- Click on the "HDR Encoder" tab. Choose your .EXR or .HDR image
- Press "Generate PNG"
- After a moment, your new PNG will appear in the UV/Image Editor automatically.
- No need to save the new image manually - the addon also saves it the same folder as your chosen .EXR/HDR image.

### Encodings

Both PNG encodings store gamma-encoded (1/2.2) values, following Unity's
lightmap conventions:

| Encoding | Gamma range | Linear range | Alpha channel |
|----------|-------------|--------------|---------------|
| RGBM     | [0, 5]      | [0, 34.49]   | Multiplier (M) |
| dLDR     | [0, 2]      | [0, 4.59]    | Unused (1.0)   |

RGBM stores the color divided by its max component; the multiplier that
restores it goes in the alpha channel. dLDR simply maps [0, 2] to [0, 1];
intensities above 2 are clamped.

### Decoding in Unity (Shader Graph, linear color space)

Import the PNG with **sRGB unchecked** (and for RGBM, **Alpha Is
Transparency** unchecked — alpha is a multiplier, not coverage), then:

- **RGBM**: `pow(RGB × A × 5, 2.2)`
- **dLDR**: `pow(RGB × 2, 2.2)`

In a gamma-space project, drop the `pow` and just multiply.

---

## How to Use (Color-based)

This method is the best for filesize / memory savings. The catch: Unity
clamps vertex colors to [0, 1] on FBX import, so HDR values must be
compressed into range first.

Typical workflow:
1. Open the "HDR Encoding Tools" panel in the 3D Viewport's righthand sidebar (press N).
2. In the 3D Viewport, select the meshes to bake light for
3. Press the "Create Vertex Color Layer" button in the addon's sidebar panel.
4. Set your Bake target (in the "Render Properties" panel) to "Active Vertex Color" (under Bake -> Output). Also set Bake Type to "Diffuse", and Influence to "Direct" + "Indirect" (leave "Color" disabled).
5. If your meshes have any transparency, press the "Unplug for Bake" button in the HDR Encoding Tools sidebar panel in the 3D Viewport.
6. Press Blender's built-in "Bake" button and wait for the bake to complete.
7. If you had previously pressed the "Unplug for Bake" button, press the "Restore Connections" button
8. General cleanup step: press the "Fix Buried Vertices" button, followed optionally by the "Smooth Vertex Colors" button
9. Press the "Compress HDR Vertex Color" button. You will notice the vertex colors become darker after doing this - this is normal.
10. Export your mesh as FBX. Make sure that you set the "Vertex Colors" setting to "Linear" in your FBX export settings.
    
Keep the compressed values if you plan to re-export; use **Decompress
and Restore HDR** when you want to preview or re-bake the true HDR
values in Blender.

### Transparency

Baking light onto a mesh with a transparent texture (foliage, leaves,
fences, etc.) is a problem: the transparent parts of the surface have no
light, so they bake to black, and that black then bleeds across the mesh
through vertex-color interpolation — often turning whole meshes dark. The
**Transparency** section automates the usual workaround of making the
material temporarily opaque for the bake:

- **Bake Color** — the solid Base Color transparent materials are set to
  during the bake (default green, which suits plants). Set it to whatever
  best matches the meshes you're baking.
- **Unplug for Bake** — on every *transparent* material of the selected
  meshes (Principled BSDF with a linked or below-1.0 Alpha), disconnects
  the Base Color and Alpha inputs and sets them to the Bake Color and
  fully opaque. Fully opaque materials are left alone. Bake your light after pressing this button.
- **Restore Connections** — reconnects the Base Color and Alpha inputs
  exactly as they were, restoring the original albedo and transparency.
  Press this button **after** baking your light.
  
The original links and values are stored on the material nodes
themselves, so Restore works reliably even after saving and reopening the
file. Shared materials are handled once. Only Principled BSDF Alpha is
covered — Mix-Shader / Transparent-BSDF transparency setups aren't
touched.

### Cleanup

HDR Encoding Tools also has some cleanup features for light that was baked to
vertex colors. 

- **Fix Buried Vertices** — repairs "shadow bleed": if vertices in a mesh
   are buried even slightly inside other geometry, the light value there bakes
   to black or near-black, and interpolation then smears that darkness up
   the sides of the mesh. This button fixes that by copying the nearest
   non-black color to those vertices that were completely buried.
- **Find Buried Islands** — finds and highlights geometry islands that are
   completely buried - and as a result, completely darkened by shadows
- **Smooth Vertex Colors** — blurs the active color attribute of every
  selected mesh in one click, averaging each vertex's color with its
  connected neighbours. Press it repeatedly for a stronger smoothing effect.
  Handy after Fix Buried Vertices to blend the repaired areas in.
