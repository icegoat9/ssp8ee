Keeping a rough scratchpad of some running dev notes here, rather than in the [README.md](README.md) which was already too long.

## PICO8 Edu Edition @URL Format

Some notes from just a few hours of trial-and-error investigating what this format is.

`https://www.pico-8-edu.com/?c=FOO&g=BAR`

Where, with a few caveats, FOO is the b64 urlsafe-encoded code and BAR is the encoded spritesheet data. `&g=...` can be omitted if not used.

By looking at how the encoded @URL changes if I change single characters or pixels in PICO8, there's some compression (some form of run-length encoding at least for sprite data-- most obvious if you have a row of pixels of the same color) applied to the code before b64 encoding. However, the P8 Edu Edition can accept uncompressed data in the URL as well, which is what I started with just to get something working quickly.

Note: It's possible this is all already well-understood and documented somewhere on the PICO8 site or is used in other PICO8 export formats, I haven't dug into those, but it was fun to spend a few hours poking around.

### Spritesheet

Using the SAVE @URL command on some simple carts and looking at the generated URL, I see that:

* A single dark blue (color 1) spritesheet pixel encodes as `B` (the b64 encoding for value 1)
* A row of 8 pixels in colors 1-8 encodes as `BCDEFGHI` (b64 for 1-8). Interestingly, this isn't b64 encoding the spritesheet memory (e.g. it doesn't seem to be packing the data into a series of 8-bit bytes and then b64 encoding that into 4/3 as many 6-bit characters), but instead is using the b64 mapping on each pixel. That's simpler to read, at least.
* However, a row of 8 dark blue (1) pixels encodes as `xE` rather than `BBBBBBBB`. There's some run-length encoding going on that stores identical consecutive pixels (only) in a more compact format, as a row of 8 alternating pixel values of 1,2,1,2,1,2,1,2 is encoded as the uncompressed `BCBCBCBC`.
  * This makes sense as a design choice, because otherwise all the black background pixels in the spritesheet would take up an enormous amount of URL space with an A for each one...

### Spritesheet compression

Trial and error on some simple pixel patterns in the spritesheet gives these encodings:

| Row of 8 sprite pixel values     | @URL b64 encoding | Decimal | Binary |
| -------- | --- | ---- | ------------- |
| 1        | B  | 1    | |
| 2        | C  | 2    | |
| 12121212 | BCBCBCBC | | |
| 11       | R  | 17   | 010001 |
| 22       | S  | 18   | 010010 |
| 111      | h  | 31   | 100001 |

Aha. It looks like the encoding of N consecutive pixels of value V is something like {N-1}{V}, with only four bits are used for the color V (since it is only a 4-bit color).

That is, the encoding of two blue (color 1) pixels `11` is {01}{0001}, while three blue pixels is {10}{0001} and two color-2 pixels is {01}{0010}. Likely one is subtracted from the number of pixels to make it zero-indexed, and so the common case of a single pixels is {00}{0001}.

Looking at longer pixel patterns, though:

| Row of 8 sprite pixel values     | @URL b64 encoding | Decimal (tbd byte order) | Binary (byte pair order swapped) |
| -------- | --- | ---- | ------------- |
| 1111     | xA | 49,0 | 000000,110001|
| 11111    | xB | 49,1 | 000001,110001 |
| 111111   | xC | 49,2 | 000010,110001 |
| (8x) 1 | xE | 49,4 | 000100,110001 |
| (16x) 1  | xM | 49,12 | 001100,110001 |
| (64x) 1  | x8 | 49,60 | 111100,110001 |
| (67x) 1  | x- | 49,63 | 111111,110001 |

