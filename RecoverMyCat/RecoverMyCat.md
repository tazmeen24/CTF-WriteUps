# ScriptCTF: RecoverMyPet (Forensics)

We're given 36 PNG files, all 60×60. Filenames look like this:

```
43_37.png
17_11.png
1_1.png
23_31.png
3_5.png
```

Nothing about the naming hints "here's the order," so i just ran exiftool on all images.

```
exiftool *
```

The `ImageDescription` field on every image had hints for the chall:

```
i once had a cat, u know. but this crazy scientist took my cat
and turned it into a donut 176 times and i cant find my cat anymore! (1/1)
```

Two things jumped out immediately - the `(1/1)` at the end, and "donut 176 times" in the middle. Checking a few more tiles, the position tags ran all the way from (1/1) to (6/6), which along with the 36-file count basically confirms this is a 6×6 grid and tells you exactly where each tile goes.

Rather than manually arranging 36 files I dumped all the descriptions at once:

```
exiftool -s3 -ImageDescription *.png > descriptions.txt
```

and parsed them with a Python script. The output looked like:

```
1/1: 43_37.png    parameters=(43,37) donut=176
1/2: 17_11.png    parameters=(17,11) donut=21
1/3: 1_1.png      parameters=(1,1)   donut=489
1/4: 23_31.png    parameters=(23,31) donut=272
1/5: 3_5.png       parameters=(3,5)   donut=321
1/6: 8_5.png       parameters=(8,5)   donut=52

2/1: 41_31.png    parameters=(41,31) donut=141
...
6/6: 73_97.png    parameters=(73,97) donut=130
```

So now there's 3 infos per tile: where it goes in the grid, how many times it got "donut"-ed, and a pair of numbers from the filename that clearly go together with that donut count somehow.

Using the grid positions known, stitching 36 tiles of 60×60 gives a 360×360 image - still scrambled, but no longer just a folder of noise. Saved that as `grid.png`:

```python
from PIL import Image
import glob
import re
import subprocess

imgs = {}
for f in glob.glob("*.png"):
    desc = subprocess.check_output(
        ["exiftool", "-s3", "-ImageDescription", f],
        text=True
    ).strip()
    m = re.search(r"\((\d+)/(\d+)\)", desc)
    if m:
        row, col = map(int, m.groups())
        imgs[(row, col)] = Image.open(f).convert("RGB")

out = Image.new("RGB", (6 * 60, 6 * 60), "white")
for row in range(1, 7):
    for col in range(1, 7):
        out.paste(
            imgs[(row, col)],
            ((col - 1) * 60, (row - 1) * 60)
        )
out.save("grid.png")
print("saved grid.png")
```

`check_output` just shells out to exiftool per file and grabs the description string back as text, then the regex yanks the `(row/col)` pair out of it and that becomes the dict key. Only fiddly bit is the `-1` when pasting, since exiftool's positions start at 1 but pixel coords obviously don't.

You could already tell there was *something* structured underneath the mess.

## Cat, donut, and the actual hint

The text "this crazy scientist took my cat and turned it into a donut" is doing more work than it looks like. "Cat" scrambling images is a known thing: the Arnold Cat Map. A quick search for "cat image encryption" turns it up immediately. And the donut bit isn't just a random word either, Arnold's cat map is defined on something called a 'torus', which is, well, donut-shaped. Wrap the image coordinates around instead of clipping them at the edge, and that's the map.

Put together: **apply the Arnold Cat Map N times**

## Making sense of the filenames

`43_37.png` gives two numbers, `a = 43` and `b = 37`. These are the parameters for the generalized Arnold transform:

```
x' = x + a*y mod N
y' = b*x + (a*b + 1)*y mod N
```

N here is 60, since each tile is 60×60. So for that particular file, a=43, b=37, and the EXIF text told us it had been through the transform 176 times.

## Undoing it

Since we're starting from the scrambled version, we need the inverse of that matrix:

```
x = (a*b + 1)*x' - a*y' mod N
y = -b*x' + y' mod N
```

For each tile: read a and b from the filename, read the iteration count from EXIF, apply the inverse that many times, done. Repeated across all 36 tiles, then placed back into their (row, col) slots from the EXIF positions, giving `recovered.png`:

