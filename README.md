# RISC-V NES Background Bitmap Renderer

This repository contains a **RISC-V 32-bit Assembly graphics project** developed for the course  **CIIC 5995 – Selected Topics in Computer Science and Engineering: Architectures and Programming Paradigms for Game Development (Spring 2026)**  
at the **University of Puerto Rico, Mayagüez**.

The project explores **low-level framebuffer-based rendering** using **memory-mapped video output** in the **Ripes simulator**, with a focus on understanding the performance and memory tradeoffs of software-driven graphics pipelines.

## Project Overview

**RISC-V NES Background Bitmap Renderer** is an educational experiment that renders a full-screen **256×240 pixel background** (NES native resolution) by transferring precomputed image data directly from main memory into the LED Matrix framebuffer.

This assignment was designed to highlight:

* The memory cost of raw bitmap graphics
* The performance limitations of brute-force framebuffer copying
* The architectural constraints of rendering without hardware acceleration

## Features

* **Full NES Resolution Rendering**

  * Resolution: **256 × 240 pixels**
  * Total pixels: **61,440**
  * Raw 32-bit color framebuffer copy

* **Direct Framebuffer Access**

  * Memory-mapped LED Matrix output
  * Sequential memory write operations
  * Software-driven rendering pipeline

* **Assembly-Based Bitmap Loader**

  * Pixel data stored using `.word` directives
  * Color format: `0x00RRGGBB`
  * Row-based memory layout for clarity and debugging

## Bitmap Representation

The background image is stored in memory as a **linear bitmap array**.

Each pixel is encoded as a 32-bit word using the format:

```
0x00RRGGBB
```

The bitmap is organized by rows for readability:

```asm
.data
.align 2
image_data:
    # Row 000
    .word 0x00EDB5B4, 0x00EDB5B4, 0x00EEB6B5, ...
```

This layout allows direct copying from memory into the LED Matrix framebuffer using sequential `lw` and `sw` instructions.

## Rendering Result

Below is the rendered background displayed using the Ripes LED Matrix peripheral:

<img width="1919" height="1030" alt="Eevee ripes" src="https://github.com/user-attachments/assets/129e77a3-6a12-484d-b115-f02f95409259" />