We see that the single b64 value in the previous table can only encode up to 4 consecutive pixels of a color (2 bits for # of pixels, 4 bits for color #). Once we need to encode more than 4 pixels there's something more going on.

If we swap the order of the bytes, it looks like the encoding of N pixels of value V is something more complicated. 
Let's define A as the 6-bit value MAX(N-4,0) and B as the 2-bit value IF(N>4,4,N). Then the encoding appears to be: `{ A }{ B }{ V }`

That is, use the two bits left in the first character to count 1-4 pixels, and if there are more, add more characters that encode the "# of pixels above 4" (but leave the high bits in the first character set to 11 rather than set them as MOD(N,4) or use them as part of a larger binary number). Interesting optimization.

Looking at even longer pixel patterns that need three or more b64 characters:

| Row of 8 sprite pixel values     | @URL b64 encoding | Decimal (tbd byte order) | Binary (byte order swapped) |
| -------- | --- | ---- | ------------- |
| (67x) 1  | x- | 49,63 | 111111,110001 |
| (68x) 1  | x-B | 49,63,1 | 000001,111111,110001, |
| (69x) 1  | x-R | 49,63,17 | 010001,111111,110001, |
| (72x) 1  | x-xB | 49,63,49,1 | 000001,110001,111111,110001 |

At first it was puzzling that we jumped so quickly from three to four characters, but then I see it's just the pattern above, repeated. So it appears that we're not trying to encode arbitrarily long sequences of the same pixel: each pair of b64 characters can represent up to 67 pixels, and if we need 68 or more pixels we just start the encoding over.






https://pico-8-edu.com/?c=Y2xzKDEpc3ByKDEsNjQsMCk=&g=xH
Testing with more consecutive pixels 
https://pico-8-edu.com/?c=Y2xzKDEpc3ByKDEsNjQsMCk=&g=x-R
https://pico-8-edu.com/?c=Y2xzKDEpc3ByKDEsNjQsMCk=&g=x-B
https://pico-8-edu.com/?c=Y2xzKDEpc3ByKDEsNjQsMCk=&g=x-
https://pico-8-edu.com/?c=Y2xzKDEpc3ByKDEsNjQsMCk=&g=xF

detect if we're in run-length encoding mode. 
That is, the encoding of two blue pixels `11` is {10}{0001}
encoded as {A}{B}, where A is four bits representing the color A is the b64-translated value of N and V is just four bits representing the color value from 1-16.

So the `xM` of 16 pixels above is 110001001100 = {110001001}{1}

https://pico-8-edu.com/?c=Y2xzKDEpc3ByKDEsNjQsMCk=&g=x-xJ
https://pico-8-edu.com/?c=Y2xzKDEpc3ByKDEsNjQsMCk=&g=x-xJ
https://pico-8-edu.com/?c=Y2xzKDEpc3ByKDEsNjQsMCk=&g=x-xB
https://pico-8-edu.com/?c=Y2xzKDEpc3ByKDEsNjQsMCk=&g=x-xJ
https://pico-8-edu.com/?c=Y2xzKDEpc3ByKDEsNjQsMCk=&g=x8

https://pico-8-edu.com/?c=Y2xzKDEpc3ByKDEsNjQsMCk=&g=xM

https://pico-8-edu.com/?c=Y2xzKDEpc3ByKDEsNjQsMCk=&g=S
https://pico-8-edu.com/?c=Y2xzKDEpc3ByKDEsNjQsMCk=&g=yE

### b64 error?

I had this wrapper mostly working, but was seeing errors where '>' and '?' characters (CHRs 62 and 63) in code were sometimes (but not always) swapped when opening a cart link generated by this spreadsheet in Education Edition. Adding or removing whitespace in the cart seemed to make the problem go away?! Eventually I discoverd that PICO8's @URL format seems to swap the meaning of '-' and '_' in its b64 encoding compared to the typical urlsafe encoding. Those represent values 62 and 63, so we'd expect a common decoding issue when > or ? characters line up with three-character boundaries in the input data (we'd also see this for other combinations of characters but mostly less used extended characters). I don't know if this is some light obfuscation or was unintentional, but I've adjusted this wrapper to match what PICO8's URL format seems to expect.

## Google Sheets notes-to-self

It's possible to build a lot of functionality in spreadsheets (for example, a string->b64 encoder), even without incorporating scripting (AppScript, VBA, etc). A few more modern spreadsheet functions have made that easier than it was a decade ago:

* `MAP()` and `LAMBDA()` are powerful and flexible alternatives to array formulas or having sheets of 'intermediate data' that later formulas process.

  * I haven't figured out if there's a cleaner way to split a string into an array of character values but this is fairly concise (assuming we can choose some character like "¶" we know isn't naturally in the input):

```
chararray = MAP(SPLIT(REGEXREPLACE(input,,"¶"),"¶"),lambda(x,CODE(x)))
```

* The `LET()` function in Google Sheets is very convenient for avoiding hard-to-read and error-prone copy and paste within formulas, avoiding the need for various 'intermediate value' dummy cells, and can be abused as a way to add quick comments within a formula. Why write:
```
mid("ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789_-",a,1) & 
mid("ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789_-",b,1) & 
mid("ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789_-",c,1) &
mid("ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789_-",d,1)
```
when you can write something like:
```
let(b64map,let(comment,"b64 encoding (nonstandard with -,_ swapped)",
    "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789_-"),
mid(b64map,a,1) & mid(b64map,b,1) & mid(b64map,c,1) & mid(b64map,d,1))
```

* `QUERY()` is also very powerful + flexible in certain circumstances though I don't use it in this project.