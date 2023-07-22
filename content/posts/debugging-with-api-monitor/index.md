+++
title = "Debugging with API Monitor"
date = 2023-07-25
draft = false

[taxonomies]
tags = ["debugging", "api-monitor", "rust"]

[extra]
keywords = "Rust, API Monitor, Debugging, API"
toc = true
+++

The [API monitor](http://www.rohitab.com/apimonitor) speaks for itself. It's a great tool for the system API calls debugging, monitoring, information extraction, exploring. Especially in cases when you don't know how exactly API works.

**Disclaimer**. (If you don't get it yet). Mentioned API Monitor is a tool for monitoring the system API calls. So, I you are back/front-end dude which is not interested in such stuff, this article will not be useful for you.

# Goals

**Goals:**

* Explain how to deal with cases when existing API definitions are not enough.
* Show on the example how we can debug custom DLLs and how to write definitions for them.

**Non-goals:**

* Write the *"ultimate"* guide to the `API Monitor`.
* Describe how the `API Monitor` works under the hood.

If you are not familiar with the API Monitor, then I propose you reading the following articles:

* [Monitoring your first application](http://www.rohitab.com/api-monitor-tutorial-monitoring-your-first-application).
* [API Monitor (youtube video)](https://youtu.be/IjIfu1y1EdQ).

The current article does not include API Monitor basics.

# Let's debug WinSCard API

> *So, what problem or edge cases do you want to talk about?*

I prefer example-based explanations, so we start from the API monitoring. And today's target is [WinSCard API](https://learn.microsoft.com/en-us/windows/win32/api/winscard/). In a short, it's a Microsoft's API for the communication with smart cards. I have a small program that uses smart card for some operations. Let's assume I don't know what exactly it's doing or know what but don't know how. Out task for now is to try to explore the API calls.

Start as usual:

1. Select only needed API.

{{ img(src="api-filter.png" alt="API filter") }}

2. Run the executable in the API Monitor. Here is what I got:

{{ img(src="api-recording.png" alt="API recording demo") }}

Cool. We see a lot of functions calls, their before and after parameters, flags, and much more. But here is one problem that blocks us from the further investigation: we don't have recorded in- and outbound APDU messages.

*Note.* The APDU message - is the communication unit between a smart card reader and a smart card. More info: [wikipedia](https://en.wikipedia.org/wiki/Smart_card_application_protocol_data_unit), [yubikey-reference/apdu](https://docs.yubico.com/yesdk/users-manual/yubikey-reference/apdu.html).

The [`ScardTransmit`](https://learn.microsoft.com/en-us/windows/win32/api/winscard/nf-winscard-scardtransmit) function has in- and outbound APDU messages and they defined in the function signature as follows:

```c++
LONG SCardTransmit(
  [in]                SCARDHANDLE         hCard,
  [in]                LPCSCARD_IO_REQUEST pioSendPci,
  [in]                LPCBYTE             pbSendBuffer, // inbound APDU message buffer
  [in]                DWORD               cbSendLength,
  [in, out, optional] LPSCARD_IO_REQUEST  pioRecvPci,
  [out]               LPBYTE              pbRecvBuffer, // outbound APDU message buffer
  [in, out]           LPDWORD             pcbRecvLength
);
```

From the function definition we can see that buffers length is defined in the `cbSendLength` and `pcbRecvLength` parameters. So, it's logical to assume that API Monitor will capture those buffers during the recording because lengths are known. But unfortunately, we have only pointer value recorded.

{{ img(src="scardtransmit.png" alt="SCardTransmit") }}

> *Why it's happening? The API Monitor captures buffers with defined (known) length as I remember from the documentation.*

The answer is in the next section.

# Time to fix XML definitions

Example how we can fix it, recording after.

# What if we have custom DLL to monitor

Write a simple program, dll (in Rust of course), XML definitions, show how it works.

# References & final note

1. [Trace APDU on Windows](https://www.mysmartlogon.com/knowledge-base/trace-apdu-on-windows/).
2. [Process memory editor, Unions and Arrays](http://www.rohitab.com/discuss/topic/37197-api-monitor-v2-r6-release-process-memory-editor-unions-and-arrays/).

Conclusions.