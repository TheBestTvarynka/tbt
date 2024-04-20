+++
title = "Smart card container name"
date = 2024-05-04
draft = false

[taxonomies]
tags = ["debugging", "windows", "scard"]

[extra]
keywords = "Debugging, API, Windows, Smart Card, Reverse Engineering"
toc = true
+++

# Getting Started

In my previous article, I explained the different smart card cache items (files), their format, and purpose. The current one is some continuation of talking about PIV smart cards in Windows. The setup is the same: Windows smart card minidriver + PIV smartcard.

## Goals

* Explain how the Windows [minidriver](https://learn.microsoft.com/en-us/windows-hardware/drivers/smartcard/smart-card-minidriver-overview) treats the smart card container name.
* Tell how a bad container name can break scard auth.
* Fun :partying_face:.

## Non-goals

* Explain every piece of smart card architecture in Windows.
* Explain every smart card minidriver function.

# Overview

*Note.* If you want to debug with me and reproduce each step on your machine, follow the steps from the [Debugging Environment](#debugging-environment) section and then come back here. If you only want to read my findings and smart card container name structure, then jump to the [Conclusion](#conclusion) section.

Let's try to sign the data using the emulated smart card.

![](./failed-signing.png)

Expectedly (otherwise, this article would not exist :slightly_smiling_face:), we got an exception. Our goal for the rest of the article is to figure out the cause of the error and fix it.

# Debugging environment

My goal is to simulate the data signing using an emulated smart card. To achieve this I need two key things:

* Some program signs the data using the smart card.
* Hook somehow the `winscard.dll` to use an emulated smart card.

A brief explanation of our debugging environment:

1. Test VM + certificate with private key suitable for the data signing.
2. We'll have a small program written in C#. The only thing this program does is sign the hardcoded data using the smart card.
3. We'll have a `bad.dll` that hooks the original `winscard.dll` with our one.
4. And the last but not least thing is a *"launcher"*: another program that creates a signing process, then loads the `bad.dll` into it to hook the `winscard.dll`, and then continues the execution.

> *Pheww, it starts looking like some kind of Frankenstein* :zany_face:

Yes. You are right :upside_down_face:.

## Compile the data signer

The code is very simple. It uses some high level C# API for the data signing. The full code can be found [here](https://github.com/TheBestTvarynka/trash-code/tree/scard-container-name/scard-container-name/SignDataTmp). Just clone the project and build it with [Visual Studio](https://visualstudio.microsoft.com/vs/community/). *Note:* it required **net6.0**.

The overall algorithm is pretty simple: we create a CSP that matches our smart card and then sign the data using the RSA crypto provider. The simplified code looks like this:

```c#
// Create needed CSP.
CspParameters csp = new CspParameters(1,
    "Microsoft Base Smart Card Crypto Provider",
    containerName // smart card container name
);
csp.KeyPassword = pwd;

// Sign the data.
RSACryptoServiceProvider rsaCsp = new RSACryptoServiceProvider(csp);
byte[] signature = rsaCsp.SignData(dataToSign, HashAlgorithmName.SHA1, RSASignaturePadding.Pkcs1);
```

As a result, you should get `SignDataTmp.exe` + `SignDataTmp.dll`.

## Compile your own version of `winscard.dll`

```bash
git clone https://github.com/Devolutions/sspi-rs.git
cd sspi-rs/ffi
git checkout 4cd21349311b8edad03cc0dc8edf56407ccc22b9
cargo build --all-features
```

Of course, during my first debugging, I had a dev version of the `sspi-rs`. But now all developed functionality has been [merged](https://github.com/Devolutions/sspi-rs/pull/210), so we need to pick a specific commit. After that you will have `sspi.dll` in the `target/debug` directory. You can rename it to `winscard.dll` if you want.

## Compile `bad.dll`

This `bad.dll` represents how I do the hooking. There are many hooking techniques. My choice criteria were that it should work and not be hard to implement. So, I chose the [MinHook](https://github.com/TsudaKageyu/minhook) library. It is easy to use and does its job well. Project source code can be found [here](https://github.com/TheBestTvarynka/trash-code/tree/scard-container-name/scard-container-name/bad). Just clone the project and build it with [Visual Studio](https://visualstudio.microsoft.com/vs/community/). *Note:* do not forget to properly link it with `MinHook`.

But here is a surprise. When you try to read the code you'll find out that I hook some weird functions instead of the `winscard.dll`. The reason for that is [delayed loading](https://learn.microsoft.com/en-us/cpp/build/reference/understanding-the-helper-function). We can not simply hook the `winscard.dll` because the `basecsp.dll` loads it via delayed loading :confused:.

But it is still possible to hook. Thanks to [@awakecoding](https://x.com/awakecoding) I know how to do it. He had [researched](https://x.com/awakecoding/status/1725249330435518736) and implemented it in the [MsRdpEx](https://github.com/Devolutions/MsRdpEx): [fix winscard.dll delay-loading interception from basecsp.dll](https://github.com/Devolutions/MsRdpEx/pull/84). I reused some code and make it work for my debugging program :sweat_smile:.

So, as a result, you should get `bad.dll`.

## Compile the launcher

The *"launcher"* is named as *"HookLibrary"* due to my own historical reasons :zany_face:. The full source code can be found [here](https://github.com/TheBestTvarynka/trash-code/tree/scard-container-name/scard-container-name/HookLoadLibrary). *Important*: do not forget to change hardcoded paths to your ones. It works as follows:

1. Runs the data signer executable in a separate process, but does not run it.
2. Injects the `bad.dll` into the process memory forcing it to load this dll. In turn, the `bad.dll` will hook all needed methods.
3. Continues the execution.

So, as a result, you should get `HookLoadLibrary.exe`.

## Run it

Before trying to run this machinery we need to configure the emulated smart card. To do this set up the following environment variables:

| Name | Value | Example |
|------|-------|---------|
| `SSPI_LOG_PATH` | path to the log file | D:\test_data\sspi.log |
| `SSPI_LOG_LEVEL` | log level | trace |
| `WINSCARD_PIN` | smart card PIN code | 214653 |
| `WINSCARD_CERT_PATH` | path to the *.cer* file containing the smart card certificate | [D:\test_data\user.cer](https://github.com/TheBestTvarynka/trash-code/blob/scard-container-name/scard-container-name/t2%40tbt.com.cer) |
| `WINSCARD_PK_PATH` | path to the *.key* file containing the smart card certificate | [D:\test_data\user.key](https://github.com/TheBestTvarynka/trash-code/blob/scard-container-name/scard-container-name/t2%40tbt.com.key) |
| `WINSCARD_CONTAINER` | container name | 1b22c362-46ba-4889-ad5c-01f7abcabcabedw |
| `WINSCARD_READER` | reader name | `Microsoft Virtual Smart Card 2` |

And finally, run `HookLoadLibrary.exe`. You can see how I do it in the *Overview* section.

[Back to the Overview section](#overview).

# Conclusion

# Doc, references, code

* [Smart Card Minidriver Overview](https://learn.microsoft.com/en-us/windows-hardware/drivers/smartcard/smart-card-minidriver-overview).
* [Emulated smart cards implementation](https://github.com/Devolutions/sspi-rs/pull/210).
* [Source code of all used debugging tools](https://github.com/TheBestTvarynka/trash-code/tree/scard-container-name/scard-container-name).