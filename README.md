# RISC-V NES Background Bitmap Renderer

This repository contains a **RISC-V 32-bit Assembly project** developed for the course
**CIIC 5995 – Selected Topics in Computer Science and Engineering: Architectures and Programming Paradigms for Game Development (Spring 2026)**
at the **University of Puerto Rico, Mayagüez**.

The project focuses on **full-screen bitmap rendering** using **memory-mapped video output** in the **Ripes simulator**, demonstrating the performance and memory cost of software-based framebuffer rendering.

## Project Overview

**RISC-V NES Background Bitmap Renderer** is an educational experiment that renders a **256×240 pixel background image** (NES native resolution) by copying a precomputed bitmap stored in memory directly to the LED Matrix framebuffer.

This assignment was designed to highlight:

* The high memory footprint of raw bitmap graphics
* The performance limitations of naive full-frame memory copying
* The architectural challenges of low-level graphics rendering without hardware acceleration

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

## Tools & Technologies

* **RISC-V 32-bit Assembly**
* **Ripes Simulator**

  * LED Matrix peripheral
  * Memory-mapped framebuffer
* **Python 3**
* **Pillow (PIL)** — Image processing library

## Learning Objectives

This project reinforces:

* Memory-mapped graphics programming
* Low-level framebuffer manipulation
* Performance analysis of software rendering
* Data layout design for large binary assets
* Tradeoffs between memory usage and rendering simplicity
* Practical limitations of brute-force rendering approaches

## License

This project is licensed under the **MIT License**.
See the [LICENSE](LICENSE) file for details.

## Contact

If you have any questions or suggestions, feel free to reach out:

* **GitHub:** [Neowizen](https://github.com/Yamil-Serrano)
