+++
title = "Windows smart card cache"
date = 2024-04-04
draft = false
template = "post.html"

[taxonomies]
tags = ["debugging", "windows", "rust", "scard"]

[extra]
keywords = "Rust, API Monitor, Debugging, API, Windows, Smart Card"
toc = true
thumbnail = "win-scard-caches-thumbnail.png"
+++

# Getting Started

A few months ago I had a great opportunity to implement [the smart card emulation](https://github.com/Devolutions/sspi-rs/pull/210). The whole point was to emulate the smart card behavior without an actual smart card. I can talk a lot about such stuff, but in this article, I'm going to share my experience in smart card cache exploration and implementation.

To make all my decisions clear for you, I provide you with some additional context. Imagine the reimplemented `winscard.dll` without actual calls to the security device but with the smart card emulation under the hood. Boom :boom:! We have scard auth without a physical device.

You can have questions like *"How can we replace the original dll"*, *"What is the point of such implementation"*, *"What are the limitations of it"*, and so on. Unfortunately, it's out of the scope of this article. Here I focused only on debugging and smart card cache (I'll explain the reason for it further in the article).

## Goals

* A brief overview of the smart card architecture in Windows.
* Explain some smart card cache items for [**PIV**](https://www.idmanagement.gov/university/piv/) smart cards in Windows.
* Fun :partying_face:.

Who should read this article:

* Ones who are interested in how PIV smart cards work on Windows.
* Ones who just need PIV scard cache items format (Hi :wave:, `msclmd.dll`).

## Non-goals

* Explain every piece of smart card architecture in Windows.

# Overview

In Windows, [WinSCard](https://learn.microsoft.com/en-us/windows/win32/api/winscard/) is the lowest accessible API for communicating with smart cards. You operate only using handles (`SCARDCONTEXT`/`SCARDHANDLE`/etc), raw [APDU](https://en.wikipedia.org/wiki/Smart_card_application_protocol_data_unit)s ([SCardTransmit](https://learn.microsoft.com/en-us/windows/win32/api/winscard/nf-winscard-scardtransmit) function), and a bunch of other low-level functions. If you look at [the CSP and KSP-based architecture diagram](https://learn.microsoft.com/en-us/windows/security/identity-protection/smart-cards/smart-card-architecture#base-csp-and-ksp-based-architecture-in-windows), you'll see WinScard almost at the bottom of the picture:

[![](https://learn.microsoft.com/en-us/windows/security/identity-protection/smart-cards/images/sc-image206.gif)](https://learn.microsoft.com/en-us/windows/security/identity-protection/smart-cards/smart-card-architecture#base-csp-and-ksp-based-architecture-in-windows)

Let's analyze this diagram above. Usually, when you deal with any crypto operations in Windows, you use some high-level API - [CryptoAPI](https://learn.microsoft.com/en-us/windows/win32/seccrypto/cryptoapi-system-architecture). But the CryptoAPI is just a high-level API and every [CSP](https://en.wikipedia.org/wiki/Cryptographic_Service_Provider) implements it. In the case of smart cards, the corresponding CSP (`basecsp.dll`, in our case) can not contain the CryptoAPI implementation because all crypto operations should be performed on the security device. This is why BaseCSP uses (depends on) the smart card minidriver. Such a minidriver translates high-level functions into WinSCard API calls. Nothing more.

Usually, every smart card vendor ships its smart card driver (you can see it on the diagram). The Windows itself contains two smart card drivers:

* One driver for Windows proprietary smart cards with closed specifications and documentation.
* And another driver for PIV-compatible smart cards. It has open specifications ([Smart Card Minidriver Specification](https://learn.microsoft.com/en-us/previous-versions/windows/hardware/design/dn631754(v=vs.85)) + [NIST SP 800-73-4](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-73-4.pdf)) but closed documentation.

> *"...two smart card drivers:"*

It was a small lie :sweat_smile:. I mean, yes, it's true, but those two drivers are implemented inside one dll: `msclmd.dll`. *For the rest of this article, we'll talk **only** about `msclmd.dll` and **PIV-compatible** smart cards.*

> *Got it. But why is card cache the main topic of this article?*

I have a few reasons for it:

* No matter how good our WinSCard implementation is, authentication will fail without a working cache.
* Only a few cache items are described in the Window's minidriver specification. Many of them are undocumented and unknown (or known but the final file structure is not clear).
* It was the hardest part to debug and implement :stuck_out_tongue_closed_eyes:. It took me a few months.

Firstly, I thought I can found cache items format in the specification. But I was wrong. Secondly, I was astonished that I didn't found any information about them in the Internet. In the result, it took me more then month to properly reverse/debug `msclmd.dll` and extract all needed information about scard cache items. And I decided to write an article you are reading now.

> ***Note**. I didn't reverse the whole smart card driver. I explored only needed functions. So, this article has gaps in some places filled with my assumptions. If you are more experienced and know more about such places, please, leave a comment and I'll edit the article to be more complete. Thanks*:relieved:

# Debugging

It's easier to work if we have a concrete destination point. Let's take the Remote Desktop Client (`mstsc.exe`), replace the `wiscard.dll`, and try to connect to the remote machine using the emulated smart card. The established RDP connection is our goal.

To make my life easier, I took the [MsRdpEx](https://github.com/Devolutions/MsRdpEx/) and hooked the `winscard.dll` using the [`MSRDPEX_WINSCARD_DLL`](https://github.com/Devolutions/MsRdpEx/pull/76) environment variable. Pretty simple and usable.

For the further work, only two instruments will be used:

* IDA.
* [WinDbg](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/).

[**Time Travel Debugging**](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/time-travel-debugging-overview). I encourage you to try it! This technique saved me a lot of time. The possibility of re-playing and moving backward gives us almost unlimited power :muscle:.

I do **not** recommend debugging smart cards with [API Monitor](http://www.rohitab.com/apimonitor) because it can miss some WinSCard API function calls.

# Let's start the journey

I suppose the whole debugging process is boring for you, so I'll show relevant reversed parts of the `msclmd.dll` with small descriptions. If you need only resulting cache items format (structure), then you should skip next sections and jump right to the [implementation](https://github.com/Devolutions/sspi-rs/blob/4409f9a5235dec0c033edce654aa6fe934a72afc/crates/winscard/src/scard_context.rs#L146-L394).

At some point in time, the `msclmd.dll` calls the [`SCardReadCacheW`](https://learn.microsoft.com/en-us/windows/win32/api/winscard/nf-winscard-scardreadcachew) function to extract some information from the smart card cache. What data should we return? What format does the data (cache file) have? The next chapters contain answers to those questions.

You can use the following material in two cases:

* You are developing your own `winscard.dll` replacement.
* You are using the WinSCard API (maybe developing your minidriver) under Windows and want to know what the `SCardReadCacheW` function returns.

Otherwise, it's just a fun reading for you. Enjoy :kissing_closed_eyes:

## `Cached_CardmodFile\Cached_Pin_Freshness`

Usually, when the driver asks you for the `Cached_CardmodFile\Cached_Pin_Freshness` cache item, it wants to form the [`CARD_CACHE_FILE_FORMAT`](https://github.com/selfrender/Windows-Server-2003/blob/5c6fe3db626b63a384230a1aa6b92ac416b0765f/ds/security/csps/wfsccsp/inc/basecsp.h#L121-L128) structure and the PIN freshness counter is the first field in the structure:

```c
typedef struct _CARD_CACHE_FILE_FORMAT
{
    BYTE bVersion;
    BYTE bPinsFreshness;
    WORD wContainersFreshness;
    WORD wFilesFreshness;
} CARD_CACHE_FILE_FORMAT, *PCARD_CACHE_FILE_FORMAT;
```

The actual cache file is just one-byte number that represents a PIN freshness counter. If you want, it can be two-byte value, but the value will be casted to one-byte. [Implementation example](https://github.com/Devolutions/sspi-rs/blob/4409f9a5235dec0c033edce654aa6fe934a72afc/crates/winscard/src/scard_context.rs#L383-L386):

```rust
const PIN_FRESHNESS: [u8; 2] = [0x00, 0x00];

cache.insert(
    "Cached_CardmodFile\\Cached_Pin_Freshness".into(),
    PIN_FRESHNESS.to_vec(),
);
```

All three freshness counters are extracted from the card cache using the `msclmd!I_GetPIVCachedFreshnessCounter` function. Every freshness counter has an ID:

* `1` - PIN freshness.
* `2` - file freshness.
* `3` - container freshness.

> *What about `bVersion` field?*

Currently, it's always equal to `1` (this value is hardcoded in the DLL. you'll see it on the next screenshots).

But the `CARD_CACHE_FILE_FORMAT` structure creation is not just cache items copying to the structure fields. The driver has an additional algorithm. Okay, let's move on.

## `Cached_CardmodFile\Cached_File_Freshness`

We are already familiar with freshness counters, so I skip this cache item introduction and jump to [the implementation](https://github.com/Devolutions/sspi-rs/blob/4409f9a5235dec0c033edce654aa6fe934a72afc/crates/winscard/src/scard_context.rs#L387-L390):

```rust
const FILE_FRESHNESS: [u8; 2] = [0x0b, 0x00];

cache.insert(
    "Cached_CardmodFile\\Cached_File_Freshness".into(),
    FILE_FRESHNESS.to_vec(),
);
```

Note. The `[0x0b, 0x00]` is almost random value. The freshness counter is just a counter. More importantly, this counter matches the next cache items (files) in this article. I'll explain later what I mean.

## `Cached_CardmodFile\Cached_Container_Freshness`

And finally, the container freshness counter. [Code](https://github.com/Devolutions/sspi-rs/blob/4409f9a5235dec0c033edce654aa6fe934a72afc/crates/winscard/src/scard_context.rs#L391-L394):

```rust
const CONTAINER_FRESHNESS: [u8; 2] = [0x01, 0x00];

cache.insert(
    "Cached_CardmodFile\\Cached_Container_Freshness".into(),
    CONTAINER_FRESHNESS.to_vec(),
);
```

> *So the driver can finally create a `CARD_CACHE_FILE_FORMAT` structure?*

*Well yes but actually no.* Let's see a small reversed part of the `msclmd!I_GetPIVCache` function:

![](CARD_CACHE_FILE_FORMAT_creation.png)

> *What is an `activity_count`?*

It's a result value of the `msclmd!I_GetCardActivityCount` function. Here is how it calculated:

![](activity_count_calculation.png)

The activity counter is calculated based on the `dwEventState` value returned from the [`SCardGetStatusChangeW`](https://learn.microsoft.com/en-us/windows/win32/api/winscard/nf-winscard-scardgetstatuschangew) function. Furthermore, this value also affects the `CARD_CACHE_FILE_FORMAT` structure creation. I pay attention to it on purpose because it's important to know the structure fields' values.

## `Cached_CardProperty_Read Only Mode_0`

To make further explanations easier, I introduce the cache value header:

```c
typedef struct _CARD_FILE_HEADER
{
    CARD_CACHE_FILE_FORMAT file_format;
    uint16 padding1;
    uint16 padding2;
    uint16 padding3;
    uint32 value_len;
} CARD_FILE_HEADER;
```

`value_len` bytes are placed right after this header. I can assume that padding bytes also have a meaning but in my practice, they are always equal to zero and I didn't find any application for them.

This is a general layout for cache items related to the PIV smart card. Now we can finally see, how freshness and activity counters affect other cache files.

In the case of the current cache item, the `value_len` is 4, and value bytes represent the `BOOL` flag. [Example implementation](https://github.com/Devolutions/sspi-rs/blob/4409f9a5235dec0c033edce654aa6fe934a72afc/crates/winscard/src/scard_context.rs#L171-L181). The meaning of this flag hides in the specification. It represents the `CP_CARD_READ_ONLY` card property:

> *If True, all write operations are blocked at the Base CSP layer. This flag also affects the data cache. If the card indicates that it is read-only, the Base CSP/KSP does not write to the cardcf file.*

## `Cached_CardProperty_Cache Mode_0`

The structure of this cache item is the same as in [the previous one](#cached-cardproperty-read-only-mode-0). The `value_len` is 4, and value bytes represent the `DWORD` number. [Example implementation](https://github.com/Devolutions/sspi-rs/blob/4409f9a5235dec0c033edce654aa6fe934a72afc/crates/winscard/src/scard_context.rs#L182-L192). It corresponds to the `CP_CARD_CACHE_MODE` card property and can have one of the possible values:

```c
#define CP_CACHE_MODE_GLOBAL_CACHE      1
#define CP_CACHE_MODE_SESSION_ONLY      2
#define CP_CACHE_MODE_NO_CACHE          3
``` 

## `Cached_CardProperty_Supports Windows x.509 Enrollment_0`

The `value_len` is equal to 4 in the `CARD_FILE_HEADER` structure and those 4 bytes represent the `BOOL` value. [Example implementation](https://github.com/Devolutions/sspi-rs/blob/4409f9a5235dec0c033edce654aa6fe934a72afc/crates/winscard/src/scard_context.rs#L193-L203). The meaning of this flag is the following:

> *Indicates whether Windows PKI should be allowed to write or renew certificates on the card. This should be used to avoid unexpected results because of a lack of support for multiple PINs in Windows PKI enrollment client.*

## `Cached_GeneralFile/mscp/cmapfile`

It is worth noting that smart cards can have a few [key containers](https://stackoverflow.com/a/2528335/15125407). And here every card container are represented in the [`CONTAINER_MAP_RECORD`](https://github.com/selfrender/Windows-Server-2003/blob/5c6fe3db626b63a384230a1aa6b92ac416b0765f/ds/security/csps/wfsccsp/inc/basecsp.h#L104-L110) structure:

```c
#define MAX_CONTAINER_NAME_LEN 40

typedef struct _CONTAINER_MAP_RECORD
{
    WCHAR wszGuid [MAX_CONTAINER_NAME_LEN];
    BYTE bFlags;        
    WORD wSigKeySizeBits;
    WORD wKeyExchangeKeySizeBits;
} CONTAINER_MAP_RECORD, *PCONTAINER_MAP_RECORD;
```

The current cache item describes one or more such container records. The `value_len` is equal to *`0x56` (size of the `CONTAINER_MAP_RECORD`) * amount of container records structures*. [Example implementation](https://github.com/Devolutions/sspi-rs/blob/4409f9a5235dec0c033edce654aa6fe934a72afc/crates/winscard/src/scard_context.rs#L204-L225).

## `Cached_CardmodFile\Cached_CMAPFile`

This one is almost the same as [the previous one](#cached-generalfile-mscp-cmapfile) but without the `CARD_FILE_HEADER` structure at the start of the cache file. You can just remove the first 16 bytes from the `Cached_GeneralFile/mscp/cmapfile` cache file and you'll get this one. [Example implementation](https://github.com/Devolutions/sspi-rs/blob/4409f9a5235dec0c033edce654aa6fe934a72afc/crates/winscard/src/scard_context.rs#L226-L240). The driver (`msclmd.dll`) usually uses this cache file to find the needed card container specified in the credentials. For example, `basecsp!I_ContainerMapFind`, or `basecsp!ContainerMapFindContainer`, or `basecsp!FindCard` functions.

## `Cached_ContainerProperty_PIN Identifier_0`

The `value_len` is equal to 4 in the `CARD_FILE_HEADER` structure and those 4 bytes represent the `DWORD` value. [Example implementation](https://github.com/Devolutions/sspi-rs/blob/4409f9a5235dec0c033edce654aa6fe934a72afc/crates/winscard/src/scard_context.rs#L241-L251). As I understand from the specification, the PIN identifier is associated with the role.

```c
typedef     DWORD                      PIN_ID, *PPIN_ID;

#define     ROLE_EVERYONE              0
#define     ROLE_USER                  1
#define     ROLE_ADMIN                 2
```

## `Cached_ContainerInfo_00`

This cache file describes the key container for more information about which keys are present. The `value_len` of the `CARD_FILE_HEADER` structure depends on the key inside of this container. You'll see it further. Here is [the example implementation](https://github.com/Devolutions/sspi-rs/blob/4409f9a5235dec0c033edce654aa6fe934a72afc/crates/winscard/src/scard_context.rs#L252-L312). It's easier to follow the explanation when you have the code.

After the `CARD_FILE_HEADER` structure we have a 16-byte container info header. The purpose of the first 12 bytes is unknown to me, but the last 4 is the length of the rest of the data (which also depends on the container key). The actual data has the following format:

```C
// This structure is assembled by me
typedef struct _CONTAINER_INFO_DATA {
  PUBLICKEYSTRUC public_key_info;
  RSAPUBKEY rsa_pub_key;
  BYTE[] public_key_modulus;
} CONTAINER_INFO_DATA;

// https://learn.microsoft.com/en-us/windows/win32/api/wincrypt/ns-wincrypt-publickeystruc
typedef struct _PUBLICKEYSTRUC {
  BYTE   bType;
  BYTE   bVersion;
  WORD   reserved;
  ALG_ID aiKeyAlg;
} BLOBHEADER, PUBLICKEYSTRUC;

// https://learn.microsoft.com/en-us/windows/win32/api/wincrypt/ns-wincrypt-rsapubkey
typedef struct _RSAPUBKEY {
  DWORD magic;
  DWORD bitlen;
  DWORD pubexp;
} RSAPUBKEY;
```

*Note.* If the `public_key_modulus` is smaller than the specified key length (for example, by one byte), then it should be padded with zeroes to match the key length.

> _But you wrote above the smart card can have **more than one** key container. Which one should we use here?_

It's a trick question. I don't know :disappointed:. In my experience, I was working with smart cards that have only one key container.

## `Cached_GeneralFile/mscp/kxc00`

In short, this cache file contains a compressed DER-encoded certificate usinf ZLIB compression. [Example implementation](https://github.com/Devolutions/sspi-rs/blob/4409f9a5235dec0c033edce654aa6fe934a72afc/crates/winscard/src/scard_context.rs#L313-L330). [Compression](https://github.com/Devolutions/sspi-rs/blob/4409f9a5235dec0c033edce654aa6fe934a72afc/crates/winscard/src/compression.rs#L11). Of course, the `value_len` of the `CARD_FILE_HEADER` structure depends on the compressed data length. So, move on. The actual data has the following format:

```c
// This structure is assembled by me
typedef struct _CONTAINER_INFO_DATA {
  int16 flags;
  int16 uncompressed_cert_len;
  BYTE[] compressed_cert_len;
} CONTAINER_INFO_DATA;

```

I suppose the first two bytes should indicate if the certificate is compressed or not. But I didn't find any evidence about it and just used extracted values from the real smart card.

> _Again. What certificate should we use?_

This time I know :stuck_out_tongue_closed_eyes:. In the specification, `kxc00` is _key exchange cert 0_.

## `Cached_CardProperty_Capabilities_0`

This cache file describes the card and card-specific minidriver combination for the functionality that is provided at this level, such as a certificate or file compression. The `value_len` is equal to 12 in the `CARD_FILE_HEADER` structure and those 12 bytes represent [the `CARD_CAPABILITIES` structure](https://github.com/selfrender/Windows-Server-2003/blob/5c6fe3db626b63a384230a1aa6b92ac416b0765f/ds/security/csps/wfsccsp/inc/cardmod.h#L138-L152):

```c
#define CARD_CAPABILITIES_CURRENT_VERSION 1

typedef struct _CARD_CAPABILITIES
{
    // The version of the structure that is being used.
    DWORD   dwVersion;
    // Set TRUE to indicate that the card minidriver implements its own compression of certificates.
    BOOL    fCertificateCompression;
    // Set TRUE to indicate that the card can generate keys.
    BOOL    fKeyGen;
} CARD_CAPABILITIES, *PCARD_CAPABILITIES;
```

[Example implementation](https://github.com/Devolutions/sspi-rs/blob/4409f9a5235dec0c033edce654aa6fe934a72afc/crates/winscard/src/scard_context.rs#L331-L343). (_Just a reminder. In Windows, the `BOOL` type is 4-byte long_ :confused:)

## `Cached_CardProperty_Key Sizes_1`

This cache file describes the different key length values that are available. The `value_len` is equal to 20 in the `CARD_FILE_HEADER` structure and those 20 bytes represent [the `CARD_KEY_SIZES` structure](https://github.com/selfrender/Windows-Server-2003/blob/5c6fe3db626b63a384230a1aa6b92ac416b0765f/ds/security/csps/wfsccsp/inc/cardmod.h#L577-L591):

```c
#define CARD_KEY_SIZES_CURRENT_VERSION 1

typedef struct _CARD_KEY_SIZES
{
    DWORD dwVersion;
    DWORD dwMinimumBitlen;
    DWORD dwDefaultBitlen;
    DWORD dwMaximumBitlen;
    DWORD dwIncrementalBitlen;
} CARD_KEY_SIZES, *PCARD_KEY_SIZES;
```

[Example implementation](https://github.com/Devolutions/sspi-rs/blob/4409f9a5235dec0c033edce654aa6fe934a72afc/crates/winscard/src/scard_context.rs#L364-L381). Sometimes the driver can ask for the `Cached_CardProperty_Key Sizes_2` cache item. I'm not sure what this number means. Maybe it depends on the key container, maybe on some algorithm id. IDK :sweat:

# Conclusion

I'm not sure what to conclude. The only thing that stuck in my head is that anything related to smart cards is complex, hard to implement/understand, or find documentation. Anyway, we have no other choice than to live with what we have and try to improve it :blush:

I hope if one day someone is stuck with a similar problem as me, then they'll find this article and use it as a reference. I just don't know who else might need such a detailed description of smart card cache files :sweat_smile:

# Doc, references, code

1. [Smart Card Architecture](https://learn.microsoft.com/en-us/windows/security/identity-protection/smart-cards/smart-card-architecture).
2. [`winscard.h`](https://learn.microsoft.com/en-us/windows/win32/api/winscard/).
3. [Smart Card Minidrivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/smartcard/smart-card-minidrivers).
4. [Smart Card Minidriver Specification](https://learn.microsoft.com/en-us/previous-versions/windows/hardware/design/dn631754(v=vs.85)).
5. Implemented smart card caches: [`scard_context.rs`](https://github.com/Devolutions/sspi-rs/blob/4409f9a5235dec0c033edce654aa6fe934a72afc/crates/winscard/src/scard_context.rs#L146-L394).
6. [Time Travel Debugging](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/time-travel-debugging-overview).
