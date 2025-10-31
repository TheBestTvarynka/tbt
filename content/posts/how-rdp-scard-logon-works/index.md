+++
title = "How RDP smart card logon works"
date = 2025-09-30
draft = false

[taxonomies]
tags = ["rust", "freerdp", "scard", "linux", "debugging", "kerberos"]

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

_**Note:** In this post, I assume that we want to connect from Linux to Windows using FreeRDP._

# How it works

## RDP NLA

Now we need to understand how the scard authorization works. Without this knowledge we can't move further. The RDP auth is very complicated thing. So, I'll focus my attention only on scard-related things.

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

Good. Now you have a brief overview of the NLA. Now it's time to move forward.

## Kerberos

To log on using smart card we need the SPNEGO to select the Kerberos as a application protocol for authentication.
**Kerberos is the only authentication protocol that can be used for scard logon.**
In order to support password-less logon, Kerberos has the PKINIT extension: [Public Key Cryptography for Initial Authentication in Kerberos (PKINIT)](https://datatracker.ietf.org/doc/html/rfc4556).

// TODO

## Scard credentials

The CredSSP client transfers user's username, password, and domain to the target server during password-based logon.
In turn, the target server tries to login the user using this credentials.
But we cannot send the smart card to the target server :upside_down_face:. According to the specification, smart card credentials are defined as follows:

```asn1
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
- `userHint` - username. **Important**: the username must be **only** username. The target server may reject the FQDN. `t2@example.com` :x: -> `t2` :white_check_mark:.
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
  We cannot precalculate or extract it from somewhere. A small trick is used to generate this value (explained below).

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

However, that's not the end: each smart card minidriver communicates with the smart card through a unified API: [WinSCard API](https://learn.microsoft.com/en-us/windows/win32/api/winscard/). :arrow_down:

## WinSCard

> The basic parts of the smart card subsystem are based on PC/SC standards (see the specifications at [https://pcscworkgroup.com](https://pcscworkgroup.com)). ([src](https://learn.microsoft.com/en-us/windows/win32/secauthn/smart-card-authentication))

The WinSCard API enables communications with smart card hardware. It is the lowest API from the software perspective.

The WinSCard API is based on the PC/SC (Personal Computer/Smart Card) specification - a globally implemented standard for cards and readers ([https://www.smartcardbasics.com/pc-sc/](https://www.smartcardbasics.com/pc-sc/)).

> The advantage of PC/SC is that applications do not have to acknowledge the details corresponding to the smart card reader when communicating with the smart card. This application can function with any reader that complies with the PC/SC standard. ([src](https://www.cardlogix.com/glossary/pc-sc/)).

## Let's put it all together

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