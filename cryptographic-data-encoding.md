# Cryptographic Data Encoding
* Version: v0.0.5 Pre-Draft
* Date: July 7, 2021
* License: CC BY 4.0
* Author: Dave Huseby <dwh@trustframe.com>
* Copyright: (c) TrustFrame, Inc. 2021

UCC is for encoding of cryptographic data as into a self-describing format using a method that simplifies both ASCII (base 64) and binary (base 2) serialization/deserialization. It has some additional novel properties (e.g. easy visual inspection of ASCII, simple parsing in binary) that make it especially useful for encoding cryptographic data.

The primary, and most important, goal is that the ASCII representation of every construct begins with easy-to-learn characters associated with the different classes (e.g. cryptographic keys) and sub-classes (e.g. Ed25519 keys) to aid in debugging and visual inspection. Since most cryptographic constructs have three levels of classification (i.e. key -> algorithm -> variant/bit length) the type tags for UCC have three layers of classification: class, sub-class, and sub-sub-class. The secondary goal is that the binary representation of every construct has an easily parsed type tag to simplify serialization and deserializtion.

To understand this spec you have to think in terms of "encoding units". An encoding unit is 24-bits of data organized as either a group of four 6-bit base 64 words (e.g. 000000 111111 000000 111111) or a group of three 8-bit words (e.g. 00000000 11111111 00000000). The group of four 6-bit words is used for encoding the data using ASCII characters in the same way that Base64 encoding works. Each of the 6-bit words maps to an ASCII character in a table of 64 characters that is similar to the Base64 table but is unique to UCC. The result is that each encoding unit is represented by four ASCII characters in the text encoding. The group of three 8-bit words is used for encoding the data when using binary. Either way, each encoding unit is 24-bits in length making it easy to encode any arbitrary data in either form depending on the system.

The diagram below illustrates how each 24-bits encoding unit is split into either four, 6-bit words for the text encoding or three, 8-bit words for the binary encoding.

```
         /--1--/--2--/--3--/--4--/  four, 6-bit words for text encoding
24 bits: xxxxxxxxxxxxxxxxxxxxxxxx
         \---1---\---2---\---3---\  three, 8-bit words for binary encoding
```

## Streaming

UCC is designed to facilitate streaming of one or more cryptographic constructs in either the text or binary encoding. Each construct is self-describing with a type tag that contains both the type and length, in bytes, of the encoded cryptographic construct. The diagram below illustrates the structure of a UCC stream.

```
/-- type tag --/-- encoded data --/-- type tag --/-- encoded data --/
```

To support encoding complex cryptographic constructs, UCC designates a few special type tags for lists of constructs as well a construct for un-typed arbitrary data. UCC lists are recursive and can contain other lists allowing for any complex cryptographic construct to be encoded.

### Parsing

When parsing the text representation of a UCC encoded cryptographic construct a parser must only accept characters in the UCC encoding alphabet and ignore all other characters, including whitespace. This allows for a text encoded UCC construct to be arbitrarily formatted for easier reading without affecting the parsing of the construct. This is specifically to allow for line wrapping with carriage returns and new line characters as well as other common line wrapping techniques such as adding a trailing '/' before a newline or indenting with tabs and/or spaces.

Software that consumes UCC encoded constructs must be tolerant of, and ignore any encoded constructs with types that it doesn't support. If the software encounters a class, sub-class, or even a sub-sub-class that it does not know how to work with, it must parse the length and skip over the construct entirely as if it wasn't there.

## Type Tags

Every UCC object begins with a type tag that is at least one encoding unit and optionally may include a second encoding unit to extend the length field to support larger constructs. The alignment of the fields is carefully chosen to facilitate easy serialization/deserialization in both its text and binary encodings. The diagram below illustrates the interpretation of the type tag when encoded into 6-bit words for the text representation.

