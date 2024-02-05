## [unicode_utils](https://github.com/matgat/unicode_utils.git)
[![linux-test](https://github.com/matgat/unicode_utils/actions/workflows/linux-test.yml/badge.svg)](https://github.com/matgat/unicode_utils/actions/workflows/linux-test.yml)
[![ms-test](https://github.com/matgat/unicode_utils/actions/workflows/ms-test.yml/badge.svg)](https://github.com/matgat/unicode_utils/actions/workflows/ms-test.yml)

A single header c++ library providing a collection of c++ constexpr function
templates for [unicode](https://www.unicode.org) encoding and decoding.

[basic example](https://gcc.godbolt.org)

```cpp
#include "unicode_text.hpp" // utxt::*
static_assert( ... );
int main()
{
}
```


## Features
* Decoding errors are handled returning/inserting the replacement codepoint `�` (`'\uFFFD'`, `utxt::codepoint::invalid`)


## Encodings enumeration

| Define    | Description            |
|-----------|------------------------|
|~~`ANSI`~~ | ~~8-bit encodings~~    |
| `UTF8`    | *utf-8*                |
| `UTF16LE` | *utf-16* little-endian |
| `UTF16BE` | *utf-16* big-endian    |
| `UTF32LE` | *utf-32* little-endian |
| `UTF32BE` | *utf-32* big-endian    |

No support for 8-bit encodings, it's time to drop them.


## API documentation


### Decode bytes to *utf-32* string
- Function
  - `to_utf32<INENC>()` or `to_utf32()`
- Inputs
  - `std::string_view` encoded as `INENC` or `std::u8string_view`
- Return value
  - `std::u32string`
- Description
  - Converts bytes a *utf-32* string

[example](https://gcc.godbolt.org)

```cpp
std::string_view utf16le_bytes = "..."sv;
std::u32string u32str = utxt::to_utf32<utxt::Enc::UTF16LE>(in_bytes);

std::u32string u32str = utxt::to_utf32(u8"..."sv);
```


### Encode *utf-32* to *utf-8* bytes
- Function
  - `to_utf8()`
- Inputs
  - `std::u32string_view` or `char32_t`
- Return value
  - `std::string` bytes encoded as *utf-8*
- Description
  - Encodes a `char32_t` string or codepoint to a *utf-8* string

[example](https://gcc.godbolt.org)

```cpp
std::cout << utxt::to_utf8(U"..."sv)
          << utxt::to_utf8(U'a');
```


### Encode *utf-32* to bytes

- Function
  - `encode_as<OUTENC>()`
  - `encode_as()` (if `OUTENC` is not known at compile time)
- Inputs
  - `utxt::Enc OUTENC` output encoding
  - `std::u32string_view` or `char32_t` codepoints to encode
- Return value
  - `std::string` encoded bytes
- Description
  - Encodes a `char32_t` string or codepoint
    The run time version contains a `switch` that instantiates the correct template

[example](https://gcc.godbolt.org)

```cpp
std::string out_bytes = utxt::encode_as<utxt::Enc::UTF16BE>(U"..."sv);
std::string out_bytes = utxt::encode_as(utxt::Enc::UTF16BE, U"..."sv);
```


### Re-encode bytes
- Functions
  - `reencode<INENC,OUTENC>()`
- Inputs
  - `utxt::Enc INENC` input encoding
  - `utxt::Enc OUTENC` output encoding
  - `std::string_view` input bytes encoded as `INENC`
- Return value
  - `std::string` output bytes encoded encoded as `OUTENC`
- Description
  - Re-encodes a string of bytes from one encoding to another

[example](https://gcc.godbolt.org)

```cpp
std::string_view in_bytes = "..."sv;
std::string out_bytes = utxt::reencode<utxt::Enc::UTF8, utxt::Enc::UTF16LE>(in_bytes);
```

An alternate function that takes a buffer and returns a `string_view`
is provided to skip the re-encoding in case `INENC==OUTENC`:

[example](https://gcc.godbolt.org)

```cpp
std::string_view in_bytes = "..."sv;
std::string maybe_reencoded_buf;
std::string_view out_bytes = utxt::reencode_if_necessary<INENC,OUTENC>(in_bytes, maybe_reencoded_buf);
```


### Re-encode bytes detecting input encoding
- Function
  - `encode_as<OUTENC>()`
  - `encode_as()` (if `OUTENC` is not known at compile time)
- Inputs
  - `utxt::Enc OUTENC` output encoding
  - `std::string_view` input bytes of unknown encoding
  - `utxt::flags_t` if specified `flag::SKIP_BOM` output won't contain the byte order mask
- Return value
  - `std::string`
- Description
  - Re-encodes a string of bytes (detecting its encoding) to a given encoding
    The run time version contains a `switch` that instantiates the correct template

[example](https://gcc.godbolt.org)

```cpp
std::string_view in_bytes = "..."sv;
std::string out_bytes = utxt::encode_as<utxt::Enc::UTF8>(in_bytes);
std::string out_bytes = utxt::encode_as(utxt::Enc::UTF8, in_bytes);
```

Alternate functions that take a buffer and return a `string_view`
are provided to skip the re-encoding in case the input and output
encodings are the same:

[example](https://gcc.godbolt.org)

```cpp
std::string_view in_bytes = "..."sv;
std::string maybe_reencoded_buf;
std::string_view out_bytes = utxt::encode_if_necessary_as<utxt::Enc::UTF8>(in_bytes, maybe_reencoded_buf);
std::string_view out_bytes = utxt::encode_if_necessary_as(utxt::Enc::UTF8>, in_bytes, maybe_reencoded_buf);
```



## Low level functions

### `bytes_buffer_t` class

A class that represents a byte stream interpreted with a given encoding.

[example](https://gcc.godbolt.org)

```cpp
utxt::bytes_buffer_t<utxt::Enc::UTF8> bytes_buf("..."sv);
while( bytes_buf.has_codepoint() )
   {
    const char32_t cp = bytes_buf.extract_codepoint();
   }
if( bytes_buf.has_bytes() )
   {// Truncated codepoint!
   }
```


### Encoding Detection

- Function
  - `detect_encoding_of()`
- Inputs
  - `std::string_view` raw bytes
- Return value
  - `struct{ Enc enc; std::uint8_t bom_size; }`
- Description
  - Detects the encoding of raw bytes - it just detects the byte order mask, no data euristics

[example](https://gcc.godbolt.org)

```cpp
const std::string_view bytes = "..."sv;
const auto [bytes_enc, bom_size] = utxt::detect_encoding_of(bytes);
switch(bytes_enc)
   {using enum utxt::Enc;
    case UTF16LE: return utxt::reencode<UTF8,UTF16LE>(bytes);
    case UTF16BE: return utxt::reencode<UTF8,UTF16BE>(bytes);
    case UTF32LE: return utxt::reencode<UTF8,UTF32LE>(bytes);
    case UTF32BE: return utxt::reencode<UTF8,UTF32BE>(bytes);
    default:      return std::string{bytes};
   }
```


### Decoding a single codepoint
- Function
  - `extract_codepoint<Enc>()`
- Inputs
  - `std::string_view` raw bytes
  - `std::size_t&` current position
- Preconditions
  - Assumes enough remaining bytes to extract the codepoint, undefined behavior otherwise
- Return value
  - `char32_t` extracted codepoint, `codepoint::invalid` in case of decoding errors
- Description
  - Extracts a codepoint from a string of raw bytes interpreted with encoding `Enc`,
    updating the current position that points to the data.

[example](https://gcc.godbolt.org)

```cpp
std::string_view bytes = "...."sv;
std::size_t pos = 0;
assert( (pos+sizeof(char32_t)) < bytes.size() );
const char32_t cp = utxt::extract_codepoint<utxt::Enc::UTF32LE>(bytes, pos);
```


### Encoding a single codepoint
- Function
  - `append_codepoint<Enc>()`
- Inputs
  - `char32_t` codepoint to encode
  - `std::string&` destination bytes
- Return value
  - `void`
- Description
  - Appends a codepoint to a given string of bytes using encoding `Enc`

[example](https://gcc.godbolt.org)

```cpp
std::string bytes;
char32_t codepoint = U'a';
utxt::Enc::UTF8append_codepoint<utxt::Enc::UTF8Enc::UTF8>(codepoint, bytes);
```
