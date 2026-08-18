# ScriptCTF Write-up: RecoverMyPet

Category: Forensics

## Initial approach

The challenge gave me **36 PNG images**, each of size `60×60`.

My first thought was:

> "Maybe all these images are pieces of one larger image. I should arrange them together."

But there was an immediate problem:

**On what basis should I arrange them?**

The filenames weren't in a simple sequence:

```text
43_37.png
17_11.png
1_1.png
23_31.png
3_5.png
...
```

At first, I wasn't sure whether those numbers represented coordinates, keys, parameters, or were just random filenames.

So instead of guessing the order, I started with the usual first step for an image-forensics challenge:

```bash
exiftool *
```

## Finding the hidden metadata

The EXIF output contained an interesting `Image Description` field.

For example:

```text
Image Description :
i once had a cat, u know. but this crazy scientist took my cat
and turned it into a donut 176 times and i cant find my cat anymore! (1/1)
```

There were two things that immediately stood out:

```text
"donut 176 times"
```

and:

```text
(1/1)
```

Checking the other images showed positions such as:

```text
(1/1)
(1/2)
(1/3)
...
(6/6)
```

That was the answer to my earlier question about **how to arrange the images**.

There were exactly 36 images and the metadata contained every position from `(1/1)` to `(6/6)`.

So the images clearly formed a:

```text
6 × 6
```

grid.


## Extracting the descriptions

Rather than manually checking all 36 files, I extracted the descriptions:

```bash
exiftool -s3 -ImageDescription *.png > descriptions.txt
```

I then parsed them with Python to get the useful information.

The result looked like:

```text
1/1: 43_37.png    parameters=(43,37) donut=176
1/2: 17_11.png    parameters=(17,11) donut=21
1/3: 1_1.png      parameters=(1,1)   donut=489
1/4: 23_31.png    parameters=(23,31) donut=272
1/5: 3_5.png      parameters=(3,5)   donut=321
1/6: 8_5.png      parameters=(8,5)   donut=52

2/1: 41_31.png    parameters=(41,31) donut=141
...
6/6: 73_97.png    parameters=(73,97) donut=130
```

Now there were three obvious pieces of information:

* `(row/column)` → where the tile belongs
* `"donut N times"` → some number of transformations
* `a_b.png` → two numbers associated with the transformation


## Building the initial grid

Since the EXIF metadata gave us the positions, I created a `6×6` arrangement.

Each image was `60×60`, so the combined image was:

```text
6 × 60 = 360
```

giving a:

```text
360×360
```

image.

I saved this as:

```text
grid.png
```

The resulting image was still scrambled, but it was no longer just a collection of unrelated files.

There were patterns and parts of an image visible.


## The "cat" and "donut" clue

The next important clue was hiding in the description itself:

> "this crazy scientist took my cat and turned it into a donut"

The word **cat** made me think of the **Arnold Cat Map**, a mathematical transformation commonly used for image scrambling.

I also tried simply Googling things along the lines of:

```text
cat cryptography
cat steganography
cat image encryption
```

and the **Arnold Cat Map** comes up very quickly.

That was a strong confirmation that the challenge was probably referring to it.

The "donut" wording also made sense because the Arnold Cat Map works on a **torus** — you can think of the image as being wrapped around a donut-shaped surface.

So:

> **cat + donut + repeated N times**

was effectively a hint towards:

> **Apply the Arnold Cat Map N times.**


## Understanding the filenames

The filenames now started making sense too.

For example:

```text
43_37.png
```

contains two values:

```text
a = 43
b = 37
```

These act as the parameters of the generalized Arnold transformation.

The transformation can be represented as:

```text
x' = x + a*y                 mod N
y' = b*x + (a*b + 1)*y       mod N
```

For our images:

```text
N = 60
```

because each tile is `60×60`.

So for:

```text
43_37.png
```

the parameters are:

```text
a = 43
b = 37
```

And the EXIF metadata tells us how many times the transformation was applied.

For that particular tile:

```text
... turned it into a donut 176 times ...
```

So it had been transformed **176 times**.

## Reversing the Arnold Cat Map

Since the images were already scrambled, I needed to use the **inverse transformation**.

The inverse can be written as:

```text
x = (a*b + 1)x' - a*y'       mod N
y = -b*x' + y'               mod N
```

For every tile, the process was:

```text
60×60 scrambled tile
        ↓
read a,b from filename
        ↓
read iteration count from EXIF
        ↓
apply inverse Arnold map
        ↓
repeat N times
        ↓
recovered tile
```

I did this independently for all 36 tiles.

Then I placed the recovered tiles back into their original `(row,column)` positions.

This produced:

```text
recovered.png
```


## The image starts revealing itself

Once the Arnold transformations were reversed, the image looked dramatically different.

Instead of the scrambled patterns, the underlying image became recognizable.

Most importantly, there was **handwritten text** in the recovered image.

The beginning was recognizable as:

```text
scrip{w@t_e_p@
```

So we knew we had successfully reached the hidden payload.

I then extracted the red pixels from the recovered image to make the handwriting easier to read.

That produced:

```text
red_text.png
```

Both `recovered.png` and `red_text.png` contained the recovered flag.

### Recovered Images

LINK FOR IMAGES??


Flag: scriptCTF{w@t_4_cu71e_p@too1$}

## Final takeaway

This was a nice example of a challenge where the clues were hidden in plain sight.

The intended path was essentially:

> **Don't guess the order → inspect metadata → discover the grid → recognize the Cat Map → use filename parameters + iteration counts → reverse the transformations → recover the handwriting → get the flag.**

The most satisfying part was that the seemingly weird sentence about a scientist turning a cat into a donut wasn't just flavor text - it was practically the instruction for the entire image-decryption step.