```
       encoding unit 1          optional encoding unit 2
 /--------------------------/ /--------------------------/
/--1--//--2--//--3--//--4--/ /--5--//--6--//--7--//--8--/. 6-bit words
fccccc fsssss fsssxx xxxxxx  xxxxxx xxxxxx xxxxxx xxxxxx
||   | ||   | || ||       |                            |
||   | ||   | || |+-------|----------------------------+.. extended length
||   | ||   | || |+-------+............................... short length
||   | ||   | |+-+........................................ sub-sub-class
||   | ||   | +........................................... ext. length flag
||   | |+---+............................................. sub-class
||   | +.................................................. exp. sub-class flag
|+---+.................................................... class
+......................................................... exp. class flag
```

This second diagram illustrates the interpretation of the type tag when encoded into 8-bit words for the binary representation.

```
       encoding unit 1         optional encoding unit 2
 /-------------------------/ /-------------------------/
/---0---//---1---//---2---/ /---3---//---4---//---5---/... 8-bit words
fcccccfs ssssfsss xxxxxxxx  xxxxxxxx xxxxxxxx xxxxxxxx
||   |||    ||| | |      |                           |
||   |||    ||| | +------|---------------------------+.... extended length
||   |||    ||| | +------+................................ short length
||   |||    ||+-+......................................... sub-sub-class
||   |||    |+............................................ ext. length flag
||   ||+----+............................................. sub-class
||   |+................................................... exp. sub-class flag
|+---+.................................................... class
+......................................................... exp. class flag
```

The first two 6-bit words are dedicated to the class and sub-class of the construct. Each of them has an experimental class/sub-class flag as the most significant bit. Normally, detecting single bit changes in the text encoding is very difficult and complicated in text. However UCC uses a novel organization of the ASCII characters in the base-64 encoding table such that when the experimental flag is 0 the text encoding is a lower case letter or a number 0-5 and when the experimental flag is 1, the text encoding is an uppercase letter or a number 6-9 or the hyphen or underscore. The important thing to remember when processing text encodings is that if you don't care about the experimental status, you can lower-case any letters and mod 6 any numbers to lower-case everything before processing.

The extended length flag and the sub-sub-class value take up the most significant 4 bits in the third 6-bit word. This design is a compromise to meet the needs of both the text representation and the binary representation. By assigning the class and sub-class fields to the first two 6-bit words, every cryptographic construct of the same class and sub-class will begin with the same ASCII characters and the case of those characters signifies the experimental status of the class and sub-class. This makes it easy for humans to learn these codes and to "read" them visually and know what type of cryptographic construct is encoded.

The 1-bit extended length flag is the most significant bit of the third 6-bit word so that the case of the third letter signifies if the type tag is just one encoding unit or if it is a two encoding unit type tag and the construct has a 32-bit length. Again, this is to allow humans to "read" the third character and know if this is a small or large construct. The 1-bit extended length flag and the 3-bit sub-sub-class and the two 6-bit class and sub-class fields are designed to all fit into two 8-bit words of the binary representation. The short length field then fits into the last 8-bit word of the encoding unit. So, along with easy visual inspection when in the text representation, we also get a simple bitfield data structure for processing the binary representation.

One of the hardest things to do with text encodings is bit manipulation using a programming language that doesn't support it natively like C or Rust. Thankfully, even in this situation, a little intuition and math makes it possible to work with the different bits in the UCC type tag, specifically in the third 6-bit word. The following code illustrates how to use Javascript to extract the sub-sub-class bits as well as the two size bits from the third 6-bit word so that they can be combined with the fourth 6-bit word to calculate the length of the data in bytes.

```javascript=
// assume the third letter in a type tag is 'o' and the fourth letter is 'H'
// 'o' is index 14 (001110 in binary) and 'H' is index 39 (100111 in binary)
// when decoded from the character the index value is the result
let v1 = 14; // decode('o') -> 14
let v2 = 39; // decode('H') -> 39

// to get the sub-sub-class, we simply divide by 4 and round down
let ssc = Math.floor(v1 / 4); // gives 3, or 011 in binary

// to get the length in bytes, we use modulus, multiplication and addition
let two_msb = v1 % 3; // isolates the two lowest bits of the third 6-bit word
let length = (two_msb << 6) + v2; // gives 167, or 10100111 in binary
```

