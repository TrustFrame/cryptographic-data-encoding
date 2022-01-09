# Cryptographic Data Encoding (CDE)

[![hackmd-github-sync-badge](https://hackmd.io/y1IYN_9RRkCxL6lFUpkhbQ/badge)](https://hackmd.io/y1IYN_9RRkCxL6lFUpkhbQ)

Version: v0.0.12 Pre-Draft  
Date: January 8, 2022  
License: CC BY 4.0  
Author: Dave Huseby <dave@cryptid.tech>  
Reference Implementation: https://github.com/cryptidtech/cde  
Copyright: (c) CryptID Technologies, Inc. 2021--2022  

This cryptographic data encoding (CDE) proposal is for encoding of cryptographic data into a self-describing format using a method that simplifies both ASCII (base 64) and binary (base 2) serialization/deserialization. It has some additional novel properties (e.g. easy visual inspection of ASCII, simple parsing in binary) that makes it especially useful for encoding cryptographic data.

The primary, and most important, goal is that the ASCII representation of every construct begins with easy-to-learn characters associated with the different classes (e.g. cryptographic keys) and sub-classes (e.g. Ed25519 keys) to aid in debugging and visual inspection. Since most cryptographic constructs have three levels of classification (i.e. key -> algorithm -> variant/bit length) the type tags for CDE have three layers of classification: class, sub-class, and sub-sub-class. The secondary goal is that the binary representation of every construct has an easily parsed type tag to simplify serialization and deserializtion.

To understand this spec you have to think in terms of "encoding units". An encoding unit is 24-bits of data organized as either a group of four 6-bit base 64 words (e.g. `000000 111111 000000 111111`) or a group of three 8-bit words (e.g. `00000000 11111111 00000000`). The group of four 6-bit words is used for encoding the data using ASCII characters in the same way that Base64 encoding works. Each of the 6-bit words maps to an ASCII character in a table of 64 characters that is similar to the Base64 table but is unique to CDE. The result is that each encoding unit is represented by four ASCII characters in the text encoding. The group of three 8-bit words is used for encoding the data when using binary. Either way, each encoding unit is 24-bits in length making it easy to encode any arbitrary data in either form depending on the system.

The diagram below illustrates how each 24-bits encoding unit is split into either four, 6-bit words for the text encoding or three, 8-bit words for the binary encoding.

```
         /--1--/--2--/--3--/--4--/  four, 6-bit words for text encoding
24 bits: xxxxxxxxxxxxxxxxxxxxxxxx
         \---1---\---2---\---3---\  three, 8-bit words for binary encoding
```

## Streaming

CDE is designed to facilitate streaming of one or more cryptographic constructs in either the text or binary encoding. Each construct is self-describing with a type tag that contains both the type and length, in bytes, of the encoded cryptographic construct. The diagram below illustrates the structure of a CDE stream.

```
/-- type tag --/-- encoded data --/-- type tag --/-- encoded data --/
```

To support encoding complex cryptographic constructs, CDE designates a few special type tags for lists of constructs as well a construct for un-typed arbitrary data. CDE lists are recursive and can contain other lists allowing for any complex cryptographic construct to be encoded.

### Parsing

When parsing the text representation of a CDE encoded cryptographic construct a parser must only accept characters in the CDE encoding alphabet and ignore all other characters, including whitespace. This allows for a text encoded CDE construct to be arbitrarily formatted for easier reading without affecting the parsing of the construct. This is specifically to allow for line wrapping with carriage returns and new line characters as well as other common line wrapping techniques such as adding a trailing '\\' before a newline or indenting with tabs and/or spaces.

Software that consumes CDE encoded constructs must be tolerant of, and ignore any encoded constructs with types that it doesn't support. If the software encounters a class, sub-class, or even a sub-sub-class that it does not know how to work with, it must parse the length and skip over the construct entirely as if it wasn't there.

## Type Tags

Every CDE object begins with a type tag that is at least one encoding unit and optionally may include upt to two additional encoding units for the length field to support larger constructs. The alignment of the fields is carefully chosen to facilitate easy serialization/deserialization in both its text and binary encodings. The diagram below illustrates the interpretation of the type tag when encoded into 6-bit words for the text representation.

```
       encoding unit 1          optional encoding unit 2     optional encoding unit 3
 /--------------------------/ /--------------------------/ /--------------------------/
/--1--//--2--//--3--//--4--/ /--5--//--6--//--7--//--8--/ /--9--//--A--//--B--//--C--/
faaaaa fbbbbb ccccxx xxxxxx  xxxxxx xxxxxx xxxxxx xxxxxx  xxxxxx xxxxxx xxxxxx xxxxxx
||   | ||   | |  ||                                                                 |
||   | ||   | |  |+-----------------------------------------------------------------+
||   | ||   | |  |        |.. varuint encoded length  
||   | ||   | +--+........... sub-sub-class
||   | |+---+................ sub-class
||   | +..................... exp. sub-class flag
|+---+....................... class
+............................ exp. class flag
```

This second diagram illustrates the interpretation of the type tag when encoded into 8-bit words for the binary representation.

```
       encoding unit 1         optional encoding unit 2    optional encoding unit 3
 /-------------------------/ /-------------------------/ /-------------------------/
/---0---//---1---//---2---/ /---3---//---4---//---5---/ /---6---//---7---//---8---/
faaaaafb bbbbcccc xxxxxxxx  xxxxxxxx xxxxxxxx xxxxxxxx  xxxxxxxx xxxxxxxx xxxxxxxx
||   |||    ||  | |                                                              |
||   |||    ||  | +--------------------------------------------------------------+
||   |||    ||  |        |.. varuint encoded length
||   |||    |+--+........... sub-sub-class
||   ||+----+............... sub-class
||   |+..................... exp. sub-class flag
|+---+...................... class
+........................... exp. class flag
```

The first two 6-bit words are dedicated to the class and sub-class of the construct. Each of them has an experimental class/sub-class flag as the most significant bit. Normally, detecting single bit changes in the text encoding is very difficult and complicated in text. However CDE uses a novel organization of the ASCII characters in the encoding table such that when the experimental flag is 0 the text encoding is a lower case letter or a number 0-4 and when the experimental flag is 1, the text encoding is an uppercase letter or a number 5-9. The important thing to remember when processing text encodings is that if you don't care about the experimental status, you can lower-case any letters and mod 5 any numbers to lower-case everything before processing.

The sub-sub-class value takes up the most significant 4 bits in the third 6-bit word. This design is a compromise to meet the needs of both the text representation and the binary representation. By assigning the class and sub-class fields to the first two 6-bit words, every cryptographic construct of the same class and sub-class will begin with the same ASCII characters and the case of those characters signifies the experimental status of the class and sub-class. This makes it easy for humans to learn these codes and to "read" them visually and know what type of cryptographic construct is encoded.

The sub-sub-class and the two 6-bit class and sub-class fields are designed to all fit into two 8-bit words of the binary representation. So, along with easy visual inspection when in the text representation, we also get a simple bitfield data structure for processing the binary representation.

The length value is encoded using a mechanism called "varuint". It encodes unsigned integers in little endian order. The most-significant bit of each 8-bit word of the length is a flag indicating that it is not the last 8-bit word of length data. This means that each 8-bit word carries 7 bits of length data and with up to two optional encoding units, CDE supports varuints of up to 7, 8-bit words in length giving a length value of up to 49-bits in size. If a cryptographic construct is less than 128 bytes in length, then the varuint length value will only take up a single 8-bit word and the CDE tag will only be 3, 8-bit words in length and only take up one encoding unit. It is important to note that most cryptographic constructs such as keys, nonces, etc are less than 128 bytes in length. If a construct is larger than 128 bytes but less than 256 MB in length, then the varuint length value will be 4, 8-bit words and the tag will take up two encoding units. If the construct is larger than 256 MB and less than 512 TB then the varuint length value will be 7, 8-bit words and the tag will take up three encoding units. The following is an illustration of how varuints are encoded.

```
1        => 00000001
127      => 01111111
128      => 10000000 00000001
255      => 11111111 00000001
16384    => 10000000 10000000 00000001
```

One of the hardest things to do with text encodings is bit manipulation using a programming language that doesn't support it natively like C or Rust does. Thankfully, even in this situation, a little intuition and math makes it possible to work with the different bits in the CDE type tag, specifically in the third 6-bit word. The following code illustrates how to use Javascript to extract the sub-sub-class bits as well as the two size bits from the third 6-bit word so that they can be combined with the fourth 6-bit word to calculate the length of the data in bytes.

```javascript=
// assume the third letter in a type tag is 'm' and the fourth letter is 'A'
// 'm' is index 12 (001100 in binary) and 'A' is index 32 (100000 in binary)
// when decoded from the character the index value is the result
let v1 = 12; // assuming cde_decode('m') gives 12
let v2 = 32; // assuming cde_decode('A') gives 32

// to get the sub-sub-class, we simply divide by 4 and round down
let ssc = Math.floor(v1 / 4); // gives 3, or 0011 in binary

// to get the length in bytes, we use modulus, multiplication and addition
let two_msb = v1 % 3; // isolates the two lowest bits of the third 6-bit word
let length = (two_msb << 6) + v2; // gives 32, or 00100000 in binary
```

The 1-bit flags for speciying if a class or sub-class is experimental is trivial to process. Because of the novel layout of the CDE character encoding table, simply looking at the letters to see if they are upper-case reveals the value of the 1-bit flags. If you want to write code, you must decode the letter to an integer and then test if the value is >= 32. The following code demonstrates this.

```javascript=
// assume the first letter for the class is 'K'. 'K' decodes to the value 42
// which is 101010 in binary.
let cls = 42; // assuming cde_decode('K') gives 42

// it is simple to determine if the code is experimental
if (cls >= 32) {
  // experimental
} else {
  // standard
}
```

### Length

The length of an encoded cryptographic construct is counted in the number of 8-bit bytes. Because the length of a cryptographic construct might not be a multiple of three bytes, there are potentially one or two padding bytes necessary to encode from the binary to the text representation. CDE does not define, nor does it preserve padding bytes in the text representation therefore the length value must be the number of bytes so that software that processes CDE text encoded constructs knows how many pad bytes are implied.

Given the length of the construct in bytes, the number of text characters to read after the type tag is calculated with the following code. This correctly accounts for the missing padding characters and will not read too many characters.

```c=
size_t length_in_text = ((length_in_bytes << 3) + 5) / 6;
```

If your programming language of choice does not support the bit shift operator `<<` you may just multiple the length value by 8.

```javascript=
let length_in_text = Math.floor(((length_in_bytes * 8) + 5) / 6);
```

Because CDE uses a varuint encoded length up to 7, 8-bit words in length, the maximum size for a cryptographic construct encoded in CDE format is 562,949,953,421,311 bytes, or exactly 512 TB.

## Lists and Non-Typed

CDE is specifically designed to support encoding arbitrarily complex cryptographic data structures/constructs. The simplest way to achieve that is to support recursive lists a la Lisp. CDE achieves this using the `-` (hyphen) character (e.g. 0b011111 6-bit word) as the class and/or sub-class to indicate that the encoded construct is a list of other constructs.

Sometimes it is useful to have a list of constructs of mixed types. CDE supports this by using the `_` (underscore) (e.g. 0b111111 6-bit word) to indicate that the class of the constructs in the list is undefined and can be anything. List of this kind must contain CDE encoded constructs with their own tags and data so that the class, sub-class, and sub-sub class of each list member is preserved.

There are three valid combinations of the `_` and `-` characters. A list of constructs of any type is defined using the combination `_-` to say that the class is undefined and the sub-class is a list. A list of lists is specified using the combination `--`. A construct that is binary large object (BLOB) of unspecified class and sub-class is specified using the combination `__`. This is useful for encoding arbitrary data. The combination of hyphen followed by underscore `-_` is not valid and is reserved for future use. If encountered, a parser shall read the length value and skip over the construct and continue processing.

### Typed List

Typed lists are specified by combining a class code other than `-` or `_` with `-` as the sub-class code. The type tag for lists is interpreted in the same way as described above except that the length field does not specify the number of 8-bit bytes but instead specifies the number constructs in the list. List constructs can also take full advantage of the varuint encoding giving them a maximum capacity of 512 trillion. The diagram below illustrates how a text-encoded typed list tag is interpreted.

```
       encoding unit 1          optional encoding unit 2     optional encoding unit 3
 /--------------------------/ /--------------------------/ /--------------------------/
/--1--//--2--//--3--//--4--/ /--5--//--6--//--7--//--8--/ /--9--//--A--//--B--//--C--/
faaaaa 011111 ccccxx xxxxxx  xxxxxx xxxxxx xxxxxx xxxxxx  xxxxxx xxxxxx xxxxxx xxxxxx
||   | |    | |  ||                                                                 |
||   | |    | |  |+-----------------------------------------------------------------+
||   | |    | |  |        |.. varuint encoded length  
||   | |    | +--+........... app-specific value
||   | +----+................ '-' list sub-class
|+---+....................... class
+............................ exp. class flag
```

### Non-Typed List

Non-typed lists are specified by combining the `_` class code with the `-` sub- class code. These are useful for constructing lists of constructs of different types. Just like typed lists described above, the length field in the type tag specifies the number of items in the list. Technically, using `-` as the class and `_` as the sub-class also defines a list of non-typed constructs but this specification choosed to define that as an invalid type tag and reserve it for future use. Standard compliant software should ignore constructs with `-_` class and sub-class just like any other constructs the software does not support. The diagram below illustrates how a text-encoded non-typed list tag is interpreted.

```
       encoding unit 1          optional encoding unit 2     optional encoding unit 3
 /--------------------------/ /--------------------------/ /--------------------------/
/--1--//--2--//--3--//--4--/ /--5--//--6--//--7--//--8--/ /--9--//--A--//--B--//--C--/
111111 011111 ccccxx xxxxxx  xxxxxx xxxxxx xxxxxx xxxxxx  xxxxxx xxxxxx xxxxxx xxxxxx
|    | |    | |  ||                                                                 |
|    | |    | |  |+-----------------------------------------------------------------+
|    | |    | |  |        |.. varuint encoded length  
|    | |    | +--+........... app-specific value
|    | +----+................ '-' list sub-class
+----+....................... '_' undefined class
```

### List of Lists

A list of lists is really just a typed list with list `-` as the class and list `-` as the sub-class. This is not a special case but warrants highlighting it as a valid case. Just like the typed list described above, the length field encodes the number of lists in the list. The diagram below illustrates how a text-encoded list of lists tag is interpreted.

```
       encoding unit 1          optional encoding unit 2     optional encoding unit 3
 /--------------------------/ /--------------------------/ /--------------------------/
/--1--//--2--//--3--//--4--/ /--5--//--6--//--7--//--8--/ /--9--//--A--//--B--//--C--/
011111 011111 ccccxx xxxxxx  xxxxxx xxxxxx xxxxxx xxxxxx  xxxxxx xxxxxx xxxxxx xxxxxx
|    | |    | |  ||                                                                 |
|    | |    | |  |+-----------------------------------------------------------------+
|    | |    | |  |        |.. varuint encoded length  
|    | |    | +--+........... app-specific value
|    | +----+................ '-' list sub-class
+----+....................... '-' list class
```

## Non-Typed Data Construct

By using the non-typed code `_` for both the class and sub-class, CDE supports encoding arbitrary data as a single non-typed construct. The diagram below illustrates how a text-encoded non-typed construct tag is interpreted.

```
       encoding unit 1          optional encoding unit 2     optional encoding unit 3
 /--------------------------/ /--------------------------/ /--------------------------/
/--1--//--2--//--3--//--4--/ /--5--//--6--//--7--//--8--/ /--9--//--A--//--B--//--C--/
111111 111111 ccccxx xxxxxx  xxxxxx xxxxxx xxxxxx xxxxxx  xxxxxx xxxxxx xxxxxx xxxxxx
|    | |    | |  ||                                                                 |
|    | |    | |  |+-----------------------------------------------------------------+
|    | |    | |  |        |.. varuint encoded length  
|    | |    | +--+........... app-specific value
|    | +----+................ '_' undefined sub-class
+----+....................... '_' undefined class
```

## Extensibility

Normally, an encoding like this uses a standard Base64 alphabet for encoding the 6-bit words as ASCII characters. But since we have a hard requirment for extensibility, this standard uses a slightly modified Base64 alphabet derived from the URL-safe variant. The CDE variant places the lower-case letters, the first five numeric characters, and hypen (`-`) in the first 32 index places with the upper-case, the last five numbers followed by underscore (`_`) in the last 32 index places. This new arrangement makes the most significant bit a flag that can be tested easily in binary and determined visually in text (i.e. upper case vs lower case letters, lower number vs upper numbers).

Since the class and sub-class text encoding is designed to be the same for all constructs of a given class (i.e. all cryptographic keys begin with `k`) and sub-class (i.e. all Ed25519 keys begin with `ke`) by re-arranging the encoding alphabet for CDE, it supports experimental extensions by using upper case versions of the same letters. For instance, 4,096-bit RSA keys are encoded such that their text representation always begins with `kr` (i.e. `k` for key, `r` for RSA) but experimental 16,384-bit RSA keys always begin with `KR`. The former says the construct is a standard RSA key (e.g. 4,096-bit RSA key) and the latter says the construct is an experimental RSA key (e.g. 16,384-bit RSA key).

Below is a table comparing the URL-Safe Base64 (B64) alphabet and the CDE alphabet. Note how the lowercase letters in the CDE alphabet begin at index 0 instead of index 26 as they do in Base64. This novel arrangement makes it trivial to identify experimental classes and sub-classes in both text encoded and binary encoded representations.

```
Index  Binary  B64  CDE      Index  Binary  B64  CDE
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
26     011010   a    0       58     111010   6    5
27     011011   b    1       59     111011   7    6
28     011100   c    2       60     111100   8    7
29     011101   d    3       61     111101   9    8
30     011110   e    4       62     111110   -    9
31     011111   f    -       63     111111   _    _
```

## Proposed Classes

The most common set of cryptographic constructs that have wide adoption and should have their own classes are:

- aead (i.e. authentic encryption with associated data)
- claim (i.e. cryptographic proofs)
- digest
- encryption
- [strobe operations](https://strobe.sourceforge.io/specs/) (useful for encoding strobe-based protocols)
- hmac
- identifier (i.e. emails, DIDs, etc)
- key
- nonce
- policy (i.e. smart contract, script, etc)
- signature
- timestamp (i.e. unix epoch, bitcoin height, etc)

Because none of these names share a common first letter it is trivial to assign a unique character to each--strobe is assigned `f` because in the strobe specification the Keccak sponge function that the protocol framework is based on is called `f`. Each class will have both an upper case letter and a lower case letter assigned to it for "standard" and "experimental" sub-classes respectively.

### Application Specific Extensions

CDE is designed to support constructing software systems that may want to define their own classes and sub-classes for common constructs instead of explicitly constructing them out of recursive lists. For instance, cryptographic provenance logs are made up of cryptographic events. In software that uses CDE to store logs of events, provenance log constructs can use `L` for log--upper-case for experimental--and then give the event construct a sub-class `E` for event, also upper-case for experimental. Then all provenance log event constructs encoded in text begin with `LE`. This is an alternative to defining the log event constructs explicitly out of recursive lists, obfuscating the type.

In many ways, developers start off by explicitly defining their constructs as recursive list constructs during the early prototyping phase. Then as the common constructs become known, they are assigned their own experimental class and sub-class to make their encodings more pedantic and easier to read and debug. Then as the software gains wide adoption and the constructs become widely used, the experimental class code is promoted to a lower-case, non-experimental when it becomes standardized.

The following table lists the characters assigned to each class of widely used cryptographic constructs.

```
Class      Standard      Experimental
AEAD       'a'           'A'
Claim      'c'           'C'
Digest     'd'           'D'
Encryption 'e'           'E'
Strobe     'f'           'F'
HMAC       'h'           'H'
Identifier 'i'           'I'
Key        'k'           'K'
Nonce      'n'           'N'
Policy     'p'           'P'
Signature  's'           'S'
Timestamp  't'           'T'
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

### Claim
```
Sub-Class     Standard
Oberon        'o'
```

### Digest

```
Sub-Class     Standard
Blake2        'b'
MD            'm'
SHA1          's'
SHA2          'h'
SHA3          'a'
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

### Encryption

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

### Strobe

```
Sub-Class     Standard
AD            'a'
CLR           'c'
ENC           'e'
KEY           'k'
MAC           'm'
PRF           'p'
Ratchet       'r'
```

#### CLR

```
Sub-Sub-Class   Value
Send            0
Recv            1
```

#### ENC

```
Sub-Sub-Class   Value
Send            0
Recv            1
```

#### MAC

```
Sub-Sub-Class   Value
Send            0
Recv            1
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
X25519        'x'
RSA           'r'
Bls12381      'b'
K256          'k'
P256          'p'
Chacha20      'c'
AES           'a'
```

#### Ed25519

```
Sub-Sub-Class   Value
Public Key      0
Secret Key      1
```

#### X25519

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

#### Bls12381

```
Sub-Sub-Class   Value
Public Key      0
Secret Key      1
```

#### K256

```
Sub-Sub-Class   Value
Public Key      0
Secret Key      1
```

#### P256

```
Sub-Sub-Class   Value
Public Key      0
Secret Key      1
```

#### AES

```
Sub-Sub-Class   Value
128-bit         0
256-bit         1
```

### Nonce

```
Sub-Class     Standard
```

### Policy

```
Sub-Class     Standard
Bitcoin       'b'
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

### Timestamp

```
Sub-Class     Standard
Unix Epoch    'u'
Iso8601       'i'
Bitcoin Hght  'b'
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
* v0.0.6, July 7, 2021 -- Cleaned up formatting and renaming.
* v0.0.7, July 9, 2021 -- Moved list type code '-' to a non-experimental index.
* v0.0.8, July 9, 2021 -- Cleaned up the working and examples from moving '-'.
* v0.0.12, January 8, 2022 -- Reworked the length to be unsigned varint, added classes/sub-classes