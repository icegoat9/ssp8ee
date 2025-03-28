Keeping a rough scratchpad of some running dev notes here, rather than in the [README.md](README.md) which was already too long.

# PICO8 Education Edition @URL Format (WIP)

This is the format created by SAVE @URL in PICO-8, which saves a sufficiently-small program as a URL that can be shared with other users of the PICO8 Education Edition.

I didn't find this format documented anywhere, so here are running notes from a few hours of trial-and-error investigating how it works. Anything below could have errors (or could become obsolete with changes after March 2025 when I'm writing this).

`https://www.pico-8-edu.com/?c=FOO&g=BAR`

Where, with a few [caveats](#b64-nonstandard-encoding), FOO is the b64 urlsafe-encoded code and BAR is the encoded spritesheet data. `&g=...` can be omitted if not used.

By looking at how the encoded @URL changes if I change single characters of code or pixels in PICO8, there's also some compression applied to the input before the b64 encoding (some form of run-length encoding at least for sprite data-- most obvious if you have a row of pixels of the same color). However, the P8 Edu Edition can accept uncompressed data in the URL as well, which is what I started with just to get something working quickly.

## Spritesheet

Using the SAVE @URL command on some simple carts and looking at the generated URL, I see that:

* A single dark blue (color 1) spritesheet pixel encodes as `B` (the b64 encoding for value 1)
* A row of 8 pixels in colors 1-8 encodes as `BCDEFGHI` (b64 for 1-8). Interestingly, this isn't b64 encoding the spritesheet memory (e.g. it doesn't seem to be packing the data into a series of 8-bit bytes and then b64 encoding that into 4/3 as many 6-bit characters), but instead is using the b64 mapping on each pixel. That's simpler to read, at least.
* However, a row of 8 dark blue (1) pixels encodes as `xE` rather than `BBBBBBBB`. There's some run-length encoding going on that stores identical consecutive pixels (only) in a more compact format, as a row of 8 alternating pixel values of 1,2,1,2,1,2,1,2 is encoded as the uncompressed `BCBCBCBC`.
  * This makes sense as a design choice, because otherwise all the black background pixels in the spritesheet would take up an enormous amount of URL space with an A for each one...

### Spritesheet Compression

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

| Row of 8 sprite pixel values     | @URL b64 encoding | Decimal | Binary (byte pair order swapped) |
| -------- | --- | ---- | ------------- |
| 1111     | xA | 49,0 | 000000,110001|
| 11111    | xB | 49,1 | 000001,110001 |
| 111111   | xC | 49,2 | 000010,110001 |
| (8x) 1 | xE | 49,4 | 000100,110001 |
| (16x) 1  | xM | 49,12 | 001100,110001 |
| (64x) 1  | x8 | 49,60 | 111100,110001 |
| (67x) 1  | x- | 49,63\* | 111111,110001 |

