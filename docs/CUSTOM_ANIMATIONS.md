# Custom Peripheral Animations

Add your own animation to the peripheral display without touching any C code
in the module. Custom animations use the exact same mechanism as the built-in
ones (cat, spaceman, pokemon…): LVGL `lv_animimg` cycling through
`LV_IMG_CF_INDEXED_1BIT` frames stored in flash.

This is the cheapest animation mechanism available on this hardware:

- **Flash only**: frames are `const` arrays, read directly from flash — no
  runtime decoding, no extra RAM beyond LVGL's draw buffer.
- **1 bit per pixel**: a 48x48 frame is ~300 bytes.
- **CPU cost is set by the frame rate**, not the frame count: LVGL redraws
  only the animation's area, `frames / CONFIG_..._MS` times per second.

Avoid runtime GIF decoding (`LV_USE_GIF`): it needs the whole GIF plus a
decode buffer in RAM and burns CPU per frame. Pre-converted frames are
strictly better on an nRF52840.

## Pipeline

### 1. Get or make a GIF / PNG frames

Any monochrome-friendly art works best. Keep it:

| Property | Recommended | Why |
|----------|-------------|-----|
| Size | ≤ 68 px wide (canvas is 68x160) | Fits the portrait canvas |
| Frames | 4–16 | Flash + smoothness sweet spot |
| Frame rate | 4–10 fps | Above ~10 fps the OLED bus and CPU pay for invisible smoothness |

### 2. Convert

```sh
python3 scripts/gif2anim.py my_art.gif --name my_art --width 48 --fps 8
```

This writes `boards/shields/nice_oled/assets/user_animation.c` and prints the
config lines to add. Options: `--dither` (Floyd–Steinberg for shaded art),
`--threshold N` (black/white cutoff), `--invert`, `--height N`.
PNG sequences also work: `gif2anim.py frame*.png ...`.

The output format is byte-compatible with the official LVGL image converter
(<https://lvgl.io/tools/imageconverter>, color format `CF_INDEXED_1_BIT`,
LVGL v8). If you prefer that tool, convert each frame there and add the
`user_anim_imgs[]` array and `user_anim_imgs_count` yourself — see the bottom
of a generated `user_animation.c` for the shape.

### 3. Enable

In your keyboard `.conf`:

```ini
# disable whichever built-in animation was active, then:
CONFIG_NICE_OLED_WIDGET_ANIMATION_PERIPHERAL_USER=y
# one full cycle in ms — frames / (MS/1000) = fps
CONFIG_NICE_OLED_WIDGET_ANIMATION_PERIPHERAL_MS=1000
```

Position it with `CONFIG_NICE_OLED_WIDGET_ANIMATION_PERIPHERAL_CUSTOM_X/_Y`
if needed.

## Budgeting CPU and memory

- **Flash** ≈ `frames × (8 + height × ceil(width / 8))` bytes. The converter
  prints the total. Keep the whole animation under ~20 KB; the peripheral
  build has plenty of headroom (~65% flash free) but restraint keeps builds
  fast and leaves room for other widgets.
- **RAM**: constant — frames render straight from flash. Nothing to tune.
- **CPU / battery**: proportional to redraws per second
  (`frames / seconds-per-cycle`) and the frame area. Doubling
  `..._PERIPHERAL_MS` halves the animation's CPU and display-bus load. If the
  peripheral's battery drains noticeably faster, raise `MS` first, then
  shrink the frame size.
