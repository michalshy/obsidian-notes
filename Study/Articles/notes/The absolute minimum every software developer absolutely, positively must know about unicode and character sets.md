link: https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/
## Beginning
In the days when Unix was invented, the only characters set that mattered were unaccented English letters, so ASCII code was able to represent every character using number from 32 to 127.
It was good, assuming everyone was English speaker.
The problem is, other numbers in 8-bit structure, so numbers 128-255 were used by people to develop their own ideas about character set.
In this manner, OEM was developed, which provided a bunch of line drawing characters (horizontal lines, vertical bars etc). 
Finally, those upper numbers were introduced in ANSI. In the ANSI standard, everybody agreed what to do below 128 with the characters, but upper numbers had lots of different ways to be handled, depending on *code pages*. For example Israel DOS uses code page 862, Greek users used 737.
In Asia people were using "double byte character set" which presented some letters as one byte and other as two bytes.
> Programmers were encouraged not to use s++ and s– to move backwards and forwards, but instead to call functions such as Windows’ AnsiNext and AnsiPrev which knew how to deal with the whole mess.

Luckily, unicode had been invented
## Unicode
In Unicode, letter maps to *code point*. 
Every platonic letter in every alphabet is assigned magic number by the Unicode consortium which is written like this: **U+0639**. This is *code point*. 
Okay, so we have a string `Hello` which in Unicode would be five code points: `U+0048`, `U+0065`, `U+006C`, `U+006C`, `U+006F`. How to store them in memory?
### Encoding
First implementation were just storing those numbers in 2 bytes. So
`hello`
would be
`00 48 00 65 00 6C 00 6C 00 6F`.
This led to including of *Unicode Byte Order Mark*, which was two bytes at the beginning informing of endian. In normal case bytes were `FE FF`, but if you would swap bytes it would be `FF FE`.
People were also complaining about zeros in encoding. This could be optimized!
This is where UTF-8 come from. In UTF-8 every code point from 0-127 is stored in single byte. Only code points above are stored using 2, 3, or up to 6 bytes.

Moral is:
Always check encoding of string when implementing operations on them. Not everything is ASCII
