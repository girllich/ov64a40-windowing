# Windowed readout on the OV64A40 (Arducam OwlSight 64MP)

The OmniVision OV64A40 has no public datasheet, and its Linux driver offers six fixed
modes and reports "Cropping: Not Supported". The sensor itself does much more. These are
measured results from one camera, taken in November 2026. Other units or sensor
revisions may differ.

- It outputs a window of any size at any position, in its full-resolution, 2×2 and 4×4
  readouts.
- A small window is fast: 2304×1728 at full resolution runs at about 34 fps, against
  2.6 fps for the whole frame. The limit is the 2-lane MIPI link.
- The lens focuses under manual control, and a plain contrast score finds the sharpest
  position.

## Test setup

- Arducam OwlSight 64MP (B0483), OV64A40, 9248×6944, 2 MIPI lanes.
- Raspberry Pi 5, kernel 6.18.60+rpt-rpi-2712, libcamera 0.7.2+rpt20261009, rpicam-apps 1.14.0.
- 456 MHz link frequency (912 Mbps per lane), 10-bit raw.
- The kernel's `drivers/media/i2c/ov64a40.c` with one added mode whose geometry comes
  from module parameters. It writes only registers the stock driver already writes, with
  different geometry values.
- Results are from raw frames, checked by looking at the image and by locating it
  inside a full-frame capture. The first tests went through libcamera (`rpicam-still`,
  `rpicam-vid`); the later ones through `v4l2-ctl` directly, which is the cleaner way.

## Geometry registers

From the stock driver's `ov64a40_program_geometry()`:

| Register | Meaning |
|---|---|
| 0x3800, 0x3802 | Analogue window start X, Y (sensor pixels) |
| 0x3804, 0x3806 | Analogue window end X, Y (inclusive) |
| 0x3808, 0x380a | Output width, height (output pixels) |
| 0x3810, 0x3812 | ISP window offset X, Y (output pixels, inside the analogue window) |
| 0x380c | Line length. Line time = value × 4 / 300 MHz |
| 0x380e (+0x3840) | Frame length in lines |
| 0x3814, 0x3815 | X, Y skip increments (odd << 4 \| even) |
| 0x3820 bit 1, 0x3821 bits 4 and 0 | Vertical, horizontal binning |

The analogue array is 9280×6976. The stock modes keep a 32-pixel margin: the analogue
window is 32 sensor pixels wider and taller than the output area, and the ISP offset is
17, 16 (full resolution), 9, 8 (2×2) or 5, 4 (4×4).

## What works

| Way to shrink the window | Result |
|---|---|
| ISP window: offset + output size | Works at every size and position tried, 640×480 to 7984×1728, in all three readouts |
| Fewer rows: analogue Y start/end | Works. Frame time scales with the row count |
| Narrower rows: analogue X start/end | Works down to 5280 wide. No frames at 4640, 3872, 2336; corrupt data at 1312. Same in all three readouts, with or without 0x5000 = 0x01. Cause unknown |
| 2×2 readout (V-binning, H-skip) plus a window | Works: 2304×1728 output over a 4608×3456 sensor area |
| 4×4 readout (binning + skip) plus a window | Works: 1152×864 output over a 4608×3456 sensor area |

### Recipe

For an output of `w × h` pixels at scale `s` (1, 2 or 4), covering sensor pixels
starting at `(left, top)`:

1. Write the common init list, then the stock register list of the readout family
   (9248×6944, 4624×3472 or 2312×1736 list), then its skip and binning settings.
2. Analogue window: X 0 to 9279 (full width). Y `top` to `top + h·s + 31`.
3. ISP offset: X = base + `left / s`, Y = base, with base 17, 16 / 9, 8 / 5, 4.
4. Output size `w × h`.
5. For full resolution and 2×2: write **0x5000 = 0x01** (see below).
6. Line length for the link (next section). Frame length `h + 288` lines worked.

With `left` and `w` multiples of 16 and `top` and `h` multiples of 8, the Bayer order
stayed the same as in the stock modes.

### The second frame: 0x5000

A window narrower than the analogue readout delivers its **first frame** correctly and
then misbehaves, in the full-resolution and 2×2 readouts:

| Output width vs analogue width | Frames after the first |
|---|---|
| At most half | Every second frame is black, and frame timing slips by the exposure time plus 2.4 ms each cycle |
| More than half, less than nearly all | The sensor stops sending; the Pi times out and restarts the stream about once a second |
| Stock sizes, and any width in 4×4 | Normal |

