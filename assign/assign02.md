---
layout: mathjax
title: "Assignment 2: Image Processing"
---

<div style="font-size: 120%; font-style: italic; text-align: center;">
  Note: this is a preliminary assignment description.
  Elements of the assignment description may be missing or incomplete,
  and details may change.
</div>

**Due**:

* Milestone 1 due **Friday, Sep 20th** by 11 pm
* Milestone 2 due **Friday, Sep 27th** by 11 pm
* Milestone 3 due **Friday, Oct 4th** by 11 pm

This is a **pair** assignment, so you may work with one partner.

<div class='admonition danger'>
  <div class='title'>Warning!</div>
  <div class='content' markdown='1'>
Assembly language programming is challenging! Make sure
you start each milestone as soon as possible, work steadily, and
ask questions early.  Also, writing unit tests and using `gdb` to
examine the detailed behavior of code under test will be critical
to successful implementation of the assembly language functions.
  </div>
</div>

## Overview

In this assignment, you will implement transformations on image files,
using both C (in Milestone 1) and x86-64 assembly language (in Milestones 2 and 3.)

### Non-functional requirements

In Milestones 2 and 3, you will be writing assembly language functions.
You **must** write these "by hand", and your assembly code must have
very detailed code comments explaining the purpose of each assembly language
instruction.

It is **not** allowed to generate assembly
code using a C compiler and submit this as your own code. We will assign
a grade of 0 to any submissions containing compiler-generated code
where hand-written assembly language is expected.

Your submission for each milestone should include a `README.txt` file
describing how you and your partner divided the work, and letting
us know about any interesting implementation details. If there is
functionality you weren't able to get working completely, this is
a good place to mention that.

We expect you to follow the [style guidelines](style.html).
However, the expectations for function length will be relaxed considerably
for your assembly language code. It is not unusual for an assembly language
function to have 100 or more lines of code. In the reference solution,
the longest function was 115 lines, although there was extensive use of
comments and whitespace to improve readability.

Of course, you should strive to make your assembly language functions
as simple and readable as possible.

We expect your code to be free of memory errors. You should use
`valgrind` to test your code to make sure there are no uses
of uninitialized variables, out of bounds memory reads or writes,
etc.  This applies to both your C code and your assembly code.

## Getting started

To get started, download [csf\_assign02.zip](csf_assign02.zip) and
unzip it.

You will implement the functions in `c_imgproc_fns.c` (Milestone 1)
and `asm_imgproc_fns.S` (Milestones 2 and 3.) You will also add
prototypes for helper functions to `imgproc.h` and implement additional
unit tests in `imgproc_tests.c`.

## Grading breakdown

*TODO: rubric for each milestone*

Note that in each milestone, we expect all of the tests executed
by your unit test program to pass. For Milestone 2 in particular, you can
comment out calls to test functions that aren't related to
`draw_pixel`. For example, for Milestone 2 your `imgproc_test.c`'s
`main` function might have code similar to the following:

```c
TEST( test_get_r );
TEST( test_get_g );
TEST( test_get_b );
TEST( test_get_a );
TEST( test_make_pixel );
TEST( test_to_grayscale );
//TEST( blend_components );
//TEST( blend_colors );
```

The tests for `get_r`, `get_g`, `get_b`, `get_a`, `make_pixel`, and
`to_grayscale` are enabled because they are all test functions involved
in the implementations of the `grayscale` transformation, which is
part of the requirements for MS2. The tests for `blend_components` and
`blend_colors` are commented out because they are used in the `composite`
transformation, which is not part of the requirements for MS2.

## Image processing

This section describes the image format and the image transformations
you will implement.

### struct Image

An instance of the `struct Image` data type represents a grid of
pixels, where each pixel has 8-bit red, green, and blue color
component values, as well as an 8-bit alpha value.

The `struct Image` type is defined as follows (in `image.h`):

```c
struct Image {
  int32_t width;
  int32_t height;
  uint32_t *data;
};
```

The `width` and `height` fields define the width and height of an image,
in pixels. The `data` field is a pointer to a dynamically-allocated array
of `uint32_t` values, each one representing one pixel. The pixels are stored
in row-major order, starting with the top row of pixels.

A color is represented by a `uint32_t` value as follows:

* Bits 24-31 are the 8 bit red component value, ranging from 0–255
* Bits 16-23 are the 8 bit green component value, ranging from 0–255
* Bits 8–15 are the 8 bit blue component value, ranging from 0–255
* Bits 0–7 are the 8 bit alpha value, ranging from 0–255

This pixel data format is known as "RGBA".

The alpha value of a color represents its opacity, with 255 meaning
"fully opaque" and 0 meaning "fully transparent". (See the
[Color blending](#color-blending) section for details on how
an alpha value allows two colors to be "blended".)

### Color blending

The color values of the destination image are always fully opaque,
with an alpha value of 255.

When a pixel is drawn to a destination image by any operation other than
`draw_tile`, the pixel's color, which we'll call the "foreground" color,
is blended with the existing color at the location where the pixel is being
drawn, which we'l call the "background" color. To find the correct color
value for the new pixel, the following computation is performed for
each color component, where $$f$$ is the foreground color component value,
$$b$$ is the background color component value, and $$\alpha$$ is the
alpha value of the foreground color:

$$\lfloor (\alpha f + (255 - \alpha)b) / 255 \rfloor$$

Note that the result of the division is truncated rather than being rounded,
so if you use integer division, it will behave in the expected way.

A blended color should have each color component value (red, green, and blue)
computed using the formula above, and the alpha value of the blended color
should be set to 255.

### Grayscale

In [the grayscale transformation](#the-grayscale-transformation), color pixels
are converted to gray pixels as follows. From the color pixel's red, green, and
blue color component values, denoted as $$r$$, $$g$$, and $$b$$, a "gray"
value, denoted $$y$$, is computed using the following formula:

$$y = \lfloor (79 \times r + 128 \times g + 49 \times b) / 256 \rfloor$$

The resulting gray pixel should have its red, green, and blue color component
values set to $$y$$, and its alpha value should be the same as the original
color pixel's alpha value.

### Image transformation functions

You will implement the following image transformation functions in both
C and assembly language:

```c
void imgproc_mirror_h( struct Image *input_img,
                       struct Image *output_img );
void imgproc_mirror_v( struct Image *input_img,
                       struct Image *output_img );
int imgproc_tile( struct Image *input_img,
                  int n, struct Image *output_img );
void imgproc_grayscale( struct Image *input_img,
                        struct Image *output_img );
int imgproc_composite( struct Image *base_img,
                       struct Image *overlay_img,
                       struct Image *output_img );
```

These functions are declared in `imgproc.h`, and each one has a detailed API
comment describing its function, the meaning of the parameters, and the
meaning of the return value (for the non-`void` functions.)

### The `mirror_h` transformation

The `mirror_h` transformation mirrors the input image horizontally.

Example images (click for full size):

Original image | Transformed image
:------------: | :---------------:
<a href="img/ingo.png"><img style="width: 20em;" alt="original cat image" src="img/ingo.png"></a> | <a href="img/ingo_mirror_h.png"><img style="width: 20em;" alt="horizontally mirrored cat image" src="img/ingo_mirror_h.png"></a>

### The `mirror_v` transformation

The `mirror_v` transformation mirrors the input image vertically.

Example images (click for full size):

Original image | Transformed image
:------------: | :---------------:
<a href="img/ingo.png"><img style="width: 20em;" alt="original cat image" src="img/ingo.png"></a> | <a href="img/ingo_mirror_v.png"><img style="width: 20em;" alt="vertically mirrored cat image" src="img/ingo_mirror_v.png"></a>

### The `grayscale` transformation

The `grayscale` transformation converts a color image to grayscale.

The [Grayscale](#grayscale) section documents how color pixel values are
converted to grayscale pixel values.

Example images (click for full size):

Original image | Transformed image
:------------: | :---------------:
<a href="img/ingo.png"><img style="width: 20em;" alt="original cat image" src="img/ingo.png"></a> | <a href="img/ingo_grayscale.png"><img style="width: 20em;" alt="grayscale cat image" src="img/ingo_grayscale.png"></a>

### The `tile` transformation

The `tile` transformation generates an image containing an $$n$$ x $$n$$
arrangement of tiles, each tile being a smaller version of the original
image, and the overall result image having the same dimensions as the
original image. The value $$n$$ is the "tiling factor".

Example images with tiling factor $$n=3$$ (click for full size):

Original image | Transformed image
:------------: | :---------------:
<a href="img/ingo.png"><img style="width: 20em;" alt="original cat image" src="img/ingo.png"></a> | <a href="img/ingo_tile_3.png"><img style="width: 20em;" alt="3x3 tiled cat image" src="img/ingo_tile_3.png"></a>

Note that when the image's width or height isn't evenly divisible by
$$n$$, the excess should be spread out evenly, starting with the leftmost tiles
(for excess width) and topmost tiles (for excess height).  For example,
in the 3 x 3 case for an 800x600 source image, the tile widths should
be 267, 267, and 266, and the tile heights should be 200, 200, and 200.

The tiles should sample every $$n$$th pixel from the source image
horizontally and vertically, starting with the pixel in the upper-left
corner of the original image.

### The `composite` transformation

The `composite` transformation overlays an "overlay" image on top of a "base"
image. Each pixel in the output image is determined by blending the corresponding
pixels of the base image (the "background" pixel) and overlay image (the
"foreground" pixel.) The foreground pixel's alpha value is used to
determine the relative contribution of the foreground and background pixels'
color component values to the color component values of the output pixel.
Each output pixel is fully opaque (alpha is 255.)

The [color blending](#color-blending) section describes how color blending should
be implemented.

As an example, consider the following base and overlay images
(click for full size):

Base image | Overlay image
:--------: | :-----------:
<a href="img/kittens.png"><img style="width: 20em;" alt="image of several kittens" src="img/kittens.png"></a> | <a href="img/dice.png"><img style="width: 20em;" alt="partially-transparent image of colored dice" src="img/dice.png"></a>

Note that most of the pixels of the overlay image are either completely or partially
transparent, meaning that their alpha values are less than 255.

Compositing the two images produces the following result image
(click for full size):

<table>
 <tr>
   <th style="text-align: center;">Composite of base and overlay</th>
 </tr>
 <tr>
   <td>
<a href="img/kittens_composite_dice.png"><img style="width: 20em;" alt="dice image overlayed on image of kittens" src="img/kittens_composite_dice.png"></a>
   </td>
 </tr>
</table>

## `c_imgproc` and `asm_imgproc` programs

The `c_imgproc` and `asm_imgproc` programs apply one of the image transformations
to an input image (or, in the case of the `composite` transformation, two input images),
and write the result to an output image file.  The `c_imgproc_main.c` source file
implements both of these programs. The only difference between `c_imgproc` and
`asm_imgproc` is whether the image transformations are implemented in C
(`c_imgproc_fns.c`) or x86-64 assembly language (`asm_imgproc_fns.S`.)

To run these programs:

<div class='highlighter-rouge'><pre>
./c_imgproc <i>input.png</i> <i>transformation</i> <i>output.png</i> [<i>transformation argument</i>]
./asm_imgproc <i>input.png</i> <i>transformation</i> <i>output.png</i> [<i>transformation argument</i>]
</pre></div>

In these commands:

* <code class='highlighter-rouge'><i>input.png</i></code> is the name of the input
  image file
* <code class='highlighter-rouge'><i>transformation</i></code> is the name of the
  image transformation to perform
* <code class='highlighter-rouge'><i>output.png</i></code> is the name of the output
  image file to write
* <code class='highlighter-rouge'>[<i>transformation argument</i>]</code> is the
  argument needed by the transformation, if any (the tiling factor for the
  `tile` transformation, and the overlay image filename for the `composite`
  transformation)

For example, to run the `mirror_h` transformation using the `c_imgproc` program:

```text
mkdir -p actual
./c_imgproc input/ingo.png mirror_h actual/c_ingo_mirror_h.png
```

The above commands would apply the `mirror_h` transformation on the input image
`input/ingo.png` to produce the output image file `actual/c_ingo_mirror_h.png`.

## Unit tests

The source file `imgproc_tests.c` is a unit test program that you should use to
test the functions in `c_imgproc_fns.c` and `asm_imgproc_fns.S`.

The starter code has some very basic tests for the API functions implementing the
various image transformations. However, you will need to write unit tests for
your *helper functions*. The idea is that your assembly language code will implement
exactly the same helper functions as your C code, and having a comprehensive set
of unit tests for these helper functions will allow you to get your assembly code
working incrementally by implementing the helper functions one at a time.

<div class='admonition info'>
  <div class='title'>Tip</div>
  <div class='content' markdown='1'>
Having a good set of unit tests for your helper functions is essential for being
able to make steady progress towards getting your assembly code to work in
Milestones 2 and 3.
  </div>
</div>
