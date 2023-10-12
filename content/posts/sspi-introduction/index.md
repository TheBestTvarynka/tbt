+++
title = "SSPI introduction"
description = "If you know nothing about [Windows SSPI](https://learn.microsoft.com/en-us/windows/win32/rpc/security-support-provider-interface-sspi-) and want to start working with it, this article will benefit you."
date = 2023-04-27
draft = false

[taxonomies]
tags = ["sspi", "windows", "winapi", "rust"]

[extra]
keywords = "SSPI, Windows, Rust, WinAPI"
toc = true
mermaid = true
thumbnail = "sspi-thumbnail.png"
+++

# Goals

The main purpose of this article is to tell you about [Windows SSPI](https://learn.microsoft.com/en-us/windows/win32/rpc/security-support-provider-interface-sspi-). If you know nothing about it and want to start working with it, this article will benefit you.

If you are familiar with SSPI and have worked with it, then this article most likely will not be useful.

**Goals**:

* Provide a comprehensive SSPI description.
* Introduce and explain basic SSPI terms.
* Understand core SSPI functions, invocation order, and the purpose of each of them.

**Non-goals**:

* Explain how **concrete** SSP packages work (like `NTLM`, `Kerberos`, etc).
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
  * `EnumerateSecurityPackagesW`: list all available security packages in the SSP.
  * `InitSecurityInterfaceW`: initialize security table.
  * `QuerySecurityPackageInfoW`: query some information about the security package.
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
  * `EncryptMessage`.
  * `DecryptMessage`.
  * `MakeSignature`.
  * `MakeSignature`.

Enough talking. Let's see SSPI in action. From the example below you will understand functions call order, and their purpose.

## SSPI in action

Two things are needed to demonstrate SSPI in action:

1. Client.
2. Server.

Both of them should use the same security package to successfully authenticate. As the client, I took [the NTLM security package](https://learn.microsoft.com/en-us/windows/win32/secauthn/microsoft-ntlm) from [the Microsoft-provided SSP](https://learn.microsoft.com/en-us/windows/win32/secauthn/ssp-packages-provided-by-microsoft). To keep my code example clearer, as the server, I took this [sspi implementation in Rust](https://crates.io/crates/sspi). All my code is written in Rust. To call Windows API functions I use crates with bindings like [winapi](https://crates.io/crates/winapi), [windows-sys](https://crates.io/crates/windows-sys). You should focus only on the client code and not on the server code.

Why NTLM? Because it's simple. It's a good protocol to use as an example (but bad for the real world [[1]](https://www.calcomsoftware.com/ntlm-security-weaknesses/#relay) [[2]](https://www.securew2.com/blog/why-ntlm-authentication-is-vulnerable). If you can avoid it then avoid it :wolf:).

[Here](https://github.com/TheBestTvarynka/trash-code/tree/main/sspi-introduction) you can find the source code.

### Initialization

Good so far. Let the journey begin! What do we need to start using SSPI? Correct! SSPI function table. How can we initialize it?  By calling the `InitSecurityInterfaceW` function. This function returns a structure (see it [above](#overview)) with function pointers. If we want to use Windows SSP then we can just use Win API. But if we decide to use some custom SSP, then we need to load the corresponding dll, find the `InitSecurityInterfaceW` function and call it. Example:

```Rust
// load our SSP
let sspi_handle = LoadLibraryW("my_cool_ssp.dll");
// get the pointer to the needed function
let init_security_interface_fn = GetProcAddress(sspi_handle, "InitSecurityInterfaceW");
// transmute (cast) the pointer to the needed function type
let init_security_interface_w_fn: INIT_SECURITY_INTERFACE_W = unsafe { std::mem::transmute(init_security_interface_w_fn) };
let sspi_function_table = init_security_interface_w_fn();
// now we have initialized function table
// ...further authentication
```

But in the scope of this article, I use Windows native SSP. So I don't need to initialize the function table. It already is initialized inside Windows. Time to see what Windows can offer us:

```Rust
let mut number_of_packages = 0;
let mut packages = null_mut();

// Query information about the available Windows security packages
// https://learn.microsoft.com/en-us/windows/win32/api/sspi/nf-sspi-enumeratesecuritypackagesw
let result = EnumerateSecurityPackagesW(&mut number_of_packages, &mut packages);
```

When I printed it on the screen ([src](https://github.com/TheBestTvarynka/trash-code/blob/main/sspi-introduction/src/initialization.rs#L11)), I got such the output:

```txt
Number of packages: 13
-------------------------
fCapabilities: 8928179
wVersion: 1
wRPCID: 9
cbMaxToken: 48256
Name: Negotiate
Comment: Microsoft Package Negotiator
-------------------------
...
```

We can see a lot of security packages. But as I said above, today we are working only with NTLM. Let's gather more information about the NTLM security package ([src](https://github.com/TheBestTvarynka/trash-code/blob/main/sspi-introduction/src/initialization.rs#L54)):

```Rust
let mut raw_package_name = str_to_win_wstring("NTLM");
let mut package_info = null_mut();

let status = QuerySecurityPackageInfoW(raw_package_name.as_mut_ptr(), &mut package_info);

println!("{} package info:", package_name);
println!("fCapabilities: {}", (*package_info).fCapabilities);
println!("wVersion: {}", (*package_info).wVersion);
println!("wRPCID: {}", (*package_info).wRPCID);
println!("cbMaxToken: {}", (*package_info).cbMaxToken);
println!("Name: {}", c_wide_string_to_rs_string((*package_info).Name));
println!(
    "Comment: {}",
    c_wide_string_to_rs_string((*package_info).Comment)
);
```

Output:

```txt
NTLM package info:
fCapabilities: 42478391
wVersion: 1
wRPCID: 10
cbMaxToken: 2888
Name: NTLM
Comment: NTLM Security Package
```

In general, different security packages support different attributes. So refer to the documentation for more concrete information.

### Credentials

Before starting the actual authentication we need to prepare credentials. In other words, to acquire credentials handle. Basically, the credentials handle is a pointer to some structure that contains prepared credentials for use during the authentication, maybe some flags, and other credentials-related information. What type of this structure? We don't know and we don't need to know. The security provider creates and works with this object. It'll be good if you also read [the official credentials handle definition](https://learn.microsoft.com/en-us/windows/win32/secauthn/acquirecredentialshandle--general). Enough talking. Now time back to the code ([src](https://github.com/TheBestTvarynka/trash-code/blob/main/sspi-introduction/src/credentials.rs#L12)).

```Rust
let mut credentials_handle = CredHandle::default();
let mut expiry = TimeStamp::default();
let mut package_name = str_to_win_wstring("NTLM");

let mut domain = str_to_win_wstring("");
let mut user = str_to_win_wstring("testuser");
let mut password = str_to_win_wstring("test");

let mut identity = SEC_WINNT_AUTH_IDENTITY_W {
    User: user.as_mut_ptr(),
    UserLength: 8,
    Domain: domain.as_mut_ptr(),
    DomainLength: 0,
    Password: password.as_mut_ptr(),
    PasswordLength: 4,
    Flags: 0x2,
};

let status = AcquireCredentialsHandleW(
    null_mut(),
    package_name.as_mut_ptr(),
    // SECPKG_CRED_OUTBOUND: https://learn.microsoft.com/en-us/windows/win32/secauthn/acquirecredentialshandle--ntlm
    2,
    null_mut(),
    &mut identity as *mut SEC_WINNT_AUTH_IDENTITY_W as *mut _,
    None,
    null_mut(),
    &mut credentials_handle as *mut _,
    &mut expiry as *mut _,
);
```

After the `AcquireCredentialsHandleW` function call, the `credentials_handle` variable will contain the credentials handle. **Note.** In SSPI all handles (context, credentials, etc) are instances of the [`SecHandle`](https://learn.microsoft.com/en-us/windows/win32/api/sspi/ns-sspi-sechandle) structure. Each such structure has two numbers (pointers): *lower* and *upper*. When you are working with SSPI you don't need to know anything more about them. If you are writing your own SSP then you shouldn't expose any information about them.

However, to satisfy your curiosity, I will say what there may be (but don't tell anyone :stuck_out_tongue_winking_eye:): One of them can contain a pointer to some object (for example, a security context object), and another one can contain a pointer to the security package name. Because SSP can contain a lot of security packages, we need to store the security package name in some way that is currently used.

This function has a lot of parameters and they are well explained in [the official documentation](https://learn.microsoft.com/en-us/windows/win32/secauthn/acquirecredentialshandle--general). Just don't forget that the type for the `pauthdata` pointer is package-specific. It means that different security packages usually require different types of `pauthdata`.

Also, the SSPI has a place for customization. Security packages implement the `QueryCredentialsAttributesW` and `SetCredentialsAttributesW` functions. With them, we can add and get additional information (attributes) about concrete credentials handle. Small example ([src](https://github.com/TheBestTvarynka/trash-code/blob/main/sspi-introduction/src/credentials.rs#L74)):

```Rust
let mut credentials_name = SecPkgCredentials_NamesW::default();
let status = QueryCredentialsAttributesW(
    client_credentials_handle,
    SECPKG_CRED_ATTR_NAMES,
    &mut credentials_name as *mut SecPkgCredentials_NamesW as *mut _,
);

println!(
    "Credentials name: {:?}",
    c_wide_string_to_rs_string(credentials_name.sUserName)
);
```

Output:

```txt
Credentials name: "testdomain\\testuser"
```

### Authentication

Perfect. We have prepared credentials (credentials handle). Time to start actual authentication. The authentication process consists of sequential `InitializeSecurityContext` function calls.

> *When we should stop?*

When this function returns `SEC_E_OK`, `SEC_I_COMPLETE_AND_CONTINUE`, or `SEC_I_COMPLETE_NEEDED` status code. i. e. we call the `InitializeSecurityContext` function until it returns the appropriate status code. The generalized authentication flow is shown in the diagram below:

{% mermaiddiagram() %}
flowchart TD
    A(Begin auth) --> B(InitializeSecurityContext)
    B --> C{status code}
    C -->|Error| F(Auth failed)
    C -->|SEC_E_OK,...| E(Auth succeeded)
    C -->|SEC_I_CONTINUE_NEEDED| D(Send buffers to the server)
    D -->|TCP| G(Receive buffers from server)
    G --> B
{% end %}

Great, pretty simple. Now let's implement it in the code ([src](https://github.com/TheBestTvarynka/trash-code/blob/main/sspi-introduction/src/authentication.rs#L24)):

```Rust
let client_status = InitializeSecurityContextW(
    &mut client_credentials_handle,
    unwrap_sec_handle(&mut client_security_context),
    target_name.as_mut_ptr(),
    // MUTUAL_AUTH and ALLOCATE_MEMORY
    0x2 | 0x100,
    0,
    // Native data representation:
    0x10,
    &mut client_input_buffers,
    0,
    &mut new_client_security_context,
    &mut client_output_buffers,
    &mut context_attributes as *mut i32 as *mut _,
    &mut expiry as *mut _,
);
```

> *Oh my gosh. So many parameters. We definitely need an explanation.*

Agreed. But before I explain each of them, I would like to insert small note about the source code you will find in the repo: I didn't create any real servers or TCP/TLS connections. I just convert and pass buffers from one function to another. You will see it in the code comments.

Hardest part described below:

* `phcredential`: a pointer to the credentials handle. We have it after the `AcquireCredentialsHandleW` function call.
* `phcontext`: a pointer to the security context handle.
> *But wait. We don't have a such one.*

Yes. On the first function call we pass the `NULL` here. The security context handle will be created by the function itself during the first invocation and written in the `phnewcontext` parameter.
* `psztargetname`: *"a pointer to a null-terminated string that indicates the service principal name (SPN)"* ([docs](https://learn.microsoft.com/en-us/windows/win32/secauthn/initializesecuritycontext--ntlm)).
* `fcontextreq`: a context requirements flags. Basically, this flags tell the security package how to do authentication. Like, what session key to use, what additional things to require, etc. Those flags differ in the different security packages.
* `reserved1`, `reserved2` are reserved and not used.
* `targetdatarep`: *"The data representation, such as byte ordering, on the target."* ([docs](https://learn.microsoft.com/en-us/windows/win32/secauthn/initializesecuritycontext--ntlm))
* `pinput`: input buffers for this function. On the first function call, we don't have any input buffers so we should pass empty `SecBufferDesc`.
> *What types of security buffers should I choose, how many of them do I need, and in what sequence?*

Docs. Search for it in the documentation of the corresponding security package. If you can't find it there, then search in the open-source projects. And remember one important thing: **never change buffers' order or type**.
* `phnewcontext`: a new security context handle will be written here. You should use this handle for the next function call.
* `poutput`: same as `pinput` but output buffers instead of input.
* `pfcontextattr`: flags that describe established security context. We use `fcontextreq` to specify some options for authentication. Ans use `pfcontextattr` to see what options have been established (set).
* `ptsexpiry`: the expiration time of the context.

The security context also has functions for setting and getting different attributes: `QueryContextAttributesW` and `SetContextAttributesW`. They have similar behavior to the credentials-related functions. For example, in the NTLM security package, we have the possibility to extract the established session key ([src](https://github.com/TheBestTvarynka/trash-code/blob/main/sspi-introduction/src/authentication.rs#L155)):

```Rust
let mut session_key = SecPkgContext_SessionKey::default();

let status = QueryContextAttributesW(
    client_security_context,
    SECPKG_ATTR_SESSION_KEY,
    &mut session_key as *mut SecPkgContext_SessionKey as *mut _,
);

println!(
    "Established session key: {:?}",
    from_raw_parts(
        session_key.SessionKey as *const u8,
        session_key.SessionKeyLength as usize
    )
);
```

Output:

```txt
Established session key: [96, 1, 13, 58, 82, 191, 222, 134, 149, 184, 3, 75, 254, 126, 225, 74]
```

You can run the code a few times. Each time you will have another session key.

Phew, the hardest part is gone. Now we (*finally*) have the established security context. What's next? Now we can do everything we want, but more importantly, we can safely transfer any messages to the server and receive server messages securely.

### Communication

Imagine the situation: we want to send a very secret message to the server. In such case we should use the `EncryptMessage` function:

```Rust
let status = EncryptMessage(client_security_context, 0, &mut message, sequence_number);
// send `message` to the server
```

I think you already guessed it, but still: encryption/decryption, and signature generation/verification are always in-place. This is why we don't have the output buffer parameter in this function.

If we want to read encrypted messages (e. g. received from the server) then:

```Rust
let status = DecryptMessage(client_security_context, 0, &mut message, sequence_number);
```

The same thing with the `MakeSignature` and `VerifySignature` functions.

### Clean up

Yes, *memory leaks are memory safe*, but it's better to not forget to clean up everything. SSPI has three functions for that:

* `FreeContextBuffer`. ([doc](https://learn.microsoft.com/en-us/windows/win32/api/sspi/nf-sspi-freecontextbuffer)). Only one rule: if the buffer was allocated by the security package, then you should free it using this function. If the buffer was allocated by yourself, then you should free it in the usual way.
* `FreeCredentialsHandle`. ([doc](https://learn.microsoft.com/en-us/windows/win32/api/sspi/nf-sspi-freecredentialshandle)).
* `DeleteSecurityContext`. ([doc](https://learn.microsoft.com/en-us/windows/win32/api/sspi/nf-sspi-deletesecuritycontext)).

```Rust
// ...
let status = FreeCredentialsHandle(client_cred_handle);
let status = DeleteSecurityContext(client_context_handle);
```

# Conclusion

Finally, this story goes to the end. If you want to use SSPI on the server side then just replace the `InitializeSecurityContextW` function with `AcceptSecurityContext`.

> *Wait! You haven't described other SSPI functions!*

Don't panic, I know it. The other functions are rarely used and do not take a direct part in the authorization process.

I hope you enjoy reading it and it'll help you in some way. If you want to share some feedback, ask a question, just task to me, then use [this page](https://tbt.qkation.com/about/) to contact me.

# Doc, references, code

1. [Security Support Provider Interface (SSPI).](https://learn.microsoft.com/en-us/windows/win32/rpc/security-support-provider-interface-sspi-)
2. [SSPI.](https://learn.microsoft.com/en-us/windows/win32/secauthn/sspi)
3. [`sspi.h`.](https://learn.microsoft.com/en-us/windows/win32/api/sspi/)
4. [Authentication Functions.](https://learn.microsoft.com/en-us/windows/win32/secauthn/authentication-functions#sspi-functions)
5. [Using SSPI.](https://learn.microsoft.com/en-us/windows/win32/secauthn/using-sspi)
6. [Example source code.](https://github.com/TheBestTvarynka/trash-code/tree/main/sspi-introduction)

