# Atlas Packer

**Version:** 1.0.0  
**Host:** Photon Studio  
**Package:** `Atlas-Packer-1.0.0-en.photon-plugin`

Atlas Packer turns selected layers into a compact, transparent RGBA texture atlas—or rearranges the original layers directly on the document canvas. It is designed for layered illustration, foliage, sprites, game assets, UI elements, and other artwork that benefits from a single texture sheet.

The plugin combines non-destructive layer capture, rectangle-based packing, optional alpha-silhouette packing, rotation controls, uniform auto-scaling, edge extension, and a native **Pack In Place** workflow. Its goal is to reduce repetitive manual arrangement while keeping the source artwork editable.

## Highlights

- **Create PNG Atlas** — capture the selected artwork and create a new RGBA atlas document.
- **Pack In Place** — rearrange the original layers on the current canvas without creating duplicate layers or merging the artwork.
- **Three rotation modes** — no rotation, 90-degree rotation, or Free Angle rotation.
- **Alpha-mask packing** — optionally pack around opaque silhouettes so transparent regions can interlock instead of reserving every sprite's full rectangle.
- **Uniform Auto Scale** — scale all sprites down by one common factor when the chosen atlas dimensions cannot contain them at their current size.
- **Padding and Edge Extension** — control the clearance between sprites and extend edge pixels into the padding to help avoid filtering artifacts.
- **Transparency-aware trimming** — trim transparent borders around captured sprites, with an optional alpha threshold.
- **Trim output to content** — optionally crop the resulting PNG canvas to the packed bounds.
- **Layout JSON export** — save sprite coordinates, rotations, scale, settings, and packing statistics for downstream tools or engine integration.
- **Visible-layer selection helper** — select visible non-group layers without manually selecting each one.

## How it works

### Create PNG Atlas

1. Select the layers to include in the atlas.
2. Set the maximum atlas width and height, spacing, transparency, rotation, and output options.
3. Click **Create PNG Atlas**.
4. Atlas Packer captures temporary renders of the selected layers inside a Photon transaction that is deliberately rolled back. It then packs the captured pixels and creates a new atlas document.
5. The source document is not flattened or rasterized by the PNG capture workflow.

The generated atlas is an RGBA image. Save or export the new document from Photon Studio using the format and destination required by your pipeline.

### Pack In Place

**Pack In Place** uses the same packing settings and calculated layout, but applies the resulting positions, scale, and rotations to the existing source layers through Photon-native layer transforms.

- It does not create duplicate layers or merge the selected artwork.
- It does not resize the document canvas.
- It honors the selected Auto Scale and rotation settings, subject to the current document bounds.
- It checks the document structure and layer types before committing the transaction. If the required operation is unavailable or a safety check fails, the operation is cancelled or rolled back rather than silently changing the layer structure.

Pack In Place is not available for selections with clipping-mask relationships that the plugin cannot safely preserve. Some Photon builds or layer types may also lack the native transform operations required by a particular layout.

## Packing options

### Rotation

| Option | Behaviour |
|---|---|
| **Off** | Sprites remain at their original orientation. |
| **90°** | Sprites may remain unrotated or rotate by 90 degrees. |
| **Free Angle** | Sprites may be rotated to other angles using the plugin's bounded angle search and local refinement. |

Free Angle automatically uses alpha-mask packing. Its angle search is a heuristic, not an exhaustive search over every possible real-valued angle.

### Alpha-mask packing

Alpha-mask packing uses the sprite's non-transparent silhouette for collision checks. Transparent portions of the bounding rectangle can therefore overlap empty portions of other sprites, which can improve packing for leaves, cut-out shapes, and irregular artwork.

- In **Off** and **90°** modes, alpha-mask packing is optional and controlled by its checkbox.
- In **Free Angle** mode, alpha-mask packing is enabled automatically.
- When the option is off in Off/90° modes, the packer uses rectangular collision bounds.

The alpha silhouette is based on non-zero alpha in the trimmed sprite. The optional alpha threshold described below controls the trimming bounds; it is not a global opacity adjustment.

### Auto Scale

When **Auto Scale** is off, the plugin attempts to pack the sprites at their captured size. If they do not fit, packing reports an error.

When **Auto Scale** is on, the plugin searches for a single uniform scale factor that allows the selected sprites to fit. It scales down only; it does not enlarge sprites beyond their captured size. The same scale factor is applied consistently to every sprite.

The scale search uses heuristic packing results and pixel-rounded sprite dimensions. It aims to use the available atlas area effectively, but it cannot guarantee a mathematically optimal layout for every collection of shapes.

### Padding

