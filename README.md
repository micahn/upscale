# upscale

Local AI image upscaler for Linux. One bash file, no Python, no system packages.

Wraps [Real-ESRGAN](https://github.com/xinntao/Real-ESRGAN)'s ncnn/Vulkan build —
the same model Upscayl uses — and gives it named and exact output sizes.

```
upscale 4K shot.png           # -> shot-4k.png at 3840x2160
upscale 4KUW a.png b.png      # -> 5120x2160, batch
upscale 800x600 shot.png      # exact pixels
upscale 1080 ~/pics/*.jpg     # globs expand, as in any shell
```

## Install

```sh
git clone https://github.com/micahn/upscale
install -Dm755 upscale ~/.local/bin/upscale
```

Requires [mise](https://mise.jdx.dev) and ImageMagick 7 (`pacman -S imagemagick`).
Any Vulkan-capable GPU works; tested on AMD via radv.

The first run installs the model itself (~47MB) and prints when it's done.
Later runs are instant.

## Sizes

| keyword    | output    | keyword  | output    |
|------------|-----------|----------|-----------|
| `720`      | 1280x720  | `720UW`  | 1680x720  |
| `1080`     | 1920x1080 | `1080UW` | 2560x1080 |
| `1440`,`2K`| 2560x1440 | `2KUW`   | 3440x1440 |
| `4K`       | 3840x2160 | `4KUW`   | 5120x2160 |
| `8K`       | 7680x4320 | `8KUW`   | 7680x3240 |

Case doesn't matter, and `-` or `_` are ignored: `4K`, `4k`, `4kUW`, `4K-UW`, `4k_uw`
all work. Anything matching `WxH` is used verbatim, so `800x600` is exact.

## Options

```
-o PATH   output file, or a directory when you pass several inputs
-m NAME   which Real-ESRGAN model (default: realesrgan-x4plus)
models    list, install or remove models
-f FMT    output format: png jpg webp tiff (default: png)
-q N      encoder quality, 1-100 (lossy formats)
-g ID     GPU device id, as seen by the Vulkan loader (default: auto)
-t SIZE   tile size in pixels; lower it to fit large images
-x        test-time augmentation: slower, slightly sharper
-j N      run N files at once (default: 2; VRAM, not cores, is the limit)
-y        proceed even when the source is too small to reach the target
selftest  check the size and scale logic, no GPU needed
uninstall remove the sandbox and this launcher
```

Default output is `<name>-<size>.png` next to the input.

With `-o` and several inputs, outputs are written flat into that directory, named
after each input's basename. Since that makes a clash possible, `upscale` checks
the whole batch first and refuses to run if two inputs would land on the same
file — before spending any GPU time on a result that would be overwritten:

```sh
upscale 4K a/photo.png b/photo.png -o out/
# upscale: two or more inputs resolve to the same output:
#   out/photo-4k.png  <-  a/photo.png and b/photo.png
#   Nothing was written. Rename them, or run them as separate batches.
```

Every input in a batch is attempted even if an earlier one fails, so an
unmatched glob partway through a list doesn't cost you the whole run. Failures
are reported per file, and `upscale` still exits non-zero if anything failed,
so scripts notice:

```sh
upscale 1080 ~/pics/*.jpg -o out/
# wrote out/a-1080.png
# no such file: /home/you/pics/dSC0001.JPG  (did your glob match anything?)
# wrote out/b-1080.png
# finished with errors; some files were skipped (see above)
```

### Batches

A batch runs two files at a time by default. Real-ESRGAN is a single-image
model, so the parallelism is whole images rather than a batched forward pass —
which means **VRAM, not cores, is the limit**. Each concurrent job holds a full
working set, so `-j nproc` will run out of memory long before it runs out of CPU.

```sh
upscale 4K ~/pics/*.jpg -j 1      # one at a time, for a small GPU
upscale 4K ~/pics/*.jpg -j 4      # if you have the VRAM for it
```

Measured here on four 700x500 sources to 4K: 25.9s at `-j 1`, 14.6s at `-j 2`.
`-j 4` was no faster than `-j 2` — the GPU is already saturated. If a job will
not fit at all, `-t` (tile size) is the lever.

Set `UPSCALE_JOBS` to change the default.

Output is printed as each file finishes rather than in input order, but the lines
for any one file stay together.

### Output format

Output is PNG by default, which is lossless but large — a detailed 4K result is
around 8.6MB. `-f` writes another format, and `-q` sets the encoder quality:

```sh
upscale 4K photo.jpg -f jpg -q 92      # ~1.3MB
upscale 4K photo.jpg -f webp          # ~150KB
upscale 4K photo.jpg -f tiff          # lossless, no PNG-style file size
```

`png`, `jpg`, `webp` and `tiff` are supported. Without `-q` each encoder uses its
own default.

### Models

`realesrgan-x4plus` is the default. It is tuned for general photographic
material, so it does not push anything towards a particular style. The other
two are tuned for anime and illustration, and `upscale` will say which is which:

```sh
upscale models                              # what is installed, and what is not
upscale models install realesrgan-x4plus    # fetch it from the pinned release
upscale models remove realesr-animevideov3  # reclaim its disk
upscale 4K photo.jpg -m realesr-animevideov3
```

| model | style | size | note |
|---|---|---|---|
| `realesrgan-x4plus` | photo | 31M | general purpose; the default |
| `realesrgan-x4plus-anime` | anime | 8M | illustration and line art |
| `realesr-animevideov3` | anime | 3M | video frames; smallest and fastest |

All three honour `-s 2`, `-s 3` and `-s 4`; only the x4 weights ship, and the
binary rescales internally. `upscale --help` lists the models installed on your
machine.

`realesrnet-x4plus` is a name the binary accepts, but the pinned release does
not ship its weights, so it cannot be installed from here and `-m` will report
it missing.

Removing a model deletes its weights from the sandbox. `upscale models install`
puts them back, from the same pinned GitHub release the first run uses, so there
is only one source to reason about.

The default changed from `realesr-animevideov3` to `realesrgan-x4plus` because
an anime-tuned model is the wrong default for most images. The download is
unchanged — all three models ship in the same release — so only the choice of
model changed, not the size of the install. Use `UPSCALE_MODEL` or `-m` to go
back.

### Model controls

`realesrgan-ncnn-vulkan` takes more flags than `upscale` uses by default. These
are passed through when given, and left to the binary's own defaults when not:

```sh
upscale 8K huge.png -t 128     # smaller tiles: fits large images on small GPUs
upscale 4K photo.jpg -x        # test-time augmentation: ~1.9x slower, sharper
upscale 4K photo.jpg -g 0      # pin a GPU device id as the Vulkan loader sees them
```

`auto` GPU selection is a good default on a hybrid-GPU laptop, but `-g` is the
escape hatch when the discrete card is busy or the integrated part is what you
want.

An unknown model name is refused up front, with the installed ones listed.
Leaving that to the binary is not an option: given a name it does not know, it
prints a wall of `fopen` errors, exits 0, and writes an unpredictable image
anyway.

Custom or fine-tuned weights are **not** supported. This build of
`realesrgan-ncnn-vulkan` only loads from its own `models/` directory, so a path
is refused with an explanation rather than silently ignored.

## How exact sizes work

Real-ESRGAN only does whole 2x/3x/4x. For any other size, `upscale` runs the model
at the smallest factor that covers the target, then resizes to the exact pixel count
with ImageMagick and centre-crops.

The trade-off: you get exactly the dimensions you asked for, and model detail, but
the final pixels went through a lanczos resample rather than coming straight out of
the network. If source images already match your target aspect ratio and just need
more resolution, the resample is a no-op and the output is pure model output.

Output is always exactly `WxH`. Aspect ratio is preserved by cropping rather than
stretching, so a 4:3 source at 4K loses the top and bottom.

Phone and camera photos are stored rotated with an EXIF orientation tag, so their
stored pixel grid is not the image you see — a portrait shot is often stored as
4032x3024 and meant to display as 3024x4032. `upscale` applies the tag before
measuring, so the scale factor and the model both see upright pixels. Images
without a tag, or with orientation 1, are untouched.

### When the source is too small

Real-ESRGAN only does 2x, 3x and 4x. If your source is small enough that even
4x misses the target, the shortfall is made up by ImageMagick, which is ordinary
interpolation — none of the detail the model exists to add. `upscale` says so
rather than presenting the result as an AI upscale:

```sh
upscale 8K tiny.png
# upscale: tiny.png is only 100x100; 7680x4320 is out of reach for the model,
#   so 1920% of the final size is plain interpolation, not model detail.
#   Pass --force if you want it anyway.
```

`-y` (or `--force`) proceeds anyway with a one-line note. Targets are capped at
16384px on an axis, since beyond that it is a typo rather than a request.

## Sandboxing

The model install never touches your normal mise installation or any system
path. The script points mise's `MISE_DATA_DIR`, `MISE_CONFIG_DIR`,
`MISE_STATE_DIR` and `MISE_CACHE_DIR` at a private subdirectory of
`~/.local/share/upscale`, and that is the only thing stored there.

Per-image intermediates do **not** go in the sandbox or next to your photos.
Each run makes a private scratch directory under `$TMPDIR` (default `/tmp`),
mode `700`, and removes it on exit — including on Ctrl-C or a crash:

```sh
TMPDIR=/somewhere/big upscale 8K huge.png    # move the scratch space
```

That keeps the input directory untouched, so read-only photo folders work, a
failed run leaves nothing behind, and `upscale` can never overwrite a file that
happens to be named like an intermediate.

```sh
upscale uninstall             # asks before deleting; -y to skip the prompt
```

`uninstall` is deliberately hard to trigger by accident, because the sandbox
root comes from `UPSCALE_HOME` and a wrong value there would otherwise mean
`rm -rf` on an arbitrary directory. It refuses to delete anything that is not
recognisably the sandbox — the directory must contain a `mise/` folder, be at
least two levels deep, and be neither `/` nor your home directory.

It also only deletes the launcher when that is unambiguous: a file named
`upscale` inside a `bin/` directory, matching the one on `PATH`. Run from a git
checkout or a dotfiles repo, it leaves the file alone and tells you where it
looked. Pass `--all` to remove the launcher regardless.

Override the location with `UPSCALE_HOME`. Override the model with `UPSCALE_MODEL`.

## Development

`upscale selftest` runs 117 assertions over the size table, scale selection,
output paths, the uninstall guard, the scratch directory, the frame selector,
and the model catalog. It needs no GPU, no model install, and no network, and
takes well under a second:

```sh
./upscale selftest
```

The model is only downloaded when an input is actually smaller than its target.
A pure downscale never invokes it, so it never pays for the install.

CI runs `selftest`, `shellcheck`, and an end-to-end pass on every push and pull
request. `.github/workflows/test.yml` holds the workflow.

## Credit

Real-ESRGAN by Xintao Wang and the [Real-ESRGAN contributors](https://github.com/xinntao/Real-ESRGAN/graphs/contributors),
MIT licensed. The ncnn-Vulkan build is the official release asset; this repo only
downloads and drives it.

## License

MIT. See [LICENSE](LICENSE).