Still captures hide this, because they use the first frame. It is not the Pi: unencoded
frames show it, at every frame rate, line length, position and row count, cold or warm.

Writing **0x5000 = 0x01** cures it. That is the value the stock driver's cropped
3840×2160 list writes (the other full-resolution and 2×2 lists leave it at 0xff; the 4×4
list writes 0x55). With it, every window tried streams continuously at the requested
rate, 640×480 to 4608×3456, including 2304×1728 at 34 fps. The image is the same to the
eye, with slightly lower minimum values and a little more grain, so the bits it clears
look like corrections that need full-width lines.

### Position

With libcamera's default orientation, measured at three positions to ±16 pixels:

- Image x = `left`.
- Image y = 6944 − `top` − `h·s`. The analogue row address counts from the bottom of
  the image libcamera delivers.

The three readouts show the same area for the same `left` and `top`.

## Speed

Line time is set by the MIPI link, not by the sensor's column readout, even though the
sensor is reading all 9280 columns. A row of `W` pixels takes `W × 10 / 1824` µs on
2 lanes at 912 Mbps. Frames were clean when the line time was at least about 1.15× that:

| Output width | Line length (0x380c) | Line time | Result |
|---|---|---|---|
| 2304 | 1100 | 14.7 µs | clean |
| 2304 | 1000 | 13.3 µs | torn rows |
| 2304 | 800 | 10.7 µs | garbage |
| 2304 | 600 | 8.0 µs | no frames |
| 3840 | 1800 | 24.0 µs | clean |
| 3840 | 1600 | 21.3 µs | torn rows |
| 4608 | 2200 | 29.3 µs | clean |
| 4608 | 2000 | 26.7 µs | torn rows |

So a 2304×1728 full-resolution window runs at 34 fps (measured, continuous),
against 2.6 fps for the stock full-resolution mode. Because line time doesn't depend on
the analogue width, narrowing the analogue window in X gains nothing.

Measured continuously, without libcamera, at 288 blanking lines:

| Window | Line length | Frame rate |
|---|---|---|
| Full resolution 2304×1728 | 1100 | 33.8 fps |
| 2×2, 2304×1728 (4608×3456 on the sensor) | 2112 (stock) | 17.6 fps |
| 4×4, 1152×864 (4608×3456 on the sensor) | 1248 (stock) | 52 fps |

The 2×2 and 4×4 line lengths are the stock ones; shorter ones were not tried there.

## Fast starts and moving the window

The stock driver writes its 4376-register init list on every stream start (1.8 s) and
powers the sensor off one second after a stream stops. Neither is needed between looks.

| Start | Register writes |
|---|---|
| Cold: full init list | 1810 ms |
| Warm, same readout: geometry only | 21 ms |
| Warm, any readout: mode-reset list, then the mode's list | 60 ms (full), 86 ms (2×2), 95 ms (4×4) |
| Move a streaming window (same size and readout) | 6 writes, no restart |

- **Keep the sensor powered** and skip the init list. With the same readout as before,
  only geometry and controls need writing, and the frame is identical to a cold start.
- **Changing readout needs a reset list.** The per-mode lists don't undo each other
  (the ROP-screening project saw magenta stripes from this). Writing every register that
  any mode list touches, at its init value, and then the wanted mode's list gives frames
  identical to a cold start for full → 4×4 → full → 2×2 → full.
- **Four registers are a trap: 0x3a5e, 0x3a6d (0xdc) and 0x3a60, 0x3a6f (0x68).** The
  init list writes them several times and its last write is 0xcd, but writing 0xcd
  afterwards gives an inverted, saturated image. The values that work are the ones the
  2×2 list and ROP-screening use. The same addresses evidently reach different storage
  at different points of the init sequence.
- **Ten registers the binned lists write are never set by the init list** (0x5200–0x5203,
  0x5280–0x5285). Leaving them at the binned values did not affect full-resolution frames.
- **A streaming window can be moved** by writing the analogue start/end (0x3800–0x3807)
  and the ISP offset (0x3810, 0x3812) with the stream running. Seen frame by frame at
  18 fps: one corrupted frame at the switch, then every frame at the new position, with
  no change in frame timing. Discard one frame after a move.
- **The output size can change live too.** Halving the height and restoring it with the
  stream running kept frame timing constant, with one bad frame at each change. The Pi's
  receiver keeps its original buffer size, so this was only confirmed from the rows that
  did arrive.
- **Keep the exposure shorter than the frame.** With an exposure longer than the frame
  length, cold and warm starts gave different brightness.

## Focus

