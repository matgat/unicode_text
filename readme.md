## [unicode_text](https://github.com/matgat/unicode_text.git)
[![linux-test](https://github.com/matgat/unicode_text/actions/workflows/linux-test.yml/badge.svg)](https://github.com/matgat/unicode_text/actions/workflows/linux-test.yml)
[![ms-test](https://github.com/matgat/unicode_text/actions/workflows/ms-test.yml/badge.svg)](https://github.com/matgat/unicode_text/actions/workflows/ms-test.yml)
[![License: Unlicense](https://img.shields.io/badge/license-Unlicense-blue.svg)](http://unlicense.org/)

A single header c++ library providing [unicode](https://www.unicode.org)
encoding and decoding facilities.

basic example ([godbolt](https://gcc.godbolt.org/z/6EWcvKx7h))

```cpp
#include <iostream>
#include <string_view>
using namespace std::literals;
#include "unicode_text.hpp" // utxt::*
int main()
{
    const std::u32string_view utf32 = U"🍕🍞🧀"sv;
    const std::string utf8 = utxt::to_utf8(utf32);
    std::cout << utf8;
}
```

### Features
* Not throwing, decoding errors are handled returning/inserting the replacement codepoint `�` (`'\uFFFD'`, `utxt::codepoint::invalid`)
* Needs `c++23` for `std::unreachable`


## Encodings enumeration

| Define    | Description            |
|-----------|------------------------|
|~~`ANSI`~~ | ~~8-bit encodings~~    |
| `UTF8`    | *utf-8*                |
| `UTF16LE` | *utf-16* little-endian |
| `UTF16BE` | *utf-16* big-endian    |
| `UTF32LE` | *utf-32* little-endian |
| `UTF32BE` | *utf-32* big-endian    |

> [!NOTE]
> No support for 8-bit encodings and related *codepages*: it's time to drop them.


## API documentation

---
### Decode bytes to *utf-32* string
Converts bytes to a *utf-32* string

    to_utf32<INENC>(…)
    to_utf32(…)

- *Input*
  - `std::string_view` encoded as `INENC` or `std::u8string_view`
- *Return value*
  - `std::u32string`



[example](https://gcc.godbolt.org/z/co81s7PYx)

```cpp
using enum utxt::Enc;
std::string_view in_bytes = "..."sv;
std::u32string u32str1 = utxt::to_utf32<UTF16LE>(in_bytes);
std::u32string u32str2 = utxt::to_utf32(u8"..."sv);
```

---
### Encode *utf-32* to *utf-8*
Encodes a `char32_t` string or codepoint to a *utf-8* string

    to_utf8(…)

- *Input*
  - `std::u32string_view` or `char32_t`
- *Return value*
  - `std::string` bytes encoded as *utf-8* (avoiding `std::u8string` until better support in *stdlib*)


[example](https://gcc.godbolt.org/z/c1TbvKdrP)

```cpp
std::cout << utxt::to_utf8(U"..."sv)
          << utxt::to_utf8(U'a');
```

---
### Encode *utf-32* to bytes
Encodes a `char32_t` string or codepoint

    encode_as<OUTENC>(…)

- *Inputs*
  - `utxt::Enc OUTENC` output encoding
  - `std::u32string_view` or `char32_t` codepoints to encode
- *Return value*
  - `std::string` encoded bytes as `OUTENC`

[example](https://gcc.godbolt.org/z/5eMWxar1Y)

```cpp
using enum utxt::Enc;
std::string out_bytes = utxt::encode_as<UTF16BE>(U"..."sv);
```

In case `OUTENC` is not known at compile time, there's an
alternate version of this function that chooses the
correct template at runtime:

```cpp
std::string out_bytes = utxt::encode_as(UTF16BE, U"..."sv);
```


---
### Re-encode bytes detecting input encoding
Re-encodes a string of bytes (detecting its encoding) to a given encoding

    encode_as<OUTENC>(…)

- *Inputs*
  - `utxt::Enc OUTENC` output encoding
  - `std::string_view` input bytes of unknown encoding
  - `utxt::flags_t` if specified `flag::SKIP_BOM` output won't contain the byte order mask
- *Return value*
  - `std::string` output bytes encoded as `OUTENC`


[example](https://gcc.godbolt.org/z/jf3Wh9jrK)

```cpp
using enum utxt::Enc;
std::string_view in_bytes = "..."sv;
std::string out_bytes = utxt::encode_as<UTF8>(in_bytes);
static_assert( utxt::encode_as<UTF16BE>(U'🔥') == "\xD8\x3D\xDD\x25"sv );
```

In case `OUTENC` is not known at compile time, there's an
alternate version of this function that chooses the
correct template at runtime:

```cpp
std::string out_bytes = utxt::encode_as(UTF8, in_bytes);
```

Alternate functions that take a buffer and return a `string_view`
are provided to skip the re-encoding in case the input and output
encodings are the same:

[example](https://gcc.godbolt.org/z/3xGW8v9T8)

```cpp
using enum utxt::Enc;
std::string_view in_bytes = "..."sv;
std::string maybe_reencoded_buf;
std::string_view out_bytes = utxt::encode_if_necessary_as<UTF8>(in_bytes, maybe_reencoded_buf);
```

The corresponding runtime version:

```cpp
std::string_view out_bytes = utxt::encode_if_necessary_as(UTF8, in_bytes, maybe_reencoded_buf);
```


---
### Re-encode bytes
Re-encodes a string of bytes from one encoding to another

    reencode<INENC,OUTENC>(…)

- *Inputs*
  - `utxt::Enc INENC` input encoding
  - `utxt::Enc OUTENC` output encoding
  - `std::string_view` input bytes encoded as `INENC`
- *Return value*
  - `std::string` output bytes encoded as `OUTENC`

[example](https://gcc.godbolt.org/z/rrsj6cnf4)

```cpp
using enum utxt::Enc;
std::string_view in_bytes = "..."sv;
std::string out_bytes = utxt::reencode<UTF8, UTF16LE>(in_bytes);
```

An alternate function that takes a buffer and returns a `string_view`
is provided to skip the re-encoding in case `INENC==OUTENC`:

[example](https://gcc.godbolt.org/z/Kno69nKE5)

```cpp
std::string_view in_bytes = "..."sv;
std::string maybe_reencoded_buf;
std::string_view out_bytes = utxt::reencode_if_necessary<INENC,OUTENC>(in_bytes, maybe_reencoded_buf);
```


---
## Low level facilities

---
### `bytes_buffer_t` class
A class that represents a byte stream interpreted with a given encoding.

[example](https://gcc.godbolt.org/z/xM76nYqxW)

```cpp
using enum utxt::Enc;
utxt::bytes_buffer_t<UTF8> bytes_buf("..."sv);
while( bytes_buf.has_codepoint() )
   {
    const char32_t cp = bytes_buf.extract_codepoint();
   }
if( bytes_buf.has_bytes() )
   {// Truncated codepoint!
   }
```

---
### Encoding Detection
Detects the encoding of raw bytes,
it just detects the byte order mask, no euristic analysis of bytes

    detect_encoding_of(…)

- *Input*
  - `std::string_view` raw bytes
- *Return value*
  - `struct{ Enc enc; std::uint8_t bom_size; }`

[example](https://gcc.godbolt.org/z/5z8q37f3f)

```cpp
std::string to_utf8(const std::string_view bytes)
{
    const auto [bytes_enc, bom_size] = utxt::detect_encoding_of(bytes);
    switch(bytes_enc)
       {using enum utxt::Enc;
        case UTF16LE: return utxt::reencode<UTF16LE,UTF8>(bytes);
        case UTF16BE: return utxt::reencode<UTF16BE,UTF8>(bytes);
        case UTF32LE: return utxt::reencode<UTF32LE,UTF8>(bytes);
        case UTF32BE: return utxt::reencode<UTF32BE,UTF8>(bytes);
        default:      return std::string{bytes};
       }
}
```

---
### Decoding a single codepoint
Extracts a codepoint from a string of raw bytes interpreted with encoding `Enc`,
updating the current position that points to the data.

    extract_codepoint<Enc>(…)

- *Inputs*
  - `std::string_view` raw bytes encoded as `Enc`
  - `std::size_t&` current position
- *Preconditions*
  - Assumes enough remaining bytes to extract the codepoint, undefined behavior otherwise
- *Return value*
  - `char32_t` extracted codepoint, `codepoint::invalid` in case of decoding errors

[example](https://gcc.godbolt.org/z/4a3WGee5c)

```cpp
using enum utxt::Enc;
std::string_view bytes = "\xD8\x3D\xDD\x25"sv;
std::size_t pos = 0;
assert( (pos+sizeof(char32_t)) <= bytes.size() );
const char32_t cp = utxt::extract_codepoint<UTF16BE>(bytes, pos);
assert( cp == U'🔥' );
```

---
### Encoding a single codepoint
Appends a codepoint to a given string of bytes using encoding `Enc`

    append_codepoint<Enc>(…)

- *Input*
  - `char32_t` codepoint to encode
- *Output*
  - `std::string&` destination bytes encoded as `Enc`

[example](https://gcc.godbolt.org/z/YW1h643nW)

```cpp
std::string bytes = "a ";
char32_t codepoint = U'🔥';
utxt::append_codepoint<UTF8>(codepoint, bytes);
assert( bytes == "a \xF0\x9F\x94\xA5"sv);
```


## Build
Build with at least `-std=c++23` (`/std:c++23` in case of *msvc*),
the culprit is `std::unreachable`.

### Testing
Run unit tests directly in [godbolt](https://gcc.godbolt.org/z/1zzTh7Yq5)
or:

```sh
$ git clone https://github.com/matgat/unicode_text.git
$ cd unicode_text
$ curl -O https://raw.githubusercontent.com/boost-ext/ut/master/include/boost/ut.hpp
$ g++ -std=c++23 -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion -o test test.cpp && ./test
```

> [!NOTE]
> On windows:
> ```bat
> $ cl /std:c++latest /permissive- /utf-8 /W4 /WX /EHsc test.cpp
> ```
