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

## Goals

The main purpose of this article is to tell you about [Windows SSPI](https://learn.microsoft.com/en-us/windows/win32/rpc/security-support-provider-interface-sspi-). If you know nothing about it and want to start working with it, this article will benefit you.

If you are familiar with SSPI and have worked with it, then this article most likely will not be useful.

**Goals**:

* Provide a comprehensive SSPI description.
* Introduce and explain basic SSPI terms.
* Understand core SSPI functions, invocation order, and the purpose of each of them.

**Non-goals**:

* Explain how concrete SSP packages work (like `NTLM`, `Kerberos`, etc).
* Teach how to write a custom SSP package. <sub>I mean, after reading this article you will have a brief idea of what you need to write but it's obviously not enough.</sub>

Happy reading! :smiley:

## What is SSPI

First of all, let's start with [the official SSPI definition](https://learn.microsoft.com/en-us/windows/win32/rpc/sspi-architectural-overview):

> SSPI is a software interface. Distributed programming libraries such as RPC can use it for authenticated communications.
> One or more software modules provide the actual authentication capabilities...

So, now we know that it's an interface. It can be used for authentication. The general idea is to provide a unified interface for the authentication and if any program will need to authenticate some user, then it "just" can use SSPI. Moreover, the authentication process (function call order, etc) doesn't depend on the underlying authentication protocol. But buffer sizes and types can be different. We'll see how we can handle such cases.

## Basic concepts and definitions

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

## SSPI

The SSPI functions table has a lot of <sub>(?terrible?)</sub> functions. We'll talk about them, and how and when you should use them.

## Doc, references, code

1. [Security Support Provider Interface (SSPI).](https://learn.microsoft.com/en-us/windows/win32/rpc/security-support-provider-interface-sspi-)
2. [SSPI.](https://learn.microsoft.com/en-us/windows/win32/secauthn/sspi)
3. [`sspi.h`.](https://learn.microsoft.com/en-us/windows/win32/api/sspi/)