The module has a voice-coil lens driven by a BU64754 at I2C address 0x76. Its kernel
driver has one control, `focus_absolute`, 0 to 1023. libcamera exposes it as a lens
position in dioptres (1/metres), range 0 to 12 in the tuning file:

    rpicam-still --autofocus-mode manual --lens-position 4.8 ...

One dioptre is 68 steps of `focus_absolute` (lens position 4.8 = 327).

Sharpness of a 2304×1728 window, scored as the variance of the Laplacian of one green
plane, against lens position, for a subject about 20 cm away (distance inferred from the lens
position, not measured):

| Lens position | 0 – 4 | 4.5 | 4.8 | 5.0 | 5.2 | 5.4 | 5.6 | 6 – 12 |
|---|---|---|---|---|---|---|---|---|
| Score | 160 – 260 | 850 | 2246 | 1871 | 1486 | 670 | 334 | 170 – 185 |

- The peak is narrow at this distance: about ±0.3 dioptre.
- The lens is fast. Stepping across a third of its range (327 steps) during a 34 fps
  stream gives one transitional frame and a settled image by the second, in both
  directions: under 60 ms.
- libcamera owns the lens while it runs. In manual mode it rewrites its position every
  frame, and with no lens options its autofocus moves it, so a direct `focus_absolute`
  write only holds when libcamera is not in the pipeline.

Because the stream is the window, a contrast score over the frame is a score of exactly
the region of interest.

## Capturing without libcamera

All of the above can be driven with `media-ctl` and `v4l2-ctl` alone, which removes the
preview pass, the lens and exposure meddling and the denoiser from the picture, and gives
exact frame timing (39 of 39 intervals at 29.57 ms for a 2304×1728 window).
The route is sensor → `csi2` pad 4 → `rp1-cfe-csi2_ch0` (`/dev/video0`), pixel format
`RG16`, bypassing the Pi's front end.

The Pi 5 receiver driver has three traps on this path:

- It starts the sensor only when every node it believes is enabled is streaming, and it
  believes its front end is in use until a `csi2` → `pisp-fe` link *changes*. Both beliefs
  survive from earlier libcamera runs. Switch every unused link on and off again first, or
  `STREAMON` succeeds and no frame ever arrives.
- `media-ctl` keeps only the last `-V` option. Put all formats in one comma-separated `-V`.
- Give every format `field:none`, or the pipeline refuses to start (-EPIPE).

Set controls after the format (setting a format resets blanking and the exposure range),
turn both flips on to match libcamera's orientation, and drop the first frame after a
start: it is darker than the rest.

## Raw data and colour

- With both flips on (libcamera's default orientation for this module) the Bayer order
  is RGGB: red at (0,0), green at (0,1) and (1,0), blue at (1,1), in all three readouts.
- `RG16` frames from the Pi hold 10-bit samples in the top bits of each 16-bit word:
  white is 65472, and the black level is 4096 (64 in 10 bits).
- Raw frames look green and flat until they are processed. Subtract the black level,
  apply white-balance gains, demosaic, apply a colour matrix, then gamma.
- The Pi's libcamera tuning file for the sensor (`ipa/rpi/pisp/ov64a40.json`) has what is
  needed: a curve of grey R/G and B/G ratios against colour temperature, and a colour
  matrix per temperature. Under white LED lighting, a grey-world estimate snapped to that
  curve gave about 5700 K, gains R 2.07 and B 1.44, and plausible colours. Not checked
  against a colour reference.

## Things that will mislead you

- **`rpicam-still` starts the sensor twice.** Without `--immediate` it runs a preview
  first, in a sensor mode it chooses by score, then restarts for the still. Which mode
  it chooses depends on the size requested, and some windows then come back as a
  constant two-value frame. Use `--immediate`, or force the preview into the same mode
  with `--viewfinder-mode`, and count stream starts in the kernel log.
- **`camera_auto_detect=1` doesn't find this sensor** on Raspberry Pi OS. It needs
  `dtoverlay=ov64a40` (add `,cam0` for the other port), which also works at runtime.

## The module

- I2C devices on the camera connector: 0x36 (sensor), 0x76 (BU64754 focus driver),
  0x50 and 0x21 (not identified).
- The 15-pin end of the supplied cable goes to the camera. The 22-pin end fits the
  Pi 5 with its contacts facing the Ethernet jack.

## Sources

- Linux driver: `drivers/media/i2c/ov64a40.c` (Jacopo Mondi), and its linux-media review thread.
- [teokakabadze/ROP-screening](https://github.com/teokakabadze/ROP-screening): hidden
  per-mode state, and MIPI timing registers that must not be written while streaming.
