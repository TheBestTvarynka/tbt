+++
title = "FreeRDP smart card support. Lore 1"
date = 2024-08-30
draft = false

[taxonomies]
tags = ["rust", "freerdp", "scard", "linux", "debugging"]

[extra]
keywords = "Rust, FreeRDP, Smart Card, Debugging"
toc = true
mermaid = true
+++

# Intro

This big article is a detailed lore about how I added a smart card auth support (*scard supprt*) to [`FreeRDP`](https://github.com/FreeRDP/FreeRDP).

# Building FreeRDP

```bash
git clone https://github.com/FreeRDP/FreeRDP.git
cd FreeRDP/
sudo pacman -S cmake
sudo pacman -S ninja
sudo pacman -S sdl2_ttf
mkdir freerdp-build
mkdir freerdp-build/debug
cmake -GNinja -B freerdp-build -S ./ -DCMAKE_BUILD_TYPE=Debug -DCMAKE_SKIP_INSTALL_ALL_DEPENDENCY=ON -DWITH_SERVER=OFF -DWITH_SAMPLE=OFF -DWITH_PLATFORM_SERVER=OFF -DUSE_UNWIND=OFF -DWITH_SWSCALE=OFF -DWITH_FFMPEG=OFF -DWITH_WEBVIEW=OFF -DCMAKE_INSTALL_PREFIX=/home/pavlo-myroniuk/apriorit/FreeRDP/freerdp-build/debug
cmake --build freerdp-build
cmake --install freerdp-build
```

The full compilation guide is located here: [FreeRDP/wiki/Compilation](https://github.com/FreeRDP/FreeRDP/wiki/Compilation).

Now let's ensure that everything works well and FreeRDP can connect to the remote server:

```bash
./freerdp-build/debug/bin/xfreerdp -v 192.168.0.109:3389 -u test -p "test" -log-level TRACE -cert ignore
```

![](./successful_ntlm_test.png)

Good. From the picture above we can say that `NTLM` auth protocol works well and `FreeRDP` can establish the RDP connection.

# What does "scard support" mean?

In such a case, the "scard supoprt" mean two things:

* Ability to log on into the remote server using smart card (**required**).
* Redirect local smart card to the remote machine (*optional: can be implemented later*).

Currently, I'll consentrate my attention at the first point. I'll talk about auth and scards for the rest of this article.

# Scard auth overview

## NLA

Now we need to understand how scard authorization works. Without this knowladge we can't move further.
The RDP auth is very compilated thing. So, I'll focus my attention only on scard related things.

So, the first thing you need to know is [NLA](https://en.wikipedia.org/wiki/Remote_Desktop_Services#Network_Level_Authentication):

> *Network Level Authentication (NLA) is a feature of RDP Server or Remote Desktop Connection (RDP Client) that requires the connecting user to authenticate themselves before a session is established with the server.*

Sounds pretty compilated. The idea is to authenticate the user before the user session establishment. It reduces the risk of denial-of-service attacks. We say let's perform mutual (hopefully) server and client authentication and reject the connection if something goes wrong.

And yes, Microsoft has a bunch of protocols for NLA. We don't need to know all of them, but I provide you some explanations of needed ones. Look at the picture below. It shows the NLA auth structure and the most common protocols:

{% mermaiddiagram() %}
flowchart LR
    subgraph CREDSSP["CredSSP"]
        subgraph SPNEGO["SPNEGO"]
            subgraph ApplicationProtocol["Application Protocol (NTLM, Kerberos, etc)"]
            end
        end
    end
{% end %}

[CredSSP overview](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-cssp/e36b36f6-edf4-4df1-9905-9e53b7d7c7b7):

> The Credential Security Support Provider (CredSSP) Protocol enables an application to securely delegate a user's credentials from a client to a target server. For example, the Microsoft Terminal Server uses the CredSSP Protocol to securely delegate the user's password or smart card PIN from the client to the server to remotely log on the user and establish a terminal services session.

However, the CredSSP protocol is only responsible for credentials delegation. It doesn't authenticate clients or servers. An actual auth is performed by the inner application protocol like NTLM, or Kerberos (preferable).

But Microsoft would not be Microsoft if they did not add complications. The CredSSP uses [the SPNEGO framework](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-spng/b16309d8-4a93-4fa6-9ee2-7d84b2451c84): ([quote src](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-cssp/e36b36f6-edf4-4df1-9905-9e53b7d7c7b7))

> SPNEGO provides a framework for two parties that are engaged in authentication to select from a set of possible authentication mechanisms.
>
> ...The CredSSP Protocol uses SPNEGO to mutually authenticate the CredSSP client and CredSSP server. It then uses the encryption key that is established under SPNEGO to securely bind to the TLS session.

Let's summarize this. The SPNEGO selects (negotiates) an appropriate authentication protocol and performs an authentication using this protocol. As the result, we'll have an established security context with some encryption key. In turn, the CredSSP will use this key to encrypt the credentials and pass them to the target CredSSP server.

Good. Now you have a brief overview of the NLA. Now it's time to move forward.

## Kerberos

To log on using smart card we need the SPNEGO to select the Kerberos as a application protocol for authentication. The Kerberos is the only one authentication protocol that can be used for the scard auth. In order to support password-less log on, the Kerberos uses the PKINIT extension: [Public Key Cryptography for Initial Authentication in Kerberos (PKINIT)](https://datatracker.ietf.org/doc/html/rfc4556).

## Smart card usage

During the connection establishing, we need smart card for the following things:

* Cerificate and public key extraction.
* Data signing.

To perform successful authenticarion, the Kerberos needs to know the user certificate and be able to sign the data with corresponding private key. Of cource, the private key is not exportable. So, the Kerberos will use some API/module to pass the padded digest and get the signature back.

The most low-level API for accessing smart cards is WinSCard API. It is originally implemented in Windows ([winscard.h](https://learn.microsoft.com/en-us/windows/win32/api/winscard/)) but also has an open source implementation ([pcsc-lite](https://pcsclite.apdu.fr/api/group__API.html)).

# FreeRDP components

The theory is good, but how all these protocols and APIs are represented in the code and how they communicate between each other? The purpose of this section is to give you answers to these questions.

## SSPI

The [SSPI](https://learn.microsoft.com/en-us/windows/win32/secauthn/sspi) is a general interface for authentication in Windows. Many security packages implement it and provide us different authentication methods viz the samwe API. CredSSP, NTLM, Kerneros, Negoteate, SChannel, and more. ALl of them implement SSPI API. More about SSPI you can read in my another blog post: [SSPI introduction](https://tbt.qkation.com/posts/sspi-introduction/).

The SSPI API has become very popular and many programs support it. The same for the FreeRDP: [`SecurityFunctionTableW`](https://github.com/FreeRDP/FreeRDP/blob/c9aa349c52bb3749ac1d37fe16c9f548ee6c53e4/winpr/include/winpr/sspi.h#L1156-L1188). The FreeRDP performs the NLA auth by calling the SSPI API. The FreeRDP is very flexible:

* It has its own SSPI implementors: [`FreeRDP/winpr/libwinpr/sspi`](https://github.com/FreeRDP/FreeRDP/tree/c9aa349c52bb3749ac1d37fe16c9f548ee6c53e4/winpr/libwinpr/sspi).
* It also can call the Windows SSPI API instead of some implementor (in order to use all SSPI features Windows has).
* We can even pass our own SSPI implementor (security package) via the `sspi-module` flag.

The FreeRDP has its own CredSSP implementation ([`nla.c`](https://github.com/FreeRDP/FreeRDP/blob/c9aa349c52bb3749ac1d37fe16c9f548ee6c53e4/libfreerdp/core/nla.c) and [`credssp_auth.c`](https://github.com/FreeRDP/FreeRDP/blob/c9aa349c52bb3749ac1d37fe16c9f548ee6c53e4/libfreerdp/core/credssp_auth.c)) but uses the SSPI for authentication.


