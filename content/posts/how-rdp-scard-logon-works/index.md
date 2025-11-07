+++
title = "How RDP smart card logon works"
date = 2025-12-01
draft = false

[taxonomies]
tags = ["rust", "freerdp", "scard", "linux", "kerberos", "winscard"]

[extra]
keywords = "Rust, FreeRDP, RDP, Smart Card, Debugging"
toc = true
mermaid = true
+++

# Intro

Around two years ago, FreeRDP got smart card logon support: [Smartcard logon with FreeRDP](https://www.hardening-consulting.com/en/posts/20231004-smartcard-logon.html).
Theoretically (and sometimes practically), it is possible to use a smart card (scard) for RDP logon in FreeRDP. In this post, I will try to explain how RDP scard logon works and how to set it up.

This post has two main parts: theoretical and practical. I was thinking about two separate posts, but then I decided that I don't want to split theory and practice.

After reading this article, you should have enough understanding and skills to set up and troubleshoot your own RDP smart card logon.

_**Note:** It is much easier to do smart card logon on Windows, so I assume we run FreeRDP on Linux (or macOS) and our target machine runs Windows._

You may be surprised (sarcasm), but it is not easy to do scard logon on Linux when connecting to the Windows machine.
It became easier with [recent improvement](https://github.com/Devolutions/sspi-rs/pull/492) in `sspi-rs`. I have a lot to say, let's dive in.

# How it works

Each section explains the theoretical aspects of RDP smart card logon and the software component we will use for it.
Sometimes it is not trivial because not all Microsoft components have exact replacements on Linux.

## RDP NLA

Now we need to understand how the scard authorization works. Without this knowledge we can't move further. The RDP is very complicated thing. So, I'll focus my attention only on scard-related things.

So, the first thing you need to know is [Network Level Authentication](https://en.wikipedia.org/wiki/Remote_Desktop_Services#Network_Level_Authentication):

> Network Level Authentication (NLA) is a feature of RDP Server or Remote Desktop Connection (RDP Client) that requires the connecting user to authenticate themselves before a session is established with the server.
>
> ...
>
> Network Level Authentication delegates the user's credentials from the client through a client-side Security Support Provider and prompts the user to authenticate before establishing a session on the server.

Basically, NLA says: "_Let's authenticate the user (and the server, if the Kerberos protocol is used) before opening any GUI windows at the beginning of the RDP connection_".
And only when the user passes the authentication, the NLA will transfer credentials to the target server.
In turn, the RDP server will use these credentials to open the session.

NLA is not a concrete protocol but a general principle. In RDP, the [CredSSP](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-cssp/85f57821-40bb-46aa-bfcb-ba9590b8fc30) protocol is responsible for the NLA.

{% mermaiddiagram() %}
flowchart LR
    subgraph CREDSSP["CredSSP"]
        subgraph SPNEGO["SPNEGO"]
            subgraph ApplicationProtocol["Negotiated Application Protocol (NTLM, Kerberos, etc)"]
            end
        end
    end
{% end %}

The CredSSP protocol performs the user authentication ([using NTLM/Kerberos/SPNEGO protocols inside](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-cssp/78e0d50f-5a99-45e8-9f8b-d8a2bd67d4eb)) and transfers the user's credentials to the target server.
The CredSSP protocol is only responsible for credentials delegation. It doesn't authenticate clients or servers. An actual auth is performed by the inner application protocol like NTLM, or Kerberos (preferable).

But Microsoft would not be Microsoft if they did not add another layer of abstraction. The CredSSP uses [the SPNEGO framework](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-spng/b16309d8-4a93-4fa6-9ee2-7d84b2451c84): ([quote src](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-cssp/e36b36f6-edf4-4df1-9905-9e53b7d7c7b7))

> SPNEGO provides a framework for two parties that are engaged in authentication to select from a set of possible authentication mechanisms.
>
> ...The CredSSP Protocol uses SPNEGO to mutually authenticate the CredSSP client and CredSSP server. It then uses the encryption key that is established under SPNEGO to securely bind to the TLS session.

Let's summarize it.

The SPNEGO selects (negotiates) an appropriate authentication protocol and performs an authentication using this protocol.
As the result, we'll have an established security context with some secure encryption key.
In turn, the CredSSP will use this key to encrypt the credentials and pass (delegate) them to the target CredSSP (RDP) server.

FreeRDP implements the CredSSP protocol, so there is not any pit falls.
Its CredSSP implementation can rely on either the built-in NTLM implementation, [the MIT KRB5 implementation](https://web.mit.edu/kerberos/krb5-latest/doc/), or an external `/sspi-module:` — a dynamic library that implements the SSPI interface. Spoiler: We will use an external library.

Good. Now you have a brief overview of the NLA. Now it's time to move forward.

## Kerberos

To log on using smart card we need the SPNEGO to select the Kerberos as a application protocol for authentication.
**Kerberos is the only authentication protocol that can be used for scard logon.**
In order to support password-less logon, Kerberos has the PKINIT extension: [Public Key Cryptography for Initial Authentication in Kerberos (PKINIT)](https://datatracker.ietf.org/doc/html/rfc4556).

The only difference between password-based Kerberos and scard-based is in the AS exchange ([The Authentication Service Exchange](https://www.rfc-editor.org/rfc/rfc4120#section-3.1)). During password-based logon, the AS exchange session key is generated from the user's password:

```rs
// https://github.com/Devolutions/picky-rs/blob/628bbcab3100a782971261022f0ec91b4f4549f5/picky-krb/src/crypto/aes/key_derivation.rs#L39-L51
pub fn derive_key_from_password<P: AsRef<[u8]>, S: AsRef<[u8]>>(
    password: P,
    salt: S,
    aes_size: &AesSize,
) -> KerberosCryptoResult<Vec<u8>> {
    let mut tmp = vec![0; aes_size.key_length()];

    pbkdf2_hmac::<Sha1>(password.as_ref(), salt.as_ref(), AES_ITERATION_COUNT, &mut tmp);

    // For AES encryption (https://www.rfc-editor.org/rfc/rfc3962.html#section-6):
    // > random-to-key function        identity function
    let temp_key = random_to_key(tmp);

    // A Key Derivation Function (https://www.rfc-editor.org/rfc/rfc3961.html#section-5.1).
    derive_key(&temp_key, KERBEROS, aes_size)
}
```

Essentially, it means that if someone knows the user's password, they can decrypt the AsRep encrypted part, extract a session key, and decrypt-and-extract all other sequential keys in the Kerberos authentication (e.g. TGS exchange session key).

Thus, always enforce a strong password policy :upside_down_face:. However, I am not here to discuss Kerberos attack vectors and vulnerabilities.
So, what is different in the scard-based logon?

During the scard-based logon, the AS exchange session key is derived using the [the Diffie-Hellman Key Exchange](https://datatracker.ietf.org/doc/html/rfc4556#section-3.2.3.1).
PKINIT also defined the [Public Key Encryption](https://datatracker.ietf.org/doc/html/rfc4556#section-3.2.3.2) but I haven't seen it in action ever.
I always see the Diffie-Hellman Key Exchange in the Wireshark capture of the `mstsc`.

> [Diffie–Hellman (DH) key exchange](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange) is a mathematical method of securely generating a symmetric cryptographic key over a public channel.

It means that two parties can **_securely_** establish an encryption key over a public network.
There are tons of articles explaining how the Diffie-Hellman Key Agreement works.
Let me paste only a small description and move on.

1. The client generates DH parameters and private key.
2. Using the DH parameters and private key, the client generates the public key.
   ```rs
   // https://github.com/Devolutions/picky-rs/blob/628bbcab3100a782971261022f0ec91b4f4549f5/picky-krb/src/crypto/diffie_hellman.rs#L145-L149

   /// [Key and Parameter Requirements](https://www.rfc-editor.org/rfc/rfc2631#section-2.2)
   /// y is then computed by calculating g^x mod p.
   pub fn compute_public_key(private_key: &[u8], modulus: &[u8], base: &[u8]) -> DiffieHellmanResult<Vec<u8>> {
       generate_dh_shared_secret(base, private_key, modulus)
   }
   ```
3. The client sends DH parameters and public key to the server (KDC in our case).
4. The server generates its own private and public keys.
5. Both parties can generate the shared secret (symmetric encryption key) using DH algorithm:
   ```rs
   // https://github.com/Devolutions/picky-rs/blob/628bbcab3100a782971261022f0ec91b4f4549f5/picky-krb/src/crypto/diffie_hellman.rs#L94-L112

   /// [Using Diffie-Hellman Key Exchange](https://www.rfc-editor.org/rfc/rfc4556.html#section-3.2.3.1)
   /// let DHSharedSecret be the shared secret value. DHSharedSecret is the value ZZ
   ///
   /// [Generation of ZZ](https://www.rfc-editor.org/rfc/rfc2631#section-2.1.1)
   /// ZZ = g ^ (xb * xa) mod p
   /// ZZ = (yb ^ xa)  mod p  = (ya ^ xb)  mod p
   /// where ^ denotes exponentiation
   fn generate_dh_shared_secret(public_key: &[u8], private_key: &[u8], p: &[u8]) -> DiffieHellmanResult<Vec<u8>> {
       let public_key = BoxedUint::from_be_slice_vartime(public_key);
       let private_key = BoxedUint::from_be_slice_vartime(private_key);
       let p = Odd::new(BoxedUint::from_be_slice_vartime(p))
           .into_option()
           .ok_or(DiffieHellmanError::ModulusIsNotOdd)?;
       let p = BoxedMontyParams::new_vartime(p);

       // ZZ = (public_key ^ private_key) mod p
       let out = pow_mod_params(&public_key, &private_key, &p);
       Ok(out.to_be_bytes().to_vec())
   }
   ```
6. Using the derived encryption key, the server encrypts all needed data and sends it to the client alongside the DH public key.

Of course, I've omitted many technical details, but I'm here to convey the overall idea, not to teach you cryptography.
The astute reader may notice that the smart card is not required for the Diffie-Hellman exchange.

The smart card is needed **_ONLY_** for signing the digest of all these public authentication parameters: DH parameters, DH public key, nonce, etc ([source code](https://github.com/Devolutions/sspi-rs/blob/ffd920b7b396739b60a6e1bae701f31e735deeef/src/pk_init.rs#L75-L239)).
The KDC will verify the signature using the client's certificate.

That's all :upside_down_face:. Someday I will write a detailed post about Kerberos scard auth, but today is not that day :grimacing:.

## Scard credentials

The CredSSP client transfers user's username, password, and domain to the target server during password-based logon.
In turn, the target server tries to login the user using this credentials.
But we cannot send the smart card to the target server :upside_down_face:. According to the specification, smart card credentials are defined as follows:

```js
-- https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-cssp/94a1ab00-5500-42fd-8d3d-7a84e6c2cf03
TSCredentials ::= SEQUENCE {
        credType    [0] INTEGER,
        credentials [1] OCTET STRING
}
-- credType = 2. Credentials contains a TSSmartCardCreds structure that defines the user's smart card credentials. 

-- https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-cssp/4251d165-cf01-4513-a5d8-39ee4a98b7a4
TSSmartCardCreds ::= SEQUENCE {
        pin         [0] OCTET STRING,
        cspData     [1] TSCspDataDetail,
        userHint    [2] OCTET STRING OPTIONAL,
        domainHint  [3] OCTET STRING OPTIONAL
}
-- https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-cssp/34ee27b3-5791-43bb-9201-076054b58123
TSCspDataDetail ::= SEQUENCE {
        keySpec       [0] INTEGER,
        cardName      [1] OCTET STRING OPTIONAL,
        readerName    [2] OCTET STRING OPTIONAL,
        containerName [3] OCTET STRING OPTIONAL,
        cspName       [4] OCTET STRING OPTIONAL
}
```

I want to go through every field and explain their meaning:

- `credType` = `2` (according to the specification).
- `pin` - smart card PIN code.
- `userHint` - username. **Important**: the username must be **only** username. The target server [may reject](https://github.com/Devolutions/sspi-rs/pull/494) the FQDN. `t2@example.com` :x: -> `t2` :white_check_mark:.
- `domainHint` - domain.
- `keySpec` = `AT_KEYEXCHANGE` = `1`. ([AD FS and certificate KeySpec property information](https://learn.microsoft.com/en-us/windows-server/identity/ad-fs/technical-reference/ad-fs-and-keyspec-property)):
  > The KeySpec property identifies how a key, that is generated or retrieved using the Microsoft CryptoAPI (CAPI) from a Microsoft legacy Cryptographic Storage Provider (CSP), can be used.

  Typically, this value equals `AT_KEYEXCHANGE`. Since this value is not encoded in the certificate or smart card, we can set it manually in the credentials.
- `cardName` - smart card name. It can be any name.
- `cspName`. [Cryptographic Service Provider](https://learn.microsoft.com/en-us/windows/win32/secgloss/c-gly):

  > An independent software module that actually performs cryptography algorithms for authentication, encoding, and encryption.

  In our case, it is always equal to [`Microsoft Base Smart Card Crypto Provider`](https://learn.microsoft.com/en-us/windows/win32/seccertenroll/cryptoapi-cryptographic-service-providers#microsoft-base-smart-card-crypto-provider).
  It supports smart cards and implements algorithms to hash, sign, and encrypt data. We do not need to know internals, but only that it uses smart cards for crypto operations.
- `readerName`. ([quote src](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpesc/d361b24c-611e-46c5-9ac6-93cb6490a472)):
  > - **_smart card reader name_**: The friendly, human-readable name of the _smart card reader_. Also referred to as a _Reader Name_.
  > - **_smart card reader_**: A device used as a communication medium between the smart card and a Host; for example, a computer. Also referred to as a Reader.

  The system automatically generates the Reader Name on Windows. But we need to take it from somewhere on Linux. In short, the PKCS11 slot description is used as the reader name (explained below).
- `containerName`. ([quote src](https://learn.microsoft.com/en-us/windows/win32/secgloss/k-gly)):
  > key container - A part of the key database that contains all the key pairs (exchange and signature key pairs) belonging to a specific user. Each container has a unique name that is used when calling the `CryptAcquireContext` function to get a handle to the container.

  Therefore, the container name is a _unique_ string that represents the smart card certificate and its private key.
  And, of course, it is internally generated during scard certificate enrollment on Windows.
  We cannot precalculate or extract it from somewhere. A small trick is used to generate this value:

  ```rust
  // https://github.com/Devolutions/sspi-rs/blob/dea60b5ef60932efdbf22e0806954d8c236fbb78/ffi/src/winscard/piv.rs#L134C1-L212
  fn chuid_to_container_name(chuid: &[u8], tag: [u8; 3]) -> Result<String> {
      // ... (some lines are omitted) ...
      // `chuid` - smart card Card Holder Unique Identifier.

      let guid = &chuid[BYTES_TO_SKIP..BYTES_TO_SKIP + 16 /* GUID length */];

      // Construct the value Windows would use for a PIV key's container name.
      let container_name = format!(
          "{:02x}{:02x}{:02x}{:02x}-{:02x}{:02x}-{:02x}{:02x}-{:02x}{:02x}-{:02x}{:02x}{:02x}{:02x}{:02x}{:02x}",
          guid[3],
          guid[2],
          guid[1],
          guid[0],
          guid[5],
          guid[4],
          guid[7],
          guid[6],
          guid[8],
          guid[9],
          guid[10],
          guid[11],
          guid[12],
          tag[0],
          tag[1],
          tag[2]
      );

      Ok(container_name)
  }
  ```

  This is how [FreeRDP constructs](https://github.com/FreeRDP/FreeRDP/blob/3fc1c3ce31b5af1098d15603d7b3fe1c93cf77a5/winpr/libwinpr/ncrypt/ncrypt_pkcs11.c#L865-L964) a container name for the certificate.
  It takes a smart card CHUID, extracts a GUID value, and combines it with the certificate PIV tag to construct a unique container name.
  Since every smart card CHUID is unique and there is only one certificate per slot, the constructed value is guaranteed to be unique.
  Moreover, the uniqueness is not the critical part. The crucial part is that the container name format must meet Windows expectations.
  Last year, in one of my posts, I explained and provided an example of how a bad container name can cause issues in smart card logon: [tbt.qkation.com/posts/scard-container-name/#what](https://tbt.qkation.com/posts/scard-container-name/#what).

## Scard driver

As explained above, the Kerberos protocol does client and server auth. And, as we can logically assume, it relies on some APIs to communicate with the smart card.

Let's first discuss how it works on Windows, and then we will move on to Linux. Look at [this diagram from the MSDN](https://learn.microsoft.com/en-us/windows/security/identity-protection/smart-cards/smart-card-architecture):

![](sc-image203.gif)

The Kerberos implementation uses the Windows high-level cryptographic API to communicate with the smart card. Using [the certificate hash](https://learn.microsoft.com/en-us/windows/win32/api/wincred/ns-wincred-cert_credential_info), it can get the user's certificate, container name, and all other smart card credentials.
Windows allows using vendor-specific CSPs, but the default `basecsp.dll` is used during RDP smart card auth.

In turn, the `basecsp.dll` uses the smart card minidriver to perform cryptographic operations.

[Smart Card Minidrivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/smartcard/smart-card-minidrivers):

> The smart card minidriver provides a simpler alternative to developing a legacy cryptographic service provider (CSP) by encapsulating most of the complex cryptographic operations from the card minidriver developer.
>
> Applications use CAPI for smart card–based cryptographic services. CAPI, in turn, routes these requests to the appropriate CSP to handle the cryptographic requirements.
> Although Base CSP can use the RSA capabilities of a smart card only by using the minidriver...

[Smart Card Minidriver Overview](https://learn.microsoft.com/en-us/windows-hardware/drivers/smartcard/smart-card-minidriver-overview):

> The card-specific minidriver is the lowest logical interface layer in the Base CSP/KSP. This minidriver lets the Base CSP/KSP and applications interact directly with a specific type of card by using SCRM.

In simple words, the smart card minidriver is a small dll provided by the card vendor that implements the Smart Card Minidriver specification.
This dll is used by BaseCSP to perform basic cryptography operations. This allows card vendors to reuse Microsoft's cryptography stack and not implement their own CSP:

![](capiinterface.png)

I have the [YubiKey 5 Nano](www.yubico.com/ua/product/yubikey-5-nano/) smart card and will use it in this article.
Of course, YubiKey has its own [smart card minidriver implementation](https://www.yubico.com/support/download/smart-card-drivers-tools/), which must be installed on the target machine: [Deploying the YubiKey Smart Card Minidriver to workstations and servers](https://support.yubico.com/hc/en-us/articles/360015654560-Deploying-the-YubiKey-Smart-Card-Minidriver-to-workstations-and-servers).

But what about Linux? We do not have minidrivers here... FreeRDP (and `sspi-rs`) relies on the PKCS11 module (dynamic library) for executing crypto operations.

> In cryptography, **PKCS #11** is a Public-Key Cryptography Standard that defines a C programming interface to create and manipulate cryptographic tokens that may contain secret cryptographic keys. It is often used to communicate with a Hardware Security Module or smart cards. ([Wiki](https://en.wikipedia.org/wiki/PKCS_11))

Fortunately, YubiKey distributes its own PKCS11 library too: [`libykcs11`](https://developers.yubico.com/yubico-piv-tool/YKCS11/).

However, that's not the end: each smart card minidriver communicates with the smart card through a unified API: [WinSCard API](https://learn.microsoft.com/en-us/windows/win32/api/winscard/). :arrow_down:

## WinSCard

> The basic parts of the smart card subsystem are based on PC/SC standards (see the specifications at [https://pcscworkgroup.com](https://pcscworkgroup.com)). ([src](https://learn.microsoft.com/en-us/windows/win32/secauthn/smart-card-authentication))

The WinSCard API enables communications with smart card hardware. It is the lowest API from the software perspective.

The WinSCard API is based on the PC/SC (Personal Computer/Smart Card) specification - a globally implemented standard for cards and readers ([https://www.smartcardbasics.com/pc-sc/](https://www.smartcardbasics.com/pc-sc/)).

> The advantage of PC/SC is that applications do not have to acknowledge the details corresponding to the smart card reader when communicating with the smart card. This application can function with any reader that complies with the PC/SC standard. ([src](https://www.cardlogix.com/glossary/pc-sc/)).

On Windows we have WinSCard API, but on Linux we have [pcsc-lite](https://pcsclite.apdu.fr/):

> PC/SC is the de facto cross-platform API for accessing smart card readers. It is published by [PC/SC Workgroup](http://www.pcscworkgroup.com/) but the _"reference implementation"_ is Windows. Linux and Mac OS X use the open source [pcsc-lite](https://pcsclite.apdu.fr/) package. ([src](https://github.com/OpenSC/OpenSC/wiki/PCSC-and-pcsc-lite))

Everything looks good from the first sight: `libykcs11` uses `pcsc-lite`, both of them are available on Linux. Seems like nothing stops us from scard logon.
Unfortunately, there is one thing.

The target machine tries to open a new session using the transferred smart card credentials. Internally, the same high-lever APIs are used except WinSCard API.
All WinSCard API calls are redirected to the client machine (via [[MS-RDPESC]: Remote Desktop Protocol: Smart Card Virtual Channel Extension](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-rdpesc/0428ca28-b4dc-46a3-97c3-01887fa44a90)). Soooo?
Obviously, we will call `pcsc-lite` API on linux instead WinSCard, because we do not have WinSCard API here. The problem is that the `pcsc-lite` API does not match WinSCard API identically.
There are differences:

* Most of them are listed in the documentation: [https://pcsclite.apdu.fr/api/group__API.html#differences](https://pcsclite.apdu.fr/api/group__API.html#differences).
* `pcsc-lite` API does not contain some functions as WinSCard. For example: `SCardReadCache`.
* Some function has `-A` and `-W` variations. For example: `SCardConnectA` and `SCardConnectW`.

A `pcsc-lite` wrapper needed to be implemented to match the original WinSCard API. [FreeRDP's  implementation](https://github.com/FreeRDP/FreeRDP/blob/3fc1c3ce31b5af1098d15603d7b3fe1c93cf77a5/winpr/libwinpr/smartcard/smartcard_pcsc.c#L3185-L3265) is not complete.
It lacks a proper WinSCard cache implementation. The target machine expects the client side to have the needed WinSCard smart card cache items.

It is a real problem: the target machine will not authenticate the client without a properly working smart card cache. Otherwise, the target machine will display a message such as _"This smart card could not be used. Additional detail may be available in the system log. Please report this error to your administrator."_ or _"The requested key container does not exist on the smart card."_. If you are interested in smart card cache items, then check my old post: [tbt.qkation.com/posts/win-scard-cache/](https://tbt.qkation.com/posts/win-scard-cache/).

To my knowledge, the `sspi-rs`'s FFI module is the only open-source library that implements proper smart card cache items: [github/Devolutions/sspi-rs/dea60b5e/ffi/src/winscard/system_scard/context.rs#L283-L564](https://github.com/Devolutions/sspi-rs/blob/dea60b5ef60932efdbf22e0806954d8c236fbb78/ffi/src/winscard/system_scard/context.rs#L283-L564).
Now, with this final software component, we can finally set up FreeRDP scard logon :star_struck: :hot_face:.


## Let's put it all together

| software component | meaning | Windows | Linux |
|-|-|-|-|
| RDP client | _no comments_ | `mstsc.exe` | FreeRDP |
| CredSSP protocol | Performs NLA. Transfers user credentials to the target server | `credssp.dll` | FreeRDP |
| Kerberos protocol | Authenticates the user and the server. Established the security context | Implemented inside Windows | [`libsspi.so`](https://github.com/Devolutions/sspi-rs/tree/master/ffi/src/sspi) - Kerberos client implementation with scard logon support |
| Smart card (mini)driver | Transforms high-level crypto operations into low-level PCSC commands | [YubiKey Smart Card Minidriver](https://www.yubico.com/support/download/smart-card-drivers-tools/) | [`libykcs11.so`](https://developers.yubico.com/yubico-piv-tool/YKCS11/) |
| PC/SC | Enables communications with smart card hardware | WinSCard | [`libsspi.so`](https://github.com/Devolutions/sspi-rs/tree/master/ffi/src/winscard) - WinSCard-compatible `pcsc-lite` wrapper with proper smart card cache implementation |

At this point, I hope it all makes sense to you :crossed_fingers:.

# Setting up

## Smart card

## Kerberos

## Demo

# Doc, references, code

- [Smartcard logon with FreeRDP](https://www.hardening-consulting.com/en/posts/20231004-smartcard-logon.html).
- [Network Level Authentication](https://en.wikipedia.org/wiki/Remote_Desktop_Services#Network_Level_Authentication).
- [CredSSP](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-cssp/85f57821-40bb-46aa-bfcb-ba9590b8fc30).
- [Public Key Cryptography for Initial Authentication in Kerberos (PKINIT)](https://datatracker.ietf.org/doc/html/rfc4556).
- [Smart Card Minidrivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/smartcard/smart-card-minidrivers).
- [Smart Card Architecture](https://learn.microsoft.com/en-us/windows/security/identity-protection/smart-cards/smart-card-architecture).
- [WinSCard API](https://learn.microsoft.com/en-us/windows/win32/api/winscard/).