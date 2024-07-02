# JSON Terminal Escapes

ANSI escape sequences were developed and standardized over 45 years ago (ANSI X3.64 standard in 1979).  Computing has changed, and with it, we need a new, more expressive, and more extensible way of communicating with the terminal.

I'm proposing a new standard based on JSON, which is universally encodable and decodable, extensible, and human readable.  We plan on using this protocol to implement advanced (and non-standard) escapes for [Wave Terminal](https://github.com/wavetermdev/waveterm).

While I don't think it is possible (or desirable) to replace the existing ANSI escape sequences, I think using JSON escapes for _new_ terminal extensions going forward is a more sustainable approach.  This will allow both host programs, and terminal implementors a standard way to use or implement new terminal functionality without the need for writing new parsers or protocols that are specific to a specific extension.  It will also improve how terminal escapes are documented.  The existing VT100 ANSI escape codes are so old they defy our "modern" way of documenting APIs.  The information exists in various resources (Wikipedia, Github, source code, diagrams) and it is hard to create a single source of truth for what the full set of commands are, what all of the parameters mean, and what they should do.

By using a "modern" format like JSON, we can created typed payloads, and real documentation that will help make the terminal more accessible (and extensible) in the years to come.

## Protocol Overview

In order to be backward compatible with the existing terminal escape codes, we'll implement JSON terminal escapes with an unused OSC code (proposed 23198 and 23199).  23198 will be used for programs to send commands/data to the terminal, and 23199 will be used by the terminal to send events, data, and responses back to program.

`ESC ']' ('23198' | '23199') ';' <num-bytes> ';' <json-payload> BEL`

Here are some example commands:

```
// use OSC 23198 for commands sent from the program/shell to the terminal
ESC ']' '23198' ';' '49' ';' '{"command": "term:cursormove", "data": {"y": -2}}' BEL
ESC ']' '23198' ';' '0' ';' '{"command": "term:setstyle", "data": {"color": 31, "bgcolor": "#aaaaaa", "bold": true}}' BEL
ESC ']' '23198' ';' '0' ';' '{"command": "term:resetstyle"}' BEL

// use OSC 23199 for commands being sent from the terminal
ESC ']' '23199' ';' '0' ';' '{"command": "event:mouseclick", "data": {"row": 10, "col": 20}}' BEL
```

For terminal sequences produced by a program, setting "num-bytes" lets the consumer know how large of a buffer to allocate for parsing the command.  It will _not_ include the final ST byte, just the size of the JSON payload.  For commands produced by hand (or when it is inconvenient to know the full length of the JSON), the value '0' is allowed, which denotes a dynamic buffer size (parser should read until the final ST byte).

## JSON Command Format

JSON commands have the following fields defined which create a full RPC protocol, that supports streaming requests and responses, between any program and the terminal.  The payload for requests and responses is generic and can include any valid JSON values.  Once standardized, the actual payload for specific commands will be strongly typed.  

Protocol Fields:
* **command** -- (string) the command to run.  by convention, systems and subsystems will be separated by colons (':').  this field is required for the first packet of any command.
* **rpcid** -- (string) for commands which require a response, an rpcid may be given.  it must be unique within the session.  UUIDs are recommended.  required for commands that expect a response (otherwise optional).  if doing a streaming request, rpcid is required on each request packet.
* **resid** -- (string) for responses.  if "resid" is set, we know it is a response to a matching "rpcid".  required for all responses.
* **timeout** -- (number) for commands with an "rpcid" this is the timeout for the command (milliseconds).  optional, if not present, there may be some default timeout.
* **cont** -- (boolean) for either requests or responses.  setting "cont" to be true means that additional request packets (with the same rpcid) or additional response packets (with the same resid) are forthcoming.  these packets can have differing formats from the initial packet, and need not contain the same "command" field.  optional, if not present, assumed to be `false`.
* **error** -- (string) for responses, indicates that an error has occurred (should not be combined with "cont".  generally an error indicates an end of the stream).  optional, if not present, indicates no error.
* **datatype** -- (string).  optional type hint for the data.  this is never needed for initial command packets (as the initial command should be strongly typed).  we may need it as a hint for streaming requests or responses where the following packets might be part of a type union.  having this field in the protocol may help with efficiently decoding data (in some languages).  specifically for binary data the value "binary" is reserved (must be paired with data64).
* **data** -- (any) the payload for the command, or the payload for the response.
* **data64** -- (string) base64 encoded payload data.  When data64 is present, data should not be present.  We allow base64 encoding when the data might contain binary sequences that might be problematic for the terminal to read.  also when encoding binary data (like files or image data).  normally data64 should be base64 decoded and then JSON parsed.  if the "datatype" field is present, and set to "binary" then data64 is just to be base64 decoded (not JSON parsed) and passed as a byte-array (or language equivalent) to consumer.  

## Detecting Support / Capabilities

We need a way for a terminal program to know that the new JSON commands are allowed.  Once we know they are supported, we can implement a JSON command to return fine grained capabilities, or respond to capabilities queries via a specific JSON command.  One way, of course, is looking at the TERM_PROGRAM environment variable, but this limits us to a static set of programs that must be maintained.

For generic capability, I propose sending a custom response to the the standard `ESC '[' c` command with the special code 23198.

Here's a sample request for capabilities:
`ESC '[' c`

And here is a sample response, claiming VT100 compatibility _with_ JSON commands:
`ESC '[' '?' 1 ';' 23198 'c'`

