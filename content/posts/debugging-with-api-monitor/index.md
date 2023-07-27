+++
title = "Debugging with API Monitor"
date = 2023-07-25
draft = false

[taxonomies]
tags = ["debugging", "api-monitor", "rust"]

[extra]
keywords = "Rust, API Monitor, Debugging, API"
mermaid = true
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

The [`ScardTransmit`](https://learn.microsoft.com/en-us/windows/win32/api/winscard/nf-winscard-scardtransmit) function has in- and outbound APDU messages and they defined in the function signature as follows :point_down::

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

To answer the previous question, lets ask a new one: how the API Monitor even parse and recognize the API functions, parameters, etc? Of course, here is no any magic, but XML definitions :pensive: . Basically, the API Monitor has XML file for every supported library with defined API in it.

> *So, maybe we can just edit existing XML for the WinSCard API and record the buffers?*

Exactly. Someone who wrote the XML definition for the WinSCard didn't put enough attention and wrote them somehow.

Initially, I read about fixing those XML definitions in [this](https://www.mysmartlogon.com/knowledge-base/trace-apdu-on-windows/) article. I was surprised that only very little devs know about it and how to use it.

Okay, enough talking, time to fix the code:

```xml
<!-- File: C:\Program Files\rohitab.com\API Monitor\API\Headers\scard.h.xml -->
<!-- [SCARD_DISPOSITION] -->
<Variable Name="[SCARD_DISPOSITION]" Type="Alias" Base="LONG">
  <Display Name="LONG" />
  <Enum>
    <Set Name="SCARD_LEAVE_CARD"              Value="0" />
    <Set Name="SCARD_RESET_CARD"              Value="1" />
    <Set Name="SCARD_UNPOWER_CARD"            Value="2" />
    <Set Name="SCARD_EJECT_CARD"              Value="3" />
  </Enum>
</Variable>
```

```xml
<!-- File: C:\Program Files\rohitab.com\API Monitor\API\Windows\WinSCard.xml -->
<Api Name="SCardDisconnect">
  <Param Type="SCARDHANDLE" Name="hCard" />
  <Param Type="[SCARD_DISPOSITION]" Name="dwDisposition" />
  <Return Type="[SCARD_ERROR]" />
</Api>
<Api Name="SCardEndTransaction">
  <Param Type="SCARDHANDLE" Name="hCard" />
  <Param Type="[SCARD_DISPOSITION]" Name="dwDisposition" />
  <Return Type="[SCARD_ERROR]" />
</Api>

<Api Name="SCardTransmit">
  <Param Type="SCARDHANDLE" Name="hCard" />
  <Param Type="LPCSCARD_IO_REQUEST" Name="pioSendPci"/>
  <Param Type="LPCBYTE" Name="pbSendBuffer" Count="cbSendLength" />
  <Param Type="DWORD" Name="cbSendLength" />
  <Param Type="LPSCARD_IO_REQUEST" Name="pioRecvPci" />
  <Param Type="LPBYTE" Name="pbRecvBuffer" PostCount="pcbRecvLength"/>
  <Param Type="LPDWORD" Name="pcbRecvLength" />
  <Return Type="[SCARD_ERROR]" />
</Api>
```

**Note.** You will have other paths if you installed the API Monitor in the non-default location.

You can compare old and new XML and see the difference. Now all should work. Let's try again to record the API calls. Here is my result:

{{ img(src="scardtransmit_with_buffer.png" alt="SCardTransmit with buffer") }}

{{ img(src="scardtransmit_with_out_buffer.png" alt="SCardTransmit with buffer") }}

Congratulations :hibiscus: . Now it works well and we can observe input and output buffers.

# What if we have a custom DLL to monitor?

Firstly, I were not planned to write this section. But if I touched fixing XML definitions then it's obviously be fun to write definitions for out custom library and try to debug it.

But before actual debugging, let's write the dll itself. I suggest a simple task:

Look at this site: [https://imgur.com](https://imgur.com). *Imgur* is an American online image sharing and image hosting service. It has an [API](https://apidocs.imgur.com/). Let's write a library that encapsulates API calls and provides us a simple interface for communication with Imgur. I plan to implement only two functions:

* `ImgurInitClient`: initializes the Imgur client (or context).
* `ImgurGetComment`: retrieves the comment information based on its id.

And in order to make more "real" example, I'll pack Rust code into the dll and call its functions from the C++ code.

{% mermaiddiagram() %}
flowchart TD
    A[C++ code] --> dll[dll]

subgraph dll
    C[Rust FFI bindings] --> D[Rust Imgur API client]
end
{% end %}

## Write a program and dll

**Note:** this section contains a lot of code. There is no point to read it very carefully. You should just have a rough idea what it does.

Good. We have a purpose but don't have any obstacles. Firstly, we need to write a pure Rust implementation. There is no point to explain it in details, so I just pase the code below:

```rust
/// The Imgur API client
pub struct ImgurApi {
    /// Imgur application client ID
    client_id: String,
    /// Imgur application client secret
    _client_secret: String,
}

impl ImgurApi {
    /// Initializes the Imgur API client
    pub fn init<I, S>(client_id: I, _client_secret: S) -> Self
    where
        String: From<I>,
        String: From<S>,
    {
        Self {
            client_id: client_id.into(),
            _client_secret: _client_secret.into(),
        }
    }

    /// Retrieves the comment information based on its id
    pub fn comment(&self, comment_id: u64) -> ImgurResult<Comment> {
        let raw_comment = Client::new()
            .get(format!("https://api.imgur.com/3/comment/{}", comment_id))
            .header(AUTH_HEADER, format!("Client-ID {}", self.client_id))
            .send()?
            .bytes()?;

        let comment: Comment = serde_json::from_slice(&raw_comment)?;

        Ok(comment)
    }
}
```

The full `imgur-api-client` code: [@TheBestTvarynka/trash-code/@c42f4d80/debugging-with-api-monitor/imgur-api-client](https://github.com/TheBestTvarynka/trash-code/tree/c42f4d809849e9475e0d3fb72afbc2d0d18b6732/debugging-with-api-monitor/imgur-api-client).

After that, the next step is to write a [FFI bindings](https://doc.rust-lang.org/nomicon/ffi.html). It's not hard because we have two small functions to export:

```rust
#[no_mangle]
/// Initializes the Imgur API client
/// We return the `*mut c_void` pointer because we don't want to expose the internal context structure to users
pub unsafe extern "C" fn ImgurInitClient(client_id: *const c_char, client_secret: *const c_char) -> *mut c_void {
    let client_id = CStr::from_ptr(client_id);
    let client_secret = CStr::from_ptr(client_secret);

    let context = ImgurApi::init(client_id.to_str().unwrap(), client_secret.to_str().unwrap());

    Box::into_raw(Box::new(context)) as *mut _
}

/// Retrieves the comment information based on its id
#[no_mangle]
pub unsafe extern "C" fn ImgurGetComment(context: *mut c_void, comment_id: c_ulonglong, comment: *mut *mut FiiComment) -> u32 {
    let context: Box<ImgurApi> = Box::from_raw(context as *mut _ );

    if let Ok(c) = context.comment(comment_id) {
        *comment = Box::into_raw(Box::new(c.into()));

        0
    } else {
        1
    }
}
```

And of course, the full `imgur-api-dll` code you can find here: [@TheBestTvarynka/trash-code/@c42f4d80/debugging-with-api-monitor/imgur-api-dll](https://github.com/TheBestTvarynka/trash-code/tree/c42f4d809849e9475e0d3fb72afbc2d0d18b6732/debugging-with-api-monitor/imgur-api-dll). When you build this crate, you'll find the `imgur_api.dll` file in the `target/debug/` directory.

Good. And the last part is a C++ code that loads our dll and calls it's functions. Before it we need to generate the `.h` file with out structures and functions. Sure, with such a small example, we can do it manually. But I wanna show you an easier way: I'll use the [`cbindgen`](https://github.com/mozilla/cbindgen). Run the following command in the terminal from the `imgur-api-dll` crate location:

```bash
cbindgen --crate imgur-api-dll --output imgur_api.h
```

As the result, you'll get the `imgur_api.h` file ready for using in the C++ project. I'll need the functions types, so I added them manually:

```c++
typedef void* (ImgurInitClientFn)(const char*, const char*);
typedef uint32_t (ImgurGetCommentFn)(void*, unsigned long long, FiiComment**);
```

The full `imgur_api.h` file: [@TheBestTvarynka/trash-code/@a333b128/debugging-with-api-monitor/imgur-api-dll/imgur_api.h](https://github.com/TheBestTvarynka/trash-code/blob/a333b128ac66a128a4a98c7fb503004812053cb8/debugging-with-api-monitor/imgur-api-dll/imgur_api.h).

Finally, we can write our C++ program. I'm not much a C++ dev, so don't judge me please.

```c++
// code listed below is simplified and some lines are omitted
// follow the link under this snippet to read the full src code
HMODULE imgur = LoadLibraryA("imgur_api.dll");
FARPROC ImgurInitClientAddress = GetProcAddress(imgur, "ImgurInitClient");
FARPROC ImgurGetCommentAddress = GetProcAddress(imgur, "ImgurGetComment");

const char* client_id = "3d8012c2f66acfb";
const char* client_secret = "708d779959d043dd6da2d158abaa022931f708a8";

void* context = ((ImgurInitClientFn*)ImgurInitClientAddress)(client_id, client_secret);

unsigned long long comment_id = 1911999579;
FiiComment* comment = nullptr;
uint32_t status = ((ImgurGetCommentFn*)ImgurGetCommentAddress)(context, comment_id, &comment);

if (!status) {
    cout << "Success! Comment data:\n";
    cout << "id: " << comment->data.id << endl;
    // some lines are omitted
}
else {
    cout << "Error: " << status << endl;
}
```

**Note:** don't worry about credentials in the code. I've already deleted this application from my Imgur account. So everything is fine.

Read the full src code here: [@TheBestTvarynka/trash-code/@a333b128/debugging-with-api-monitor/TestImgurDll](https://github.com/TheBestTvarynka/trash-code/tree/a333b128ac66a128a4a98c7fb503004812053cb8/debugging-with-api-monitor/TestImgurDll). Basically, our program has three main parts:

* Initialization: here we obtain module handle, functions pointers.
* Comment information retrieving.
* Print the result of the execution.

Pretty simple, I think. If you run this program, you should get smth like this:

{{ img(src="TestImgurApiDll-execution.png" alt="TestImgurApiDll execution") }}

Now we have working program that uses custom external DLL. Perfect. The most interesting and fun part begins in the next section.

## Writing XML definitions

## Debugging

## Fixing XML definitions and debugging

# References & final note

1. [Trace APDU on Windows](https://www.mysmartlogon.com/knowledge-base/trace-apdu-on-windows/).
2. [Process memory editor, Unions and Arrays](http://www.rohitab.com/discuss/topic/37197-api-monitor-v2-r6-release-process-memory-editor-unions-and-arrays/).
3. [Rustonomicon - Foreign Function Interface](https://doc.rust-lang.org/nomicon/ffi.html).
4. [The (unofficial) Rust FFI Guide](https://michael-f-bryan.github.io/rust-ffi-guide/).

Conclusions.