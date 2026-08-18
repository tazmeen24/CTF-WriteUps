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

Using the grid positions known, stitching 36 tiles of 60×60 gives a 360×360 image - still scrambled, but no longer just a folder of noise. Saved that as `grid.png`, and you could already tell there was *something* structured underneath the mess.

## Cat, donut, and the actual hint

The text - "this crazy scientist took my cat and turned it into a donut" is doing more work than it looks like. "Cat" scrambling images is a known thing: the Arnold Cat Map. A quick search for "cat image encryption" turns it up immediately. And the donut bit isn't just a random word either Arnold's cat map is defined on a something called a 'torus', which is, well, donut-shaped. Wrap the image coordinates around instead of clipping them at the edge, and that's the map.

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

For each tile: read a and b from the filename, read the iteration count from EXIF, apply the inverse that many times, done. Repeated across all 36 tiles, then placed back into their (row, col) slots from the EXIF positions, giving `recovered.png`.

## The reveal

Once every tile was un-scrambled and put back together, the image stopped looking like noise and turned into something with actual handwriting on it. Early on you could already make out the start of a flag string forming. Pulling just the red channel out of the recovered image made the handwriting a lot easier to read - that's `red_text.png`.

<img src="https://raw.githubusercontent.com/tazmeen24/scriptCTF-2026-WriteUps/main/RecoverMyCat/recovered.png" alt="Recovered image" width="360"> <img src="https://raw.githubusercontent.com/tazmeen24/scriptCTF-2026-WriteUps/main/RecoverMyCat/red_text.png" alt="Red channel extraction" width="360">

**Flag:** `scriptCTF{w@t_4_cu71e_p@too1$}`
