+++
title = "Debugging with API Monitor"
date = 2023-07-28
draft = false
template = "post.html"

[taxonomies]
tags = ["debugging", "api-monitor", "rust", "windows"]

[extra]
keywords = "Rust, API Monitor, Debugging, API"
mermaid = true
toc = true
thumbnail = "debugging-with-api-monitor-thumbnail.png"
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

You can compare old and new XML and see the difference. Now all should work. Let's try again to record API calls. Here is my result:

{{ img(src="scardtransmit_with_buffer.png" alt="SCardTransmit with buffer") }}

{{ img(src="scardtransmit_with_out_buffer.png" alt="SCardTransmit with buffer") }}

Congratulations :hibiscus: . Now it works well and we can observe input and output buffers.

# What if we have a custom DLL to monitor?

Firstly, I was not planned to write this section. But if I touched fixing XML definitions then it's obviously fun to write definitions for our custom library and try to debug it.

But before actual debugging, let's write the program and dll itself. I suggest a simple task:

Look at this site: [https://imgur.com](https://imgur.com). *Imgur* is an American online image-sharing and image-hosting service. It has an [API](https://apidocs.imgur.com/). Let's write a library that encapsulates API calls and provides us with a simple interface for communication with Imgur. I plan to implement only two functions:

* `ImgurInitClient`: initializes the Imgur client (or context).
* `ImgurGetComment`: retrieves the comment information based on its id.

And in order to make a more "real" example, I'll pack Rust code into the dll and call its functions from the C++ code.

{% mermaiddiagram() %}
flowchart TD
    A[C++ code] --> dll[dll]

subgraph dll
    C[Rust FFI bindings] --> D[Rust Imgur API client]
end
{% end %}

## Write a program and dll

**Note:** this section contains a lot of code. There is no point to read it very carefully. You should just have a rough idea of what it does.

Good. We have a purpose but don't have any obstacles. Firstly, we need to write a pure Rust implementation. There is no point to explain it in detail, so I just paste the code below:

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

Good. And the last part is a C++ code that loads our dll and calls its functions. Before it, we need to generate the `.h` file with our structures and functions. Sure, with such a small example, we can do it manually. But I wanna show you an easier way: I'll use the [`cbindgen`](https://github.com/mozilla/cbindgen). Run the following command in the terminal from the `imgur-api-dll` crate location:

```bash
cbindgen --crate imgur-api-dll --output imgur_api.h
```

As a result, you'll get the `imgur_api.h` file ready for use in the C++ project. I'll need the types of functions, so I add them manually:

```c++
typedef void* (ImgurInitClientFn)(const char*, const char*);
typedef uint32_t (ImgurGetCommentFn)(void*, unsigned long long, FiiComment**);
```

The full `imgur_api.h` file: [@TheBestTvarynka/trash-code/@a333b128/debugging-with-api-monitor/imgur-api-dll/imgur_api.h](https://github.com/TheBestTvarynka/trash-code/blob/a333b128ac66a128a4a98c7fb503004812053cb8/debugging-with-api-monitor/imgur-api-dll/imgur_api.h).

Finally, we can write our C++ program. I'm not much of a C++ dev, so don't judge me, please.

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
} else {
    cout << "Error: " << status << endl;
}
```

**Note:** don't worry about credentials in the code. I've already deleted this application from my Imgur account. So everything is fine.

Read the full src code here: [@TheBestTvarynka/trash-code/@a333b128/debugging-with-api-monitor/TestImgurDll](https://github.com/TheBestTvarynka/trash-code/tree/a333b128ac66a128a4a98c7fb503004812053cb8/debugging-with-api-monitor/TestImgurDll). Basically, our program has three main parts:

* Initialization: here we obtain module handle and pointers of functions.
* Comment information retrieving.
* Print the result of the execution.

Pretty simple, I think. If you run this program, you should get smth like this:

{{ img(src="TestImgurApiDll-execution.png" alt="TestImgurApiDll execution") }}

Now we have a working program that uses custom external DLL. Perfect. The most interesting and fun part begins in the next section.

## Writing XML definitions

There are no official guides on how to write such XML definitions. I just explored existing XMLs in the `C:\Program Files\rohitab.com\API Monitor\API` directory and wrote my own for the `imgur_api.dll`. Here is my shitty XML:

```xml
<ApiMonitor>
    <Include Filename="Headers\windows.h.xml" />
    <Module Name="imgur_api.dll" CallingConvention="STDCALL" Category="Imgur">
        <Variable Name="FfiCommentData" Type="Struct">
            <Field Type="UINT64" Name="id" />
            <Field Type="const char*" Name="image_id" />
            <Field Type="const char*" Name="comment" />
            <Field Type="const char*" Name="author" />
            <Field Type="UINT64" Name="author_id" />
            <Field Type="char" Name="on_album" />
            <Field Type="const char*" Name="album_cover" />
            <Field Type="UINT32" Name="ups" />
            <Field Type="UINT32" Name="downs" />
            <Field Type="UINT32" Name="points" />
            <Field Type="UINT32" Name="datetime" />
            <Field Type="UINT64" Name="parent_id" />
            <Field Type="char" Name="deleted" />
            <Field Type="char" Name="is_voted" />
            <Field Type="char" Name="vote" />
            <Field Type="const char*" Name="platform" />
            <Field Type="char" Name="has_admin_badge" />
            <Field Type="UINT64*" Name="children" />
            <Field Type="UINT32" Name="children_len" />
        </Variable>
        <Variable Name="FfiComment" Type="Struct">
            <Field Type="FfiCommentData" Name="data" />
            <Field Type="UINT32" Name="status" />
            <Field Type="char" Name="success" />
        </Variable>
        <Variable Name="FfiComment*" Type="Pointer" Base="FfiComment" />
        <Variable Name="FfiComment**" Type="Pointer" Base="FfiComment*" />

        <Api Name="ImgurInitClient">
            <Param Type="const char*" Name="client_id" />
            <Param Type="const char*" Name="client_secret" />
            <Return Type="void*" />
        </Api>
        <Api Name="ImgurGetComment">
            <Param Type="void*" Name="context" />
            <Param Type="unsigned long" Name="comment_id" />
            <Param Type="FfiComment**" Name="comment" />
            <Return Type="UINT32" />
        </Api>
    </Module>
</ApiMonitor>
```

> *Why "shitty"?*

I don't like XML at all. So, for me, every XML is shitty.

The file with the src code: [@TheBestTvarynka/trash-code/@a333b128/debugging-with-api-monitor/imgur_api.xml](https://github.com/TheBestTvarynka/trash-code/blob/a333b128ac66a128a4a98c7fb503004812053cb8/debugging-with-api-monitor/imgur_api.xml). The code above looks pretty simple and easy to understand, but I'll give you some advice on how not to face problems:

* If the loaded definitions don't have all functions or don't have any at all, then they are probably invalid and you need to fix the XML. The API Monitor will not show you any message about what does wrong. For example, it'll not tell you that the function uses an unknown param type. It'll just ignore this function.
* You can split types and variables definitions into `.h.xml` and `.xml` files. The idea is obvious: you can include `.h.xml` files in other API definitions. In such a way you can reduce the code duplication.
* If you have the defined `MyStruct` structure, that does not mean that you automatically have the `MyStruct*` and `MyStruct**` pointer types. You should also define pointer types as I did in the code above for the `FfiComment**`.
* Why did I use the `char` instead of `BOOL`? The defined `BOOL` type has a 4-byte len but I need only one byte:

```xml
<!-- C:\Program Files\rohitab.com\API Monitor\API\Headers\common.h.xml -->
<Variable Name="BOOL" Type="Integer" Size="4">
    <Enum DefaultName="TRUE">
        <Set Name="TRUE"    Value="1" />
        <Set Name="FALSE"   Value="0" />
    </Enum>
    <Success Return="NotEqual" Value="0" />
</Variable>
```

So I decided just to use `char`. It's enough for us.

## Debugging

> *How to tell the API Monitor about our new API?*

Just place the file into the `API` directory. On the next API Monitor start it'll load all API definitions again, including our new one. For example:

```
C:\Program Files\rohitab.com\API Monitor\API\Imgur\imgur_api.xml
```

If you did everything right, then you should get smth like this:

{{ img(src="imgur_api_filter.png" alt="Imgur API filter") }}

Now let's debug the test application we wrote before. Start it as a usual process monitoring. Here is my result:

{{ img(src="monitoring_summary.png" alt="Monitoring summary") }}

Now we can fully observe what has been passed into and returned from our functions. Also, we compare the API Monitor values with those printed in the terminal to ensure that we did nothing wrong in the XML definitions.

{{ img(src="imgur_init_client.png" alt="ImgurInitClient function call") }}

On the screenshot above you can see secrets passed to the init function. Just like I saw passwords and emails during debugging the Windows SSPI :stuck_out_tongue_winking_eye:.

{{ img(src="imgur_get_comment.png" alt="ImgurGetComment function call") }}

Cool :sunglasses:. The values are the same as in the terminal.

# References & final note

1. [Trace APDU on Windows](https://www.mysmartlogon.com/knowledge-base/trace-apdu-on-windows/).
2. [Process memory editor, Unions and Arrays](http://www.rohitab.com/discuss/topic/37197-api-monitor-v2-r6-release-process-memory-editor-unions-and-arrays/).
3. [Rustonomicon - Foreign Function Interface](https://doc.rust-lang.org/nomicon/ffi.html).
4. [The (unofficial) Rust FFI Guide](https://michael-f-bryan.github.io/rust-ffi-guide/).

Sometimes our tools are capable of much more than we think. Maybe something you're searching for is just around the corner.