Even with the complicated bit pattern in the third 6-bit word, the UCC type tag layout is entirely usable. The other 1-bit flags for testing if a class or sub-class is experimental or if the type tag is a short or extended type tag, the code is even simpler. Because of the novel layout of the UCC character encoding table, simply looking at the letters to see if they are upper-case reveals the value of the 1-bit flags. If you want to write code, you must decode the letter to an integer and then test if the value is >= 32. The following code demonstrates this.

```javascript=
// assume the first letter for the class is 'K'. 'K' decodes to the value 42
// which is 101010 in binary.
let cls = 42; // decode('K') -> 42

// it is simple to determine if the code is experimental
if (cls >= 32) {
  // experimental
} else {
  // standard
}
```

### Length

The length of an encoded cryptographic construct is counted in the number of 8-bit bytes. Because the length of a cryptographic construct might not be a multiple of three bytes, there are potentially one or two padding bytes necessary to encode from the binary to the text representation. UCC does not define, nor does it preserve padding bytes in the text representation therefor the length value must be the number of bytes so that software that processes UCC text encoded constructs knows how many pad bytes are implied.

Given the length of the construct in bytes, the number of text characters to read after the type tag is calculated with the following code. This correctly accounts for the missing padding characters and will not read too many characters.

```c=
size_t length_in_text = ((length_in_bytes << 3) + 5) / 6;
```

If your programming language of choice does not support the bit shift operator `<<` you may just multiple the length value by 8.

```javascript=
let length_in_text = Math.floor(((length_in_bytes * 8) + 5) / 6);
```

The type tag has two ways of storing the length of the cryptographic construct: a short length 8-bit field or an extended length 32-bit field. If the size of the construct is 255 bytes or fewer, then the length value fits into the 8-bit short length field. In that case an extended length is not required and the extended length flag field is set to 0 and the third ASCII character will be lower case in the text representation. If the size of the construct is greater than 255 bytes then the construct requires an extended type tag and the third ASCII character will be upper case in the text representation. The length of the construct is encoded as a 32-bit value using the 8-bit short length field in the first encoding unit and the 24-bits in the second encoding unit to form a 32-bit extended length field. The maximum size for a cryptographic construct encoded in UCC format is 4,294,967,295 bytes, or roughly 4 GB.

The following code declares a bitfield struct for simple processing of an extended type tag in C.

```c=
#include <stdint.h>

struct {
  uint32_t        :  8;  /* unused */
  uint32_t ecls   :  1;  /* exp. class */
  uint32_t cls    :  5;  /* class */
  uint32_t escls  :  1;  /* exp. subclass */
  uint32_t scls   :  5;  /* sub-class */
  uint32_t ext    :  1;  /* extended length */
  uint32_t sscls  :  3;  /* sub-sub-class */
  uint32_t len    :  8;  /* length */
} type_tag_t;

struct {
  uint64_t        : 16;  /* unused */
  uint64_t ecls   :  1;  /* exp. class */
  uint64_t cls    :  5;  /* class */
  uint64_t escls  :  1;  /* exp. sub-class */
  uint64_t scls   :  5;  /* sub-class */
  uint64_t ext    :  1;  /* extended length flag */
  uint64_t sscls  :  3;  /* sub-sub-class */
  uint64_t len    : 32;  /* extended length */
} ext_type_tag_t;
```

## Lists and Non-Typed

UCC is specifically designed to support encoding arbitrarily complex cryptographic data structures/constructs. The simplest way to achieve that is to support recursive lists. UCC achieves this using the `-` (hyphen) character (e.g. 0b111110 6-bit word) to indicate that the encoded construct is a list of other constructs. The `-` character is only used in the sub-class field to indicate a list of constructs of the given class.

Sometimes it is useful to have a list of constructs of mixed types. UCC supports this by using the `_` (underscore) (e.g. 0b111111 6-bit word) to indicate that the class or sub-class is not specified.

By using the `-` and `_` as class and sub-class in different ways, UCC supports lists of constructs of a specific class (i.e. "typed lists") by using a class code followed by a `-`, lists of non-typed constructs (i.e. "non-typed lists" containing a mix of classes) by using `_-`, lists of lists by using `--`, and also a non-list/single, non-typed construct for storing arbitrary data by using `__`.

