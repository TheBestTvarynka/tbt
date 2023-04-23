+++
title = "SSPI introduction"
date = 2023-04-11
draft = false

[taxonomies]
tags = ["sspi", "windows", "winapi", "rust"]

[extra]
keywords = "SSPI, Windows"
toc = true
+++

# Goals

The main purpose of this article is to tell you about [Windows SSPI](https://learn.microsoft.com/en-us/windows/win32/rpc/security-support-provider-interface-sspi-). If you know nothing about it and want to start working with it, this article will benefit you.

If you are familiar with SSPI and have worked with it, then this article most likely will not be useful.

**Goals**:

* Provide a comprehensive SSPI description.
* Introduce and explain basic SSPI terms.
* Understand core SSPI functions, invocation order, and the purpose of each of them.

**Non-goals**:

* Explain how concrete SSP packages work (like `NTLM`, `Kerberos`, etc).
* Teach how to write a custom SSP package. <sub>I mean, after reading this article you will have a brief idea of what you need to write but it's obviously not enough.</sub>

**Does this article replace reading the documentation?** Of course not. It's just a good place to start.

Happy reading! :smiley:

# What is SSPI

First of all, let's start with [the official SSPI definition](https://learn.microsoft.com/en-us/windows/win32/rpc/sspi-architectural-overview):

> SSPI is a software interface. Distributed programming libraries such as RPC can use it for authenticated communications.
> One or more software modules provide the actual authentication capabilities...

So, now we know that it's an interface. It can be used for authentication. The general idea is to provide a unified interface for the authentication and if any program will need to authenticate some user, then it "just" can use SSPI. Moreover, the authentication process (function call order, etc) doesn't depend on the underlying authentication protocol. But buffer sizes and types can be different. We'll see how we can handle such cases.

# Basic concepts and definitions

**SSP** - [Security Support Provider](https://learn.microsoft.com/en-us/windows/win32/secauthn/security-support-provider-authentication-packages) - a dynamic library (`.dll` file) that has one or more SSPI implementations. **Security package** - one SSPI implementation. Usually, it reflects some authorization protocol.

Good, now let's think a little bit. What do we mean under "successful authentication"? Usually, it's something like "username + password is correct, we can do our work further" or "we've got some token that proves our identity/permissions".

In SSPI, the process of authenticating is called context establishing. The result of authenticating is the (established) security context with the session key. The session key can be used for further encryption/decryption (you'll see it later), checksum calculation/verification, and so on. It's valid only for this session. After another context establishing the security context will have another session key. The Microsoft documentation gives us a good [**session context** definition](https://learn.microsoft.com/en-us/windows/win32/secauthn/sspi-context-semantics):

> A security context is the set of security attributes and rules in effect during a communication session.

*A communication session.* After successful authentication, the client and the server can continue to send messages to each other. They can use the session key to secure this communication. It means you can still need SSPI even after the authentication process.

Okay, sounds pretty simple and reasonable. Time to see the actual SSPI interface. All mandatory functions are listed in one structure: the SSPI function table. [Here it is](https://learn.microsoft.com/en-us/windows/win32/api/sspi/ns-sspi-securityfunctiontablew):

```C++
typedef struct _SECURITY_FUNCTION_TABLE_W {
  unsigned long                        dwVersion;
  ENUMERATE_SECURITY_PACKAGES_FN_W     EnumerateSecurityPackagesW;
  QUERY_CREDENTIALS_ATTRIBUTES_FN_W    QueryCredentialsAttributesW;
  ACQUIRE_CREDENTIALS_HANDLE_FN_W      AcquireCredentialsHandleW;
  FREE_CREDENTIALS_HANDLE_FN           FreeCredentialsHandle;
  void                                 *Reserved2;
  INITIALIZE_SECURITY_CONTEXT_FN_W     InitializeSecurityContextW;
  ACCEPT_SECURITY_CONTEXT_FN           AcceptSecurityContext;
  COMPLETE_AUTH_TOKEN_FN               CompleteAuthToken;
  DELETE_SECURITY_CONTEXT_FN           DeleteSecurityContext;
  APPLY_CONTROL_TOKEN_FN               ApplyControlToken;
  QUERY_CONTEXT_ATTRIBUTES_FN_W        QueryContextAttributesW;
  IMPERSONATE_SECURITY_CONTEXT_FN      ImpersonateSecurityContext;
  REVERT_SECURITY_CONTEXT_FN           RevertSecurityContext;
  MAKE_SIGNATURE_FN                    MakeSignature;
  VERIFY_SIGNATURE_FN                  VerifySignature;
  FREE_CONTEXT_BUFFER_FN               FreeContextBuffer;
  QUERY_SECURITY_PACKAGE_INFO_FN_W     QuerySecurityPackageInfoW;
  void                                 *Reserved3;
  void                                 *Reserved4;
  EXPORT_SECURITY_CONTEXT_FN           ExportSecurityContext;
  IMPORT_SECURITY_CONTEXT_FN_W         ImportSecurityContextW;
  ADD_CREDENTIALS_FN_W                 AddCredentialsW;
  void                                 *Reserved8;
  QUERY_SECURITY_CONTEXT_TOKEN_FN      QuerySecurityContextToken;
  ENCRYPT_MESSAGE_FN                   EncryptMessage;
  DECRYPT_MESSAGE_FN                   DecryptMessage;
  SET_CONTEXT_ATTRIBUTES_FN_W          SetContextAttributesW;
  SET_CREDENTIALS_ATTRIBUTES_FN_W      SetCredentialsAttributesW;
  CHANGE_PASSWORD_FN_W                 ChangeAccountPasswordW;
  void                                 *Reserved9;
  QUERY_CONTEXT_ATTRIBUTES_EX_FN_W     QueryContextAttributesExW;
  QUERY_CREDENTIALS_ATTRIBUTES_EX_FN_W QueryCredentialsAttributesExW;
} SecurityFunctionTableW, *PSecurityFunctionTableW;
```

Along with table *W*, also a similar [table A](https://learn.microsoft.com/en-us/windows/win32/api/sspi/ns-sspi-securityfunctiontablea) exists. What's the difference? This is not a part of this story but here is [the answer on SO](https://stackoverflow.com/a/7424550/9123725). Today we'll work only with the W table. But remember that all things that apply to the W table, are also applicable to the A table.

# SSPI

The SSPI functions table has a lot of <sub>(?terrible?)</sub> functions. We'll talk about them, and how and when you should use them.

## Overview

To bring at least some order, we will split SSPI functions into four categories. [The official documentation](https://learn.microsoft.com/en-us/windows/win32/secauthn/authentication-functions#sspi-functions) has a good description of it so we follow it:

1. **Package management.** *SSPI package management functions initiate a security package, enumerate available packages, and query the attributes of a security package.*
  * `EnumerateSecurityPackagesW`
  * `InitSecurityInterfaceW`
  * `QuerySecurityPackageInfoW`
2. **Credential management.** *Functions used to obtain credentials handle for the security context, set credentials attributes, and query information about credentials.*
  * `AcquireCredentialsHandleW`
  * `ExportSecurityContext`
  * `FreeCredentialsHandle`
  * `ImportSecurityContextW`
  * `QueryCredentialsAttributesW`
3. **Context management.** *Functions used to establish security context, query and set its attributes, etc.*
  * `AcceptSecurityContext`
  * `ApplyControlToken`
  * `CompleteAuthToken`
  * `DeleteSecurityContext`
  * `FreeContextBuffer`
  * `ImpersonateSecurityContext`
  * `InitializeSecurityContextW`
  * `QuerySecurityContextToken`
  * `QueryContextAttributesW`
  * `SetContextAttributesW`
  * `RevertSecurityContext`
4. **Message support.** *Those functions usually are called when the security context is established. You can use them to secure messages (encryption, hashing, etc).*
  * `EncryptMessage`
  * `DecryptMessage`
  * `MakeSignature`
  * `MakeSignature`

Enough talking. Let's see SSPI in action. From the example below you will understand functions call order, and their purpose.

## SSPI in action

Two things are needed to demonstrate SSPI in action:

1. Client.
2. Server.

Both of them should use the same security package to successfully authenticate. As the client, I took [the NTLM security package](https://learn.microsoft.com/en-us/windows/win32/secauthn/microsoft-ntlm) from [the Microsoft-provided SSP](https://learn.microsoft.com/en-us/windows/win32/secauthn/ssp-packages-provided-by-microsoft). To keep my code example clearer, as the server, I took this [sspi implementation in Rust](https://crates.io/crates/sspi). All my code is written in Rust. To call Windows API functions I use crates with bindings like [winapi](https://crates.io/crates/winapi), [windows-sys](https://crates.io/crates/windows-sys).

Why NTLM? Because it's simple. It's a good protocol to use as an example (but bad for the real world [[1]](https://www.calcomsoftware.com/ntlm-security-weaknesses/#relay) [[2]](https://www.securew2.com/blog/why-ntlm-authentication-is-vulnerable). If you can avoid it then avoid it :wolf:).

[Here](https://github.com/TheBestTvarynka/trash-code/tree/main/sspi-introduction) you can find the source code.

### Initialization

### Credentials

### Authentication

### Communication

### Clean up

# Conclusion

# Doc, references, code

1. [Security Support Provider Interface (SSPI).](https://learn.microsoft.com/en-us/windows/win32/rpc/security-support-provider-interface-sspi-)
2. [SSPI.](https://learn.microsoft.com/en-us/windows/win32/secauthn/sspi)
3. [`sspi.h`.](https://learn.microsoft.com/en-us/windows/win32/api/sspi/)
4. [Authentication Functions.](https://learn.microsoft.com/en-us/windows/win32/secauthn/authentication-functions#sspi-functions)
5. [Using SSPI.](https://learn.microsoft.com/en-us/windows/win32/secauthn/using-sspi)