```python
from PIL import Image
import numpy as np

params = [
    [(43,37,176), (17,11,21),  (1,1,489),   (23,31,272), (3,5,321),   (8,5,52)],
    [(41,31,141), (61,53,76),  (41,59,300), (9,13,586),  (13,9,499),  (2,3,663)],
    [(19,29,491), (7,4,586),   (5,3,646),   (13,19,460), (59,41,666), (23,17,573)],
    [(19,13,237), (3,2,436),   (53,61,157), (2,1,150),   (31,23,433), (29,19,514)],
    [(7,11,65),   (1,2,220),   (5,8,180),   (29,37,311), (31,41,283), (37,43,413)],
    [(4,7,47),    (11,7,210),  (37,29,5),   (17,23,587), (11,17,299), (73,97,130)]
]

N = 60
img = np.array(Image.open("grid.png").convert("RGB"))

def inverse_arnold(im, a, b):
    out = np.empty_like(im)
    for y in range(N):
        for x in range(N):
            old_x = ((a * b + 1) * x - a * y) % N
            old_y = (-b * x + y) % N
            out[old_y, old_x] = im[y, x]
    return out

recovered = Image.new("RGB", (6 * N, 6 * N))
for row in range(6):
    for col in range(6):
        tile = img[row * N:(row + 1) * N, col * N:(col + 1) * N]
        a, b, iterations = params[row][col]
        print(f"tile {row+1}/{col+1}: a={a}, b={b}, iterations={iterations}")

        for _ in range(iterations):
            tile = inverse_arnold(tile, a, b)

        recovered.paste(Image.fromarray(tile), (col * N, row * N))

recovered.save("recovered.png")
print("saved recovered.png")
```

`params` is just the parsed filenames/EXIF data typed out by hand, laid out in the same 6×6 shape as the grid so `params[row][col]` matches up with the tile at that position - nothing here was brute-forced, it's literally copy-pasted from what exiftool gave us.

The actual work happens in `inverse_arnold`. For every pixel it asks "where did this pixel come from before the scramble," using the inverse equations from earlier (`old_x = (ab+1)x - ay`, `old_y = -bx + y`, mod N), and writes that into a brand new array instead of overwriting `im` as it goes, if you mutate the same array you're reading from mid-loop you'll end up computing some inverse positions off of *already-inverted* pixels, which corrupts the result.

And then the loop that actually matters:

```python
for _ in range(iterations):
    tile = inverse_arnold(tile, a, b)
```

One call = one step backward. Run it `iterations` times and you've walked the tile all the way back to before any scrambling happened. Once a tile's done it just gets pasted into its slot in `recovered`.

## The reveal

Once every tile was un-scrambled and put back together, the image stopped looking like noise and turned into something with actual handwriting on it. Early on you could already make out the start of a flag string forming. Pulling just the red channel out of the recovered image made the handwriting a lot easier to read - that's `red_text.png`:

```python
from PIL import Image
import numpy as np

im = np.array(Image.open("recovered.png").convert("RGB"))
r = im[:, :, 0].astype(int)
g = im[:, :, 1].astype(int)
b = im[:, :, 2].astype(int)

mask = (
    (r > g + 25) &
    (r > b + 25) &
    (r > 100)
)

out = np.ones_like(im) * 255
out[mask] = [0, 0, 0]

out = Image.fromarray(out).resize((1440, 1440))
out.save("red_text.png")
print("saved red_text.png")
```

Split into r/g/b channels, cast to int so the subtractions don't wrap around weirdly on uint8. A pixel only counts as "red" if it clearly beats the other two channels (margin of 25, not just barely) and isn't too dark that `r > 100` floor is what keeps dark background noise out of the mask. Everything that passes gets painted black on a white canvas, everything else stays white, then it's just scaled up 4x so the handwriting's actually readable.

Worth noting `recovered.png` on its own already has the flag readable if you read it carefully, either image gets you the flag.

<img src="https://raw.githubusercontent.com/tazmeen24/scriptCTF-2026-WriteUps/main/RecoverMyCat/recovered.png" alt="Recovered image" width="360"> <img src="https://raw.githubusercontent.com/tazmeen24/scriptCTF-2026-WriteUps/main/RecoverMyCat/red_text.png" alt="Red channel extraction" width="360">

**Flag:** `scriptCTF{w@t_4_cu71e_p@too1$}`
