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
-m NAME   which Real-ESRGAN model (default: realesr-animevideov3)
selftest  check the size and scale logic, no GPU needed
uninstall remove the sandbox and this launcher
```

Default output is `<name>-<size>.png` next to the input.

### Models

`realesr-animevideov3` is the default. It handles x2, x3 and x4, so it's the only
one that works for every target. For photographs `realesrgan-x4plus` usually looks
better but forces x4:

```sh
upscale 4K photo.jpg -m realesrgan-x4plus
```

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

## Sandboxing

Nothing is written outside `~/.local/share/upscale`. The script points mise's
`MISE_DATA_DIR`, `MISE_CONFIG_DIR`, `MISE_STATE_DIR` and `MISE_CACHE_DIR` at a
private subdirectory, so the model install never touches your normal mise
installation or any system path.

```sh
upscale uninstall   # deletes the sandbox and the script itself
```

Override the location with `UPSCALE_HOME`. Override the model with `UPSCALE_MODEL`.

## Credit

Real-ESRGAN by Xintao Wang and the [Real-ESRGAN contributors](https://github.com/xinntao/Real-ESRGAN/graphs/contributors),
MIT licensed. The ncnn-Vulkan build is the official release asset; this repo only
downloads and drives it.

## License

MIT. See [LICENSE](LICENSE).
