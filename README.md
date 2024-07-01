# JSON Terminal Escapes

ANSI escape sequences were developed and standardized over 45 years ago (ANSI X3.64 standard in 1979).  Computing has changed, and with it, we need a new, more expressive, and more extensible way of communicating with the terminal.

I'm proposing a new standard based on JSON, which is universally encodable and decodable, extensible, and human readable.

In order to be backward compatible with the existing terminal escape codes, we'll implement JSON terminal escapes with an unused OSC code (proposed 23198 and 23199).  23198 will be used for programs that want to send commands/data to the terminal, and 23199 will be used by the terminal to send data and responses back to program.

`ESC ']' ('23198' | '23199') ';' <num-bytes> ';' <json-payload> ST`

Here are some example commands:

```
// use OSC 23198 for commands sent from the program/shell to the terminal
ESC ']' '23198' ';' '49' ';' '{"command": "term:cursormove", "data": {"y": -2}}' ST
ESC ']' '23198' ';' '0' ';' '{"command": "term:setstyle", "data": {"color": 31, "bgcolor": "#aaaaaa", "bold": true}}' ST
ESC ']' '23198' ';' '0' ';' '{"command": "term:resetstyle"}' ST

// use OSC 23199 for commands being sent from the terminal
ESC ']' '23199' ';' '0' ';' '{"command": "event:mouseclick", "data": {"row": 10, "col": 20}}' ST
```

For terminal sequences produced by a program, setting "num-bytes" lets the consumer know how large of a buffer to allocate for parsing the command.  It will _not_ include the final ST byte, just the size of the JSON payload.  For commands produced by hand (or when it is inconvenient to know the full length of the JSON), the value '0' is allowed, which denotes a dynamic buffer size (parser should read until the final ST byte).

---

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