We see that the single b64 value in the previous table can only encode up to 3 consecutive pixels of a color (2 bits for # of pixels, 4 bits for color #). Once we need to encode more than 3 pixels there's something more going on.

If we swap the order of the bytes, it looks like the encoding of N pixels of value V is something more complicated. 
Let's define A as the 6-bit value `MAX(N-4,0)` and B as the 2-bit value `IF(N>4,4,N)`. Then the encoding appears to be: `{ A }{ B-1 }{ V }`

That is, use the two bits remaining in the character that indicates color to count 1-3 pixels, and if there are more, set these two bits to indicate 4 and add another character that encodes the "# of pixels above 4". It's an interesting optimization to not use all of the bits (including the 2 in the first character) to encode an 8-bit value, but I can see it makes the decoder more straightforward: the decoder can treat any character whose high bits are not '11' as representing 1-3 pixels, and if it sees '11' in the high bits it knows to look to another character for the # of pixels.

Looking at even longer pixel patterns that need three or more b64 characters:

| Row of 8 sprite pixel values     | @URL b64 encoding | Decimal (tbd byte order) | Binary (byte order swapped) |
| -------- | --- | ---- | ------------- |
| (67x) 1  | x- | 49,63\* | 111111,110001 |
| (68x) 1  | x-B | 49,63\*,1 | 000001,111111,110001, |
| (69x) 1  | x-R | 49,63\*,17 | 010001,111111,110001, |
| (72x) 1  | x-xB | 49,63\*,49,1 | 000001,110001,111111,110001 |

At first it was puzzling that we jumped so quickly from three to four characters, but then I see it's just the pattern above, repeated. So it appears that we're not trying to encode arbitrarily long sequences of the same pixel: each pair of b64 characters can represent up to 67 pixels, and if we need 68 or more pixels we just start the encoding over. This uses slightly more characters than a more complex scheme in a few circumstances (in particular, if you have between 70 and 139 pixels of a color in a row this uses 4 characters rather than a theoretical 3 you could use with a different encoding scheme), but it keeps the decoder simpler. Also, in PICO8 applications I think a common reason to have many pixels of the same color is the black background that wraps around. If you're only using the first 4 sprites, for example, you have 96 black pixels in each 128-pixel-wide spritesheet row. And I imagine the main goal is to encode these in just a few characters (`w-wZ` encodes 96 black pixels). It doesn't really matter if this takes 3 or 4 characters when the goal is not to use 96 characters.

*Why are the 63s above labeled 63\*?* See the next section...

## b64 nonstandard encoding

I had my @URL Generator Spreadsheet mostly working, but was seeing errors where '>' and '?' characters (CHRs 62 and 63) in code were sometimes (but not always) swapped when opening a cart link generated by the spreadsheet in Education Edition. Adding or removing whitespace in the cart seemed to make the problem go away?! 

Eventually I discoverd that PICO8's @URL format seems to swap the meaning of '-' and '_' in its b64 encoding compared to the typical urlsafe encoding. Those represent values 62 and 63, so we'd expect a common decoding issue when > or ? characters line up with three-character boundaries in the input data (we'd also see this for other combinations of characters but mostly less used extended characters).

I don't know if this is some light obfuscation or was unintentional, but I've adjusted the spreadsheet wrapper and all the below notes on decoding to match what PICO8's URL format seems to expect.

So, specifically, for the PICO8 @URL encoding I'm using this b64 mapping of values 0-63:
`ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789_-`  (not `-_`)

### Padding

**Side note:** In the 'spreadsheet IDE', rather than doing proper b64 encoding by adding padding '=' characters if the input length isn't a multiple of 3 (since each triplet of three bytes becomes 4 b64 values), I pad the input with spaces to a multiple of three because that was the easiest way to get running. This works fine for text data where a few extra spaces at the end don't matter, but would be incorrect for arbitrary data...

## @URL Code Encoding

I've done some similar trial and error tests of encoding simple program data and seeing what the output looks like. I've discovered there's some compression I don't fully understand applied to all but the most trivial code. However, before I dive into those details, this compression seems to be optional and indicated to PICO8 Edu by a specific header added to the data.  That is, if I simply b64 encode (with the caveat above) program code and pass it as a @URL parameter, PICO8 Edu Edition does successfully interpret it, it's just much less space-efficient. So for my proof-of-concept spreadsheet IDE I'm going to ignore compression for now to get it working.

### Encoding Investigation

For reference, the character '0' is CHR(48) and '?' is CHR(63), chosen for having recognizable binary representations of '00110000' and '00111111'. 

I show padding characters = as just unknown bits x for now.

| Program text | @URL b64 encoding | b64 Decimal | Binary |
| -------- | --- | ---- | ------------- |
| 0        | MA== | 12,0    | 001100,00xxxx,xxxxxx,xxxxxx |
| 00       | MDA= | 12,3,0  | 001100,000011,0000xx,xxxxxx|
| 000        | MDAw | 12,3,0,48 | 001100,000011,000000,110000 |
| 000000        | MDAwMDAw |     | encoding of 000 repeated 2x |
| ?        | Pw== | 15,48     | 001111,11xxxx,xxxxxx,xxxxxx |
| ??        | Pz8= | 15,51,60     | 001111,110011,1111xx,xxxxxx |
| ??????        | Pz8-Pz8- | 15,51,60,63     | 001111,110011,111100,111111 |
| 0?0?0? | MD8wPzA- | 12,3,60,48,15,51,0,63 | 001100,000011,111100,110000,001111,110011,000000,111111 |

Ah, in these simple short snippets we can see the binary patterns 00110000 and 00111111 repeating, it's a simple direct b64 encoding without compression.

However, with longer repeating strings there's some addition compression being done on the input before b64 encoding:

| Program text | @URL b64 encoding | Decimal | Binary |
| -------- | --- | ---- | ------------- |
| 000        | MDAw | 12,3,0,48 | 001100,000011,000000,110000 |
| 0 (9x)        | MDAwMDAwMDAw |     | just the above repeated 3x |
| 0 (12x)        | AHB4YQAMAAsHGDw= |     | **not** the 000 encoding repeated 4x! |
| 0 (24x)        | AHB4YQAYAAwHGPwG |     | **not** the 000 encoding repeated 8x! |

Further looking at outputs on a few realistic code snippets, some see straightforward b64 encoding and some don't:

| Program text | @URL b64 encoding | Does this trivially b64 decode to the input string? |
| ------------ | --- | ------------- |
| cls(1) circfill(50,50,30,8) | Y2xzKDEpIGNpcmNmaWxsKDUwLDUwLDMwLDgp | Yes |
| cls(1)cls(2)cls(3) | Y2xzKDEpY2xzKDIpY2xzKDMp | Yes |
| cls(1)cls(1)cls(1) | AHB4YQASABE3H-8G20esu1w= | No! |
| cls(1)cls(1)cls(1)cls(1)cls(1)cls(1)cls(1)cls(1)cls(1) | AHB4YQA2ABM3H-8G20esu-z-Pw== | No! |

The example with clearly repeating code snippets encodes to something that decodes as binary data, again suggesting some compression routine has been run that looks for common multi-character substrings and replaces them with some token. You can also see why, as the code with many copies of cls(1) is barely larger than the smaller code. This is a good stopping point for now.

### Further Investigation

One more intriguing hint. If I b64 decode the earlier string `AHB4YQASABE3H-8G20esu1w=`, I get binary data but with the text string PXA near the beginning. I recall that PX8 and PX9 were sample lightweight compression libraries [published by Zep on the BBS](https://www.lexaloffle.com/bbs/?tid=34058), so I'll speculate that PXA is a next-gen version used to compress data for the @URL format and this is a header that tells the decoder built in to PICO8 Edu what compression is used?

In particular, the first few output values in the b64-decoded version of `cls(1)cls(1)cls(1)` are `239,191,129,112,120,97,239,191,189,18,239,191,189,...`, and 112,120,97 is the string "PXA". So perhaps 239,191,129 indicates separators on a header, "PXA" is the compression method, and the standalone number "18" is a version (or length, or some other config value)? I could also just ask @zep and see if he wants to share, but this has been a fun diversion for now.

| Program text | @URL b64 encoding | without prefix AHB4YQ__ |
| ------------ | --- | --- |
| 0 (12x)  | AHB4YQAMAAsHGDw=     | AAsHGDw= |
| 0 (24x)  | AHB4YQAYAAwHGPwG     | AAwHGPwG |
| 0 (72x)  | AHB4YQBIAA4HGPz--78= | AA4HGPz--78= |
| 1 (72x)  | AHB4YQBIAA4XGPz--78= | AA4XGPz--78= |
| 2 (72x)  | AHB4YQBIAA4nGPz--78= | AA4nGPz--78= |
| ? (72x)  | AHB4YQBIAA73GPz--78= | AA73GPz--78= |
| 1 (257x) | AHB4YQEBABkXGPz---------------9-AA== | ABkXGPz---------------9-AA== |

From this minimal data it appears the header of this format is AHB4YQA__A, where __ is an encoding of the length of the encoded data as a 64-bit number (in the above examples, M=12, Y=24, BI=72, and EB=257, if interpreted as 64-bit numbers using b64 translation). For now I'll consider the trailing A of that prefix part of the data (as it makes the post-header data section a nice multiple of 4 characters in length).

We can also see some structure in the rest of the data by eye (not even yet decoding this back to 8-bit numbers), to look at further another day. The significant increase in encoded length between 1(72x) and 1(257x) is a bit surprising, at first...

A few more input->output examples for simple repeating inputs and what values they decode to if they are byte arrays encoded as b64 (with the help of https://www.fumdum.com/convert/base64), just to check if the character or it's repeat quantity show up obviously in the data (as they did in the simple pixel run length encoding above). 

| Program text  | @URL b64 without prefix AHB4YQ__ | b64 decoded to decimal? | b64 raw values |
| ------------- | --- | --- | --- |
| 0 *(char #48) (12x)*  | AAsHGDw=     |  000 011 007 024 060                  | 0 0 44  7  6  3 48 |
| ◝ *(char #255) (12x)* | AAzfg8ED     | 000 012 223 131 193 003               | 0 0 51 31 32 60  4  3 |
| ◝ *(13x)*             | AAzfg8EF     | 000 012 223 131 193 005               | 0 0 51 31 32 60  4  5 |
| ◝ *(14x)*             | AAzfg8EH     | 000 012 223 131 193 007               | 0 0 51 31 32 60  4  7 |
| ◝ *(24x)*             | AAzfg8Fv     | 000 012 223 131 193 111               | 0 0 51 31 32 60  4 47 |
| ◝ *(25x)*             | AA3fg8F-AA== | \*000 013 223 131 193 127 000        | 0 0 55 31 32 60  4 63  0  0 |
| ◝ *(72x)*             | AA-fg8H---8L | \*000 015 223 131 193 255 255 255 011 | 0 0 63 31 32 60  6 63 63 63 60 11 |
| 0 *(12x)* ◝ *(12x)*   | AA8HGDzfg8ED | 000 015 007 024 060 223 131 193 003   |  |
| {0◝} *(12x)*           | AA4HfA8OfwE  | 000 014 007 124 015 014 127 001       |  |
| 1 *(257x)* | ABkXGPz- *(15x)* 9-AA== | \*000 025 023 024 252 255 *(12x)* 127 000 |  |

(\* not sure I got these decodings right with the online tool I used: I quickly swapped in a / for the - character as that's what the online decoder expected for value 63)

Unfortunately the encoding is not immediately obvious by eye (it would be a smoking gun for a very simple RLE scheme if we saw the values 255 and 12/13/14/24/25/72 encoded somewhere for the strings that include repeats of char 255 that number of times), though it's also possible I'm reading the byte order swapped or with an offset.

When looking for a byte swap or bit shift in data, I often quickly scan for values that seem to be 1-2 bits shifted from a significant value.

For example, it's slightly suggestive that all of the encodings of char#255 feature 223 in the output which is 255-32 (one bit off), and encoding of char #48 includes a 24 in that same position which is one bit shift from 48, though with an offset in b64 decoding (which shifts between 8-bit and 6-bit values) I'd expect an error to lead to a 2-bit offset. Looking at the string of decimal or b64 values as a long binary string doesn't show any sequences of 8 consecutive 1's at any offset so it's not as simple as just an offset.

# Google Sheets dev-notes-to-self

It's possible to build a lot of functionality in spreadsheets (for example, a string->b64 encoder), even without incorporating scripting (AppScript, VBA, etc). A few more modern spreadsheet functions have made that easier than it was a decade ago in the time of Array Formulas and intermediate-data worksheets and so on:

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