### Typed List

Typed lists are specified by combining a class code other than `-` or `_` with `-` as the sub-class code. The type tag for lists is interpreted in the same way as described above except that the length field does not specify the number of encoding units for the list but instead specifies the number constructs in the list. List constructs can also take advantage of the extended length field giving them a maximum capacity of 12 billion list items. The diagram below illustrates how a text-encoded typed list tag is interpreted.

```
       encoding unit 1          optional encoding unit 2
 /--------------------------/ /--------------------------/
/--1--//--2--//--3--//--4--/ /--5--//--6--//--7--//--8--/. 6-bit words
fccccc 111110 fsssxx xxxxxx  xxxxxx xxxxxx xxxxxx xxxxxx
||   | |    | || ||       |                            |
||   | |    | || |+-------|----------------------------+.. extended length
||   | |    | || |+-------+............................... short length
||   | |    | |+-+........................................ unused
||   | |    | +........................................... ext. length flag
||   | +----+............................................. '-' list sub-class
|+---+.................................................... class
+......................................................... exp. class flag
```

### Non-Typed List

Non-typed lists are specified by combining the `_` class code with the `-` sub- class code. These are useful for constructing lists of constructs of different types. Just like typed lists described above, the length field in the type tag specifies the number of items in the list. Technically, using `-` as the class and `_` as the sub-class also defines a list of non-typed constructs but this specification choosed to define that as an invalid type tag and reserve it for future use. Standard compliant software should ignore constructs with `-_` class and sub-class just like any other constructs the software does not support. The diagram below illustrates how a text-encoded non-typed list tag is interpreted.

```
       encoding unit 1          optional encoding unit 2
 /--------------------------/ /--------------------------/
/--1--//--2--//--3--//--4--/ /--5--//--6--//--7--//--8--/. 6-bit words
111111 111110 fsssxx xxxxxx  xxxxxx xxxxxx xxxxxx xxxxxx
|    | |    | || ||       |                            |
|    | |    | || |+-------|----------------------------+.. extended length
|    | |    | || |+-------+............................... short length
|    | |    | |+-+........................................ unused
|    | |    | +........................................... ext. length flag
|    | +----+............................................. '-' list sub-class
+----+.................................................... '_' non-typed class
```

### List of Lists

A list of lists is really just a typed list with the `-` list code as the class and `-` as the sub-class. This is not a special case but warrants highlighting it as a valid case that UCC compliant software should handle. Just like the typed list described above, the length field encodes the number of lists in the list. The diagram below illustrates how a text-encoded list of lists tag is interpreted.

```
       encoding unit 1          optional encoding unit 2
 /--------------------------/ /--------------------------/
/--1--//--2--//--3--//--4--/ /--5--//--6--//--7--//--8--/. 6-bit words
111110 111110 fsssxx xxxxxx  xxxxxx xxxxxx xxxxxx xxxxxx
|    | |    | || ||       |                            |
|    | |    | || |+-------|----------------------------+.. extended length
|    | |    | || |+-------+............................... short length
|    | |    | |+-+........................................ unused
|    | |    | +........................................... ext. length flag
|    | +----+............................................. '-' list sub-class
+----+.................................................... '-' list class
```

## Non-Typed Data Construct

By using the non-typed code `_` for both the class and sub-class, UCC supports encoding arbitrary data as a single non-typed construct. The diagram below illustrates how a text-encoded non-typed construct tag is interpreted.

```
       encoding unit 1          optional encoding unit 2
 /--------------------------/ /--------------------------/
/--1--//--2--//--3--//--4--/ /--5--//--6--//--7--//--8--/. 6-bit words
111111 111111 fsssxx xxxxxx  xxxxxx xxxxxx xxxxxx xxxxxx
|    | |    | || ||       |                            |
|    | |    | || |+-------|----------------------------+.. extended length
|    | |    | || |+-------+............................... short length
|    | |    | |+-+........................................ unused
|    | |    | +........................................... ext. length flag
|    | +----+............................................. '_' non-typed sub
+----+.................................................... '_' non-typed class
```