**Padding** is the reserved spacing between neighboring sprites. It helps keep samples from adjacent sprites separate when the atlas is filtered or mipmapped.

### Edge Extension

**Edge Extension** copies edge colors outward into the sprite's padding area. It does not increase the source sprite's resolution or change the packing scale.

| Mode | Behaviour |
|---|---|
| **RGB only** | Extends edge color while keeping the added pixels transparent. |
| **RGBA** | Extends color and alpha, making the extended edge pixels visible. |

Edge Extension must not exceed half of the Padding value. It is most useful when padding is available and the target renderer uses texture filtering or mipmaps.

### Alpha threshold

By default, the alpha threshold option is off. In that state, pixels with alpha greater than zero define the trim bounds.

When enabled, the selected threshold (0–255) is used when finding the non-transparent bounds of each sprite. Pixels at or below the threshold are ignored when calculating those bounds. This option is intended to remove faint transparent-edge noise; it does not apply a color or opacity correction to the remaining sprite pixels.

### Trim output to content

When enabled, the PNG atlas canvas is cropped to the rightmost and bottommost packed bounds rather than always using the full requested dimensions. This option affects the generated PNG canvas. Pack In Place does not resize the Photon document canvas.

### Visible-layer selection

The optional **Select all visible non-group layers** control is a selection helper. It asks Photon to select visible layers and excludes organizational containers such as groups when their child layers are present. It changes the active selection, not the artwork pixels.

## Controls and defaults

The following defaults are used when the plugin instance is activated:

| Setting | Default |
|---|---:|
| Maximum atlas width | 4096 px |
| Maximum atlas height | 4096 px |
| Padding | 16 px |
| Edge Extension | 8 px |
| Edge Extension mode | RGB only |
| Alpha threshold | Off (threshold value: 4) |
| Rotation | Off |
| Alpha-mask packing | Off for Off/90°; automatic for Free Angle |
| Auto Scale | Off |
| Trim output to content | Off |
| Select all visible non-group layers | Off |

## Limits and considerations

- Atlas width and height must each be integers from **64 to 8192 px**.
- The total atlas area is limited to **16,777,216 pixels** (16 megapixels). The per-axis maximum does not mean that an 8192 × 8192 atlas is accepted.
- Padding must be an integer from **0 to 256 px**.
- Edge Extension must be an integer from **0 to 128 px**, and must not exceed half the Padding value.
- Alpha threshold must be an integer from **0 to 255**.
- Empty sprites are skipped. If all captured sprites are empty, the atlas cannot be created.
- Packing is heuristic. Complex silhouettes, many sprites, or difficult rotations can leave unused space; the plugin does not promise a globally optimal solution.
- Free Angle evaluates a bounded set of candidate angles and locally refines promising candidates. A different angle search may produce a different layout, and more exhaustive search would cost additional time.
- Pixel capture depends on the Photon host's pixel-transfer limits. Very large layers or documents may exceed the available transfer buffer. When this happens, try processing fewer layers or using a smaller document copy.
- Pack In Place requires Photon-native translation and/or affine-transform operations. If the host does not expose the required operation, the plugin stops without applying the layout.
- Pack In Place currently rejects selections with clipping-mask relationships that cannot be safely maintained.
- Plugin options are initialized from the built-in defaults when the plugin instance starts. This release does not write the options to Photon’s settings store, so do not rely on them being remembered after the plugin instance is restarted.

## Suggested workflow

1. Work on a copy of the document when experimenting with unfamiliar layer types or complex clipping relationships.
2. Select only the artwork that belongs in the atlas.
3. Choose the target atlas dimensions and set Padding before adjusting Edge Extension.
4. Start with **Rotation: Off** for predictable orientation. Enable **90°** or **Free Angle** when additional packing flexibility is useful.
5. Try alpha-mask packing for irregular silhouettes with large transparent regions.
6. Enable Auto Scale only when the current sprite dimensions do not fit the target atlas.
7. Use **Create PNG Atlas** to produce a separate atlas document, or **Pack In Place** when you specifically want to rearrange the original layers.
8. Export the layout JSON when you need to reuse the generated sprite coordinates in another tool or pipeline.

## Intended use

Atlas Packer is intended for artists and technical artists who prepare texture atlases from editable Photon layer stacks. It is particularly useful when a collection contains many separate sprites, transparent or irregular silhouettes, or assets that need repeatable spacing and rotation rules.

It is a packing assistant—not a replacement for reviewing the final texture in the target engine. Always inspect the generated atlas at the intended texture-filtering and mipmap settings before shipping it.

---

**Atlas Packer 1.0.0** — keep your layer-based artwork editable, and turn it into a compact atlas when you are ready to ship.
