# Rust library for OEM Code pages

![CI (master)](<https://github.com/tats-u/rust-oem-cp/workflows/CI%20(master)/badge.svg>)
![CI (Release)](<https://github.com/tats-u/rust-oem-cp/workflows/CI%20(Release)/badge.svg>)
[![oem_cp at crates.io](https://img.shields.io/crates/v/oem_cp.svg)](https://crates.io/crates/oem_cp)
[![oem_cp at docs.rs](https://docs.rs/oem_cp/badge.svg)](https://docs.rs/oem_cp)

This library handles many SBCS (single byte character sets) that are used as OEM code pages. OEM code pages are used for encoding file names in ZIP archives and characters in the terminal in Windows today.

# Supported code pages

| Code Page | Note                                             |
| --------- | ------------------------------------------------ |
| 437       | OEM United States                                |
| 737       | OEM Greek (formerly 437G); Greek (DOS)           |
| 775       | OEM Baltic; Baltic (DOS)                         |
| 850       | OEM Multilingual Latin 1; Western European (DOS) |
| 852       | OEM Latin 2; Central European (DOS)              |
| 855       | OEM Cyrillic (primarily Russian)                 |
| 857       | OEM Turkish; Turkish (DOS)                       |
| 860       | OEM Portuguese; Portuguese (DOS)                 |
| 861       | OEM Icelandic; Icelandic (DOS)                   |
| 862       | OEM Hebrew; Hebrew (DOS)                         |
| 863       | OEM French Canadian; French Canadian (DOS)       |
| 864       | OEM Arabic; Arabic (864)                         |
| 865       | OEM Nordic; Nordic (DOS)                         |
| 866       | OEM Russian; Cyrillic (DOS)                      |
| 869       | OEM Modern Greek; Greek, Modern (DOS)            |
| 874       | ANSI/OEM Thai (ISO 8859-11); Thai (Windows)      |

Notes are quoted from https://docs.microsoft.com/en-us/windows/win32/intl/code-page-identifiers

# How to use

Add `oem_cp` to the dependencies in `Cargo.toml` in your projects.

```toml
[dependencies]
# *snip*
oem_cp = "0.1.1"
# *snip*
```

# Examples

## Use specific code pages

### Encoding Unicode string to SBCS bytes

```rust
use oem_cp::{encoding_string_checked, encoding_string_lossy};
use oem_cp::code_table::{ENCODING_TABLE_CP437, ENCODING_TABLE_CP737};

assert_eq!(encode_string_checked("π≈22/7", &*ENCODING_TABLE_CP437), Some(vec![0xE3, 0xF7, 0x32, 0x32, 0x2F, 0x37]));
// Archimedes in Greek
assert_eq!(encode_string_checked("Αρχιμήδης", &*ENCODING_TABLE_CP737), Some(vec![0x80, 0xA8, 0xAE, 0xA0, 0xA3, 0xE3, 0x9B, 0x9E, 0xAA]));
// ¾ (U+00BE) is not included in CP437
assert_eq!(encoding_string_checked("½+¼=¾", &*ENCODING_TABLE_CP437), None);
// Unknown characters can be replaced with ? (0x3F)
assert_eq!(encoding_string_lossy("½+¼=¾", &*ENCODING_TABLE_CP437), vec![0xAB, 0x2B, 0xAC, 0x3D, 0x3F]);
```

### Decoding SBCS bytes to Unicode string

```rust
use oem_cp::{decode_string_complete_table, decode_string_incomplete_table_checked, decode_string_incomplete_table_lossy};
use oem_cp::code_table::{DECODING_TABLE_CP437, DECODING_TABLE_CP874};

assert_eq!(&decode_string_complete_table(vec![0xFB, 0xAC, 0x3D, 0xAB], &DECODING_TABLE_CP437), "√¼=½");

// For encoding that has some undefined code points, you must use decode_string_incomplete_table_{checked,lossy} instead of decode_string_complete_table

// means shrimp in Thai (U+E49 => 0xE9)
assert_eq!(decode_string_incomplete_table_checked(vec![0xA1, 0xD8, 0xE9, 0xA7], &DECODING_TABLE_CP874), Some("กุ้ง".to_string()));
// 0x81-0x84,0x86-0x90,0x98-0x9F is invalid in CP874
assert_eq!(decode_string_incomplete_table_checked(vec![0x30, 0x81], &DECODING_TABLE_CP874), None);
// You can use decode_string_incomplete_table_lossy instead
assert_eq!(&decode_string_incomplete_table_lossy(vec![0xA1, 0xD8, 0xE9, 0xA7], &DECODING_TABLE_CP874), "กุ้ง");
// Undefined code points are replaced with U+FFFD (replacement character)
assert_eq!(&decode_string_incomplete_table_lossy(vec![0x30, 0x81], &DECODING_TABLE_CP874), "0\u{FFFD}");
```

## Select appropriate codepage from integer

```rust
use oem_cp::code_table::{ENCODING_TABLE_CP_MAP, DECODING_TABLE_CP_MAP};
use oem_cp::{encoding_string_checked, encoding_string_lossy};

if let Some(cp874_table) = (*DECODING_TABLE_CP_MAP).get(&874) {
    assert_eq!(cp874_table.decode_string_checked(vec![0xA1, 0xD8, 0xE9, 0xA7]), Some("กุ้ง".to_string()));
    // undefined mapping 0x81 for CP874
    assert_eq!(cp874_table.decode_string_checked(vec![0x81]), None);
    assert_eq!(&cp874_table.decode_string_lossy(vec![0x81]), "\u{FFFD}");
} else {
    panic!("Why the hell CP874 isn't registered?");
}

if let Some(cp437_table) = (*ENCODING_TABLE_CP_MAP).get(&437) {
    assert_eq!(encode_string_checked("π≈22/7", cp437_table), Some(vec![0xE3, 0xF7, 0x32, 0x32, 0x2F, 0x37]));
    // ¾ is undefined in CP437
    assert_eq!(encoding_string_checked("½+¼=¾", cp437_table), None);
    // It's replaced with ? (0x3F)
    assert_eq!(encoding_string_lossy("½+¼=¾", cp437_table), vec![0xAB, 0x2B, 0xAC, 0x3D, 0x3F]);
} else {
    panic!("Why the hell CP437 isn't registered?");
}
```

# Support for ANSI/EBCDIC/MBCS code pages

For ANSI (125x) and MBCS (932-950; for CJK languages) code pages, please use [encoding_rs](https://github.com/hsivonen/encoding_rs) instead.

This library is only for extended ASCII encodings (0x00-0x80 must be compatible with ASCII), so EBCDIC encodings will never be supported.

# Symbols from 0x01 to 0x19

This library doesn't support [symbols mapped from 0x01 to 0x19 in CP437](https://en.wikipedia.org/wiki/Code_page_437). 0x01-0x19 are mapped to U+0001-U+0019. If you prefer symbols, use [codepage_437](https://github.com/nabijaczleweli/codepage-437) instead.

# Licenses

MIT