*Reference image: [Eevee by Alleph3D](https://cults3d.com/es/modelo-3d/juegos/pokemon-eevee-alleph3d-2) — used for educational purposes only.*

## Rendering Pipeline (Assembly)

The rendering process consists of:

1. Clearing the framebuffer
2. Loading the bitmap base address
3. Copying **61,440 pixels** from Memory into the LED Matrix memory region

Simplified rendering loop:

```asm
draw:
    beqz t1, end_draw

    lw t2, 0(t0)
    sw t2, 0(a0)

    addi t0, t0, 4
    addi a0, a0, 4
    addi t1, t1, -1

    j draw
```

This brute-force approach intentionally exposes the **performance bottleneck** of software rendering at high resolutions.

## Python Bitmap Conversion Tool

Because manually converting images into assembly `.word` format is impractical, a custom **Python conversion script** was created.

This tool:

* Loads any image file
* Resizes it to **256×240**
* Converts pixels into `0x00RRGGBB` format
* Exports the result into a `.s` assembly data file compatible with Ripes

### Conversion Script

```python
from PIL import Image


def image_to_compact_nes_asm(image_path, output_file="Image_data.s"):
    img = Image.open(image_path).convert('RGB')
    width, height = 256, 240
    img = img.resize((width, height), Image.Resampling.LANCZOS)
    
    total_pixels = width * height
    
    with open(output_file, 'w') as f:
        f.write(".data\n")
        f.write(".align 2\n")
        f.write("image_data:\n")
        
        for y in range(height):
            f.write(f"    # Row {y:03d}\n")
            for x in range(0, width, 16):
                f.write("    .word ")
                colors = []
                for i in range(16):
                    px = x + i
                    if px < width:
                        r, g, b = img.getpixel((px, y))
                        colors.append(f"0x00{r:02X}{g:02X}{b:02X}")
                f.write(", ".join(colors) + "\n")
    
    print(f"Compact file: {total_pixels:,} pixels -> {output_file}")


# Usage example:
image_to_compact_nes_asm("Example.jpg")
```
## Data Size Optimization Challenge

For the second stage of the project, the professor introduced a new technical constraint: **reduce the size of the `.data` segment by at least 30%** while preserving full-screen background rendering.

This requirement forced a redesign of the original bitmap-based renderer, which previously relied on raw 32-bit color values for every pixel.

Instead of copying full RGB values directly to the framebuffer, the new implementation adopts a **palette-based indexed color approach**, similar to the techniques used in classic console hardware such as the NES.

## Transition From 32bpp to 4bpp Indexed Rendering

### Original Implementation (32bpp)

In the first version of the renderer:

* Each pixel was stored as a **32-bit value (0x00RRGGBB)**
* Memory usage scaled linearly with resolution
* Total image memory footprint:

```
256 × 240 × 4 bytes = 245,760 bytes (~240 KB)
```

While simple to implement, this approach was extremely inefficient in terms of memory consumption.


### Optimized Implementation (4bpp + Palette)

To meet the optimization requirement, the renderer was redesigned to use **indexed color with a fixed 16-color palette**.

Instead of storing full RGB values per pixel, each pixel is now represented by a **4-bit palette index**.

Two pixels are packed into a single byte using the following format:

```
Byte layout:

[ high nibble | low nibble ]
[ pixel 0     | pixel 1    ]
```

This allows a single byte to encode two pixels, reducing memory usage by a factor of eight compared to the original 32-bit format.

### Palette Storage

The color palette contains up to **16 entries**, each stored as a 32-bit RGB value:

```
16 colors × 4 bytes = 64 bytes
```

During rendering, pixel indices are used to look up the actual RGB color values from this palette.

## Palette Storage Design Decision

Although the renderer uses a compact 4bpp indexed bitmap, the palette itself is intentionally stored in **32-bit 0x00RRGGBB format**.

This design choice was made to match the LED Matrix hardware interface, which expects 32-bit color values. Reducing the palette to an 8-bit or 16-bit color format would only save a small amount of memory (at most a few dozen bytes) while requiring additional color expansion logic during rendering.

Keeping the palette in native framebuffer format allows direct writes to video memory without extra conversion steps, simplifying the render pipeline and avoiding unnecessary instruction overhead.

## Memory Footprint Comparison

Resolution: **256 × 240 (61,440 pixels)**

| Format       | Pixel Data    |
| ------------ | ------------- |
| 32bpp bitmap | 245,760 bytes |
| 4bpp indexed | 30,720 bytes  |

This optimization results in an approximate **87.5% reduction in memory usage**, greatly exceeding the minimum 30% reduction target defined by the assignment.

## Rendering Result (Optimized Version)

Below is the output produced by the optimized 4bpp palette-based renderer running on the Ripes LED Matrix peripheral:
<img width="1920" height="997" alt="2026-01-30_22-27" src="https://github.com/user-attachments/assets/353d02a6-3678-41d5-9e09-d03aa0543a68" />
*Now you can notice that the rendered image uses a different color distribution compared to the previous version. This is because the renderer is limited to a 16-color palette instead of full 32-bit color. While this introduces visible color quantization, it also gives the output a characteristic retro-style appearance similar to classic console graphics.*


## Conversion Script (4bpp Palette Exporter)

To support the new indexed rendering pipeline, the original bitmap conversion tool was modified to generate **4bpp packed image data** and a **16-color palette** instead of raw 32-bit pixel values.
```python
from PIL import Image

# Configuration
IMAGE_WIDTH = 256
IMAGE_HEIGHT = 240
PALETTE_SIZE = 16          # Max 16 colors (4 bits)
USE_DITHERING = True       # Smooth color reduction
OUTPUT_FILE = "Image_4bpp.s"


def image_to_4bpp_asm(image_path, output_file=OUTPUT_FILE):

    # Step 1: Load and resize image
    print("Loading image...")
    img = Image.open(image_path).convert("RGB")
    img = img.resize((IMAGE_WIDTH, IMAGE_HEIGHT), Image.Resampling.LANCZOS)

    # Step 2: Reduce to 16 colors
    print("Reducing to 16 colors...")

    dither = Image.FLOYDSTEINBERG if USE_DITHERING else Image.NONE
    quantized = img.quantize(colors=PALETTE_SIZE, dither=dither)

    # Step 3: Extract color palette
    print("Extracting palette...")

    raw_palette = quantized.getpalette()
    palette = []

    for i in range(PALETTE_SIZE):
        r = raw_palette[i * 3 + 0]
        g = raw_palette[i * 3 + 1]
        b = raw_palette[i * 3 + 2]

        # Convert to 0x00RRGGBB format
        color32 = (r << 16) | (g << 8) | b
        palette.append(color32)

    # Step 4: Get pixel indices
    print("Getting pixel indices...")

    pixels = list(quantized.getdata())

    # Step 5: Pack pixels (2 per byte)
    print("Packing pixels...")

    packed_pixels = bytearray()

    for i in range(0, len(pixels), 2):
        pixel0 = pixels[i] & 0x0F
        pixel1 = pixels[i + 1] & 0x0F

        packed_byte = (pixel0 << 4) | pixel1
        packed_pixels.append(packed_byte)

    # Step 6: Write ASM file
    print("Writing ASM file...")

    with open(output_file, "w") as f:
        f.write(".data\n")
        f.write(".align 2\n\n")

        # Write palette
        f.write("# Palette (0x00RRGGBB)\n")
        f.write("palette:\n")

        for i in range(0, PALETTE_SIZE, 4):
            line = []
            for j in range(4):
                if i + j < PALETTE_SIZE:
                    color = palette[i + j]
                    r = (color >> 16) & 0xFF
                    g = (color >> 8) & 0xFF
                    b = color & 0xFF
                    line.append(f"0x00{r:02X}{g:02X}{b:02X}")

            f.write("    .word " + ", ".join(line) + "\n")

        # Write image data
        f.write("\n# Image data (4bpp packed)\n")
        f.write("# Format: [pixel0 | pixel1]\n")
        f.write("image_data:\n")

        bytes_per_row = IMAGE_WIDTH // 2
        idx = 0

        for y in range(IMAGE_HEIGHT):
            f.write(f"    # Row {y:03d}\n")

            for x in range(0, bytes_per_row, 16):
                chunk = packed_pixels[idx:idx + 16]
                idx += 16

                hexes = [f"0x{b:02X}" for b in chunk]
                f.write("    .byte " + ", ".join(hexes) + "\n")

    # Step 7: Show statistics
    image_bytes = len(packed_pixels)
    palette_bytes = PALETTE_SIZE * 4

    print("\n----- Export complete -----")
    print(f"Resolution     : {IMAGE_WIDTH}x{IMAGE_HEIGHT}")
    print(f"Colors         : {PALETTE_SIZE}")
    print(f"Image size     : {image_bytes:,} bytes")
    print(f"Palette size   : {palette_bytes} bytes")
    print(f"Total          : {image_bytes + palette_bytes:,} bytes")
    print(f"Output file    : {output_file}")
    print("---------------------------")


# Run
image_to_4bpp_asm("Example.jpg")
```

## Updated Rendering Pipeline

The optimized renderer introduces an additional decoding stage during rendering:

1. Read one packed byte from memory
2. Extract both 4-bit pixel indices
3. Use each index to access the color palette
4. Retrieve the corresponding 32-bit RGB value
5. Write two pixels to the framebuffer

Although this increases the number of CPU operations per pixel, it significantly reduces memory bandwidth usage and overall data footprint.

## Educational Impact

This optimization stage introduced several important low-level graphics concepts:

* Palette-based rendering
* Indexed color formats
* Bit packing and unpacking
* Memory bandwidth optimization
* CPU vs memory tradeoffs

By implementing this change, the project evolved from a brute-force framebuffer copy into a more realistic retro-style rendering pipeline, closer to how constrained hardware systems handle graphics efficiently.

## Tools & Technologies

* **RISC-V 32-bit Assembly**
* **Ripes Simulator** (LED Matrix peripheral, memory-mapped framebuffer)
* **Python 3**
* **Pillow (PIL)** — Image processing


## License

This project is licensed under the **MIT License**.
See the [LICENSE](LICENSE) file for details.

## Contact

If you have any questions or suggestions, feel free to reach out:

* **GitHub:** [Neowizen](https://github.com/Yamil-Serrano)