## Extensibility

Normally, an encoding like this uses a standard Base64 alphabet for encoding the 6-bit base 64 numbers as ASCII characters. But since we have a hard requirment for extensibility, this standard uses a slightly modified Base64 alphabet derived from the URL-safe variant. The UCC variant places the lower-case letters and first six numeric characters in the first 32 index places with the upper-case and last four numbers followed by hyphen and underscore in the last 32 index places. This new arrangement makes the most significant bit a flag that can be tested easily in binary and determined visually in text (i.e. upper case vs lower case letters, lower number vs upper numbers).

Since the class and sub-class text encoding is designed to be the same for all constructs of a given class (i.e. all cryptographic keys begin with `k`) and sub-class (i.e. all Ed25519 keys begin with `ke`) by re-arranging the encoding alphabet for UCC, it supports experimental extensions by using upper case versions of the same letters. For instance, 4,096-bit RSA keys are encoded such that their text representation always begins with `kr` (i.e. `k` for key, `r` for RSA) but experimental 16,384-bit RSA keys always begin with `Kr`. The former says the construct is a standard RSA key (e.g. 4,096-bit RSA key) and the latter says the construct is an experimental RSA key (e.g. 16,384-bit RSA key).

Below is a table comparing the URL-Safe Base64 (B64) alphabet and the UCC alphabet. Note how the lowercase letters in the UCC alphabet begin at index 0 instead of index 26 as they do in Base64. This novel rearrangement makes it trivial to identify experimental classes and sub-classes in both text encoded and binary encoded representations.

```
Index  Binary  B64  UCC      Index  Binary  B64  UCC
0      000000   A    a       32     100000   g    A
1      000001   B    b       33     100001   h    B
2      000010   C    c       34     100010   i    C
3      000011   D    d       35     100011   j    D
4      000100   E    e       36     100100   k    E
5      000101   F    f       37     100101   l    F
6      000110   G    g       38     100110   m    G
7      000111   H    h       39     100111   n    H
8      001000   I    i       40     101000   o    I
9      001001   J    j       41     101001   p    J
10     001010   K    k       42     101010   q    K
11     001011   L    l       43     101011   r    L
12     001100   M    m       44     101100   s    M
13     001101   N    n       45     101101   t    N
14     001110   O    o       46     101110   u    O
15     001111   P    p       47     101111   v    P
16     010000   Q    q       48     110000   w    Q
17     010001   R    r       49     110001   x    R
18     010010   S    s       50     110010   y    S
19     010011   T    t       51     110011   z    T
20     010100   U    u       52     110100   0    U
21     010101   V    v       53     110101   1    V
22     010110   W    w       54     110110   2    W
23     010111   X    x       55     110111   3    X
24     011000   Y    y       56     111000   4    Y
25     011001   Z    z       57     111001   5    Z
26     011010   a    0       58     111010   6    6
27     011011   b    1       59     111011   7    7
28     011100   c    2       60     111100   8    8
29     011101   d    3       61     111101   9    9
30     011110   e    4       62     111110   -    -
31     011111   f    5       63     111111   _    _
```

## Proposed Classes

The most common set of cryptographic constructs that have wide adoption and should have their own classes are:

- aead (i.e. authentic encryption with associated data)
- cipher (i.e. encrypted data)
- digest
- hmac
- identifier
- key
- nonce
- policy (i.e. smart contract, script, etc)
- signature

Because none of these names share a common first letter so it is trivial to assign a unique character to each. Each class will have both an upper case letter and a lower case letter assigned to it for "standard" and "experimental" sub-classes respectively.

### Application Specific Extensions

UCC is designed to support constructing software systems that may want to define their own classes and sub-classes for common constructs instead of explicitly constructing them out of recursive lists. For instance, cryptographic provenance logs are made up of cryptographic events. In software that uses UCC to store logs of events, provenance log constructs can use `L` for log--upper-case for experimental--and then give the event construct a sub-class `E` for event, also upper-case for experimental. Then all provenance log event constructs encoded in text begin with `LE`. This is an alternative to defining the log event constructs explicitly out of recursive lists, obfuscating the type.