Once basic JSON command support is known, we can issue a JSON command to query for more specific support.

I propose a command "detect" which returns a data packet which specifies what commands are supported, or not:

Detect payload response:
* **capabilties** -- (map) a map, whose keys are system/extension names, and values are maps representing different levels of support, implemented commands, and excluded commands  
	* **level** -- (number) if levels are defined for this extension, what level we support
	* **include** -- (array of strings) what commands are supported (support globbing with `*`)
	* **exclude** -- (array of strings) what commands are not excluded (support globbing with `*`)


## Command Standardization

I'm hoping to do a round of command standardization.  It would be great to get a "level 1" standard in place that supports the bulk of the standard vt100 commands, but in JSON format with well defined, and typed, data payloads.  This would allow almost all terminal emulators to participate in "level 1".

Extensions beyond the vt100 set, like image protocols, or other advanced features that terminals are considering can be implemented as extensions, or standardized into separate system namespaces with terminals choosing to implement or not implement the standard as they see fit.

## Library and Language Support

As part of Wave Terminal, I'm working on Go and TypeScript support for the parsing of these commands.  We can also submit a patch to xtermjs to support any commands that we standardize (as they should be compatible with the existing CSI/OSC commands that they already support).

There is also work to be done on creating SDKs for implementing the RPC client/servers in various languages.  The RPC protocol is fairly simple and mostly trivial to implement.  The complications mostly arise from what language specific API should be used for streaming requests and responses.  Because the initial set of commands (the standard VT100 ones) will not require any request/response streaming, that feature can also be left out of the initial parser implementations.  One other thing to note is that the RPC protocol is symmetric.  The client can send requests, and the server can also send requests.  This is by design and allows us to implement terminal events (like keyboard, mouse, or other types of events) using the same JSON protocol.

Lastly, we'll have to create a new "ncurses"-like library/SDK that allows host programs to take full advantage of the new capabilities and extensions.

## FAQ

**Why do we need a "data" attribute instead of having a flat JSON structure, especially for simple commands?**

Initially I did plan on flattening out data, so commands could avoid another level of JSON.  But, the overhead is quite small (9 bytes), and it greatly simplifies the typing (programmatic typing) of the payloads, and allows for future extensibility of the protocol without worrying about reserving keys or creating "namespaces" for fields, etc.  It also simplifies the parsing of the data packets when writing SDKs and a clean separation of concerns where the top level is dealt with by the RPC layer exclusively and the data is dealt with by the implementation.

**Why do we need two OSC escapes instead of sharing the same number for both direction?**

Maybe we don't.  Open to the idea of simplifying down to one escape sequence number.  My concern was that when input echoing is on, the escape code sometimes leaks into the output, and if we store the output and say, reprocess it, we want to make sure not to have the output trigger another command.  Maybe this isn't a real concern since programs should be using raw mode anyway.

**Why do we need an "RPC" protocol at all?**

There are already terminal commands that expect output.  Right now that output must be synchronous, which prevents us from issuing longer running commands (async), or mixing multiple commands and making sure we assign the output to the correct command.  Also for advanced commands like uploading or downloading an image or a file, we really need streaming support in both direction, with the ability to asynchronously match responses to requests.  Imagine streaming a `tail -f` through an advanced command.  That command will run forever giving output.  The RPC mechanism makes sure we can support commands like that with no additional custom infrastructure required for clients/servers beyond the implementation of the RPC service that this spec provides.

But, because the "rpcid" is optional, we also allow for efficient "fire-and-forget" commands (like most of the traditional vt100 stuff).  For moving the cursor, or changing a style we _don't_ want a response (both for overhead, and for cluttering up the output stream for clients who are not running the RPC stuff).  This gives us a nice mix of both worlds, where advanced clients implement the RPC protocol, but simple clients can just send commands, omit the rpcid and continue to use escape codes in the traditional way, except with more consistent syntax.

**Do we need "data64"?**

Maybe not.  Theoretically all of the unicode bytes that are allowed in a JSON string should not stop our OSC sequence.  So using just JSON string escapes we should be good until we hit the ST character.  But this made me nervous for binary files (especially for terminals which might not completely respect the spec).  Plus, JSON strings can be a bit weird for binary data, so having this out seemed safe.

**Should we terminate with ST or BEL?**

I like the idea of terminating with ST, but BEL should be more compatible everywhere so I believe it is the safer choice.

**What about well defined error codes?**

We could add an "errorcode" string field.  My thinking was to instead overload the "error" field with special syntax if you want to send an error code.  like "ECTIMEOUT: Request timed out".  If the error starts with "EC" + /[A-Z0-9]+/ then this is an error code and can be converted to specific errors in the clients and checked by the implementations.  People will feel different ways about encoding like this, but I believe this is simpler without adding an extra field.  This also lets simple clients not worry about supporting codes at first.

**Multimodal Support**

Using JSON for this API was also important for multimodal support.  Using JSON-lines (making sure the entire packet fits on one line), it is easy to send these commands to a special socket (without the OSC wrapper).  As JSON these commands can also be easily sent over websockets and HTTP requests as well.  In fact the same RPC client code can be used with different adapters for Terminal, Raw Socket, WebSocket, and HTTP requests (even named pipes, domain sockets, carrier pidgeon, etc.).  Wave Terminal even uses these internally within the backend to pass commands between different systems.

## Authors / Copyright

Copyright 2024

* Michael Sawka -- Command Line Inc (Wave Terminal)