In many ways, developers start off by explicitly defining their constructs as recursive list constructs during the early prototyping phase. Then as the common constructs become known, they are assigned their own experimental class and sub-class to make their encodings more pedantic and easier to read and debug. Then as the software gains wide adoption and the constructs become widely used, the experimental class code is promoted to a lower-case, non-experimental when it becomes standardized.

The following table lists the characters assigned to each class of widely used cryptographic constructs.

```
Class      Standard      Experimental
AEAD       'a'           'A'
Cipher     'c'           'C'
Digest     'd'           'D'
HMAC       'h'           'H'
Identifier 'i'           'I'
Key        'k'           'K'
Nonce      'n'           'N'
Policy     'p'           'P'
Signature  's'           'S'
List       '-'           N/A
Non-Typed  '_'           N/A
```

## Proposed Sub-Classes

### AEAD

```
Sub-Class               Standard
AES256-GCM              'a'
ChaCha20-Poly1305       'c'
ChaCha20-Poly1305-IETF  'i'
XChaCha20-Poly1305-IETF 'x'
```

### Cipher

```
Sub-Class     Standard
AES           'a'
XChaCha20     'x'
```

#### AES

```
Sub-Sub-Class   Value
AES-256         0
AES-128         1
AES-192         2
```

### Digest

```
Sub-Class     Standard
Blake2        'b'
MD            'm'
SHA1          '1'
SHA2          '2'
SHA3          '3'
```

#### Blake2

```
Sub-Sub-Class   Value
Blake2b         0
Blake2s         1
```

#### MD

```
Sub-Sub-Class   Value
MD5             0
MD4             1
MD2             2
MD6             3
```

#### SHA2

```
Sub-Sub-Class   Value
SHA2-256        0
SHA2-512        1
SHA2-224        2
SHA2-384        3
SHA2-512/224    4
SHA2-512/256    5
```

#### SHA3

```
Sub-Sub-Class   Value
SHA3-256        0
SHA3-512        1
SHA3-224        2
SHA3-384        3
SHAKE128        4
SHAKE256        5
```

### HMAC

```
Sub-Class     Standard
```

### Identifier

```
Sub-Class     Standard
ADI           'a'
DID           'd'
Email         'e'
```

### Key

```
Sub-Class     Standard

Ed25519       'e'
RSA           'r'
```

#### Ed25519

```
Sub-Sub-Class   Value
Public Key      0
Secret Key      1
```

#### RSA

```
Sub-Sub-Class   Value
Public Key      0
Secret Key      1
```

### Nonce

```
Sub-Class     Standard
```

### Policy

```
Sub-Class     Standard
Bitcoin       'b'
CCLang        'c'
Solidity      's'
```

### Signature

```
Sub-Class     Standard
Minisign      'm'
OpenSSL       'o'
PGP           'p'
X509          'x'
```

### List

```
Sub-Class     Standard
List          '-'
```

### Non-Typed

```
Sub-Class     Standard
List          '-'
Non-Typed     '_'
```

## Change Log
* v0.0.1, July 2, 2021 -- Initial version with novel ASCII encoding table and type tags of one or two encoding units.
* v0.0.2, July 3, 2021 -- Thanks to Sam Smith for pointing out that this won't round trip without padding so added padding.
* v0.0.3, July 5, 2021 -- More of Sam's feedback incorporated. Rearranged the ASCII encoding table so that lower-case letters/numbers are 0-31 and upper-case letters/numbers are 32-63 so that the 1-bit flags for experimental and extended length are the MSB in each of the first three ASCII characters. This makes visual "reading" of the type tag tell whether it is experimental and if it is an extended length type tag or not. Switched to "standard" types using lower case and experimental types using upper case letters. Added more sub-class and sub-sub-class values for widely used classes.
* v0.0.4, July 6, 2021 -- Added examples of how to work with the bit fields in the first three characters when using a text oriented language such as Javascript.
* v0.0.5, July 7, 2021 -- Renamed this to Cryptographic Data Encoding and pushed to [Github](https://github.com/TrustFrame/cryptographic-data-encoding)