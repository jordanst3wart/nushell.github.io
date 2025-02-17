---
title: Plugin protocol reference
---

# Plugin protocol reference

## How Nu runs plugins

Nu plugins **must** be an executable file with a filename starting with `nu_plugin_`. All interaction with the plugin is handled over standard input (stdin) and output (stdout). Standard error (stderr) is not redirected, and can be used by the plugin to print messages directly.

Plugins are always passed `--stdio` as a command line argument. Other command line arguments are reserved for options that might be added in the future, including other communication methods. Plugins that support the protocol as described in this document **should** reject other arguments and print an informational message to stderr.

Immediately after spawning a plugin, Nu expects the plugin to send its encoding type. Currently two encoding types are supported: [`json`](#json) and [`msgpack`](#messagepack). The desired encoding type **should** be sent first with the length of the string as a single byte integer and then the encoding type string. That is, with C-like escape syntax, `"\x04json"` or `"\x07msgpack"`. In this document, the JSON format will be used for readability, but the MessagePack format is largely equivalent. See the [Encoding](#encoding) section for specific intricacies of the formats.

Nu will then send messages in the desired encoding. The first message is always [`Hello`](#hello). The plugin **must** send a `Hello` message indicating the expected Nu version that it is compatible with, and any supported protocol features. The engine will also send a `Hello` message with its version, and any supported protocol features. The plugin **may** verify that it is compatible with the Nu version provided by the engine, but the engine will end communication with a plugin if it is determined to be unsupported. The plugin **must not** use protocol features it supports if they are not also confirmed to be supported by the engine in its `Hello` message. It is not permitted to send any other messages before sending `Hello`.

The plugin **should** then receive and respond to messages until its stdin is closed.

Typical plugin interaction after the initial handshake looks like this:

1. The engine sends a [`Call`](#call). The call contains an ID used to identify the response.
2. If the `input` of the call specified a stream, the engine will send [stream messages](#stream-messages). These do not need to be consumed before the plugin sends its response.
3. The plugin sends a [`CallResponse`](#callresponse), with the same ID from step 1.
4. If the plugin specified stream data as output in the response, it **should** now send [stream messages](#stream-messages) with the corresponding stream ID(s).

The plugin **should** respond to further plugin calls. The engine **may** send additional plugin calls before responses have been received, and it is up to the plugin to decide whether to handle each call immediately as it is received, or to process only one at a time and hold on to them for later. In any case, sending another plugin call before a response has been received **should not** cause an error.

The engine **may** send a [`Goodbye`](#goodbye) message to the plugin indicating that it will no longer send any more plugin calls. Upon receiving this message, the plugin **may** choose not to accept any more plugin calls, and **should** exit after any in-progress plugin calls have finished.

## `Hello`

After the encoding type has been decided, both the engine and plugin **must** send a `Hello` message containing relevant version and protocol support information.

| Field        | Type   | Usage                                                                                 |
| ------------ | ------ | ------------------------------------------------------------------------------------- |
| **protocol** | string | **Must** be `"nu-plugin"`.                                                            |
| **version**  | string | The engine's version, or the target version of Nu that the plugin supports.           |
| **features** | array  | Protocol features supported by the plugin. Unrecognized elements **must** be ignored. |

To be accepted, the `version` specified **must** be [semver](https://semver.org) compatible with the engine's version. "0.x.y" and "x.y.z" for differing values of "x" are considered to be incompatible.

There are currently no protocol features defined, and they are only likely to be used once Nu releases versions after stabilization at "1.0.0".

Plugins **may** decide to refuse engine versions with more strict criteria than specified here.

Example:

```json
{
  "Hello": {
    "protocol": "nu-plugin",
    "version": "0.90.2",
    "features": []
  }
}
```

## Input messages

These are messages sent from the engine to the plugin. [`Hello`](#hello) and [`Stream messages`](#stream-messages) are also included.

### `Call`

The body of this message is a 2-tuple (array): (`id`, `call`). The engine sends unique IDs for each plugin call it makes. The ID is needed to send the [`CallResponse`](#callresponse).

#### `Signature` plugin call

Ask the plugin to send its command signatures. Takes no arguments. Returns [`Signature`](#signature-plugin-call-response) or [`Error`](#error-plugin-call-response)

Example:

```json
{
  "Call": [0, "Signature"]
}
```

#### `Run` plugin call

Tell the plugin to run a command. The argument is the following map:

| Field      | Type                                        | Usage                                                 |
| ---------- | ------------------------------------------- | ----------------------------------------------------- |
| **name**   | string                                      | The name of the command to run                        |
| **call**   | [`EvaluatedCall`](#evaluatedcall)           | Information about the invocation, including arguments |
| **input**  | [`PipelineDataHeader`](#pipelinedataheader) | Pipeline input to the command                         |
| **config** | [`Value`](#value) or null                   | Plugin configuration, if available                    |

<a name="evaluatedcall"></a>

`EvaluatedCall` is a map:

| Field          | Type                                              | Usage                                                   |
| -------------- | ------------------------------------------------- | ------------------------------------------------------- |
| **head**       | [`Span`](#span)                                   | The position of the beginning of the command execution. |
| **positional** | [`Value`](#value) array                           | Positional arguments.                                   |
| **named**      | 2-tuple (string, [`Value`](#value) or null) array | Named arguments, such as switches.                      |

Named arguments are always sent by their long name, never their short name.

Returns [`PipelineData`](#pipelinedata-plugin-call-response) or [`Error`](#error-plugin-call-response).

Example:

```json
{
  "Call": [
    0,
    {
      "Run": {
        "name": "inc",
        "call": {
          "head": {
            "start": 40400,
            "end": 40403
          },
          "positional": [
            {
              "String": {
                "val": "0.1.2",
                "span": {
                  "start": 40407,
                  "end": 40415
                }
              }
            }
          ],
          "named": [
            [
              "major",
              {
                "Bool": {
                  "val": true,
                  "span": {
                    "start": 40404,
                    "end": 40406
                  }
                }
              }
            ]
          ]
        }
      }
    }
  ]
}
```

#### `CustomValueOp` plugin call

Perform an operation on a custom value received from the plugin. The argument is a 2-tuple (array): (`custom_value`, `op`).

The custom value is specified in spanned format, and not as a `Value` - see the example.

There is currently one supported operation: `ToBaseValue`, which **should** return a plain value that is representative of the custom value, or an error if this is not possible. Sending a custom value back for this operation is not allowed. Returns [`PipelineData`](#pipelinedata-plugin-call-response) or [`Error`](#error-plugin-call-response).

```json
{
  "Call": [
    0,
    {
      "CustomValueOp": [
        {
          "item": {
            "name": "version",
            "data": [0, 0, 0, 0, 1, 0, 0, 0, 2, 0, 0, 0]
          },
          "span": {
            "start": 90,
            "end": 96
          }
        },
        "ToBaseValue"
      ]
    }
  ]
}
```

### `Goodbye`

Indicate that no further plugin calls are expected, and that the plugin **should** exit as soon as it is finished processing any in-progress plugin calls.

This message is not a map, it is just a bare string, as it takes no arguments.

Example:

```json
"Goodbye"
```

## Output messages

These are messages sent from the plugin to the engine. [`Hello`](#hello) and [`Stream messages`](#stream-messages) are also included.

### `CallResponse`

#### `Error` plugin call response

An error occurred while attempting to fulfill the request.

| Field     | Type                    | Usage                                                                                |
| --------- | ----------------------- | ------------------------------------------------------------------------------------ |
| **label** | string                  | The main error message to show at the top of the error.                              |
| **msg**   | string                  | Additional context for the error. If `span` is provided, `msg` points at the `span`. |
| **span**  | [`Span`](#span) or null | The span that points to what source code caused the error.                           |

Example:

```json
{
  "CallResponse": [
    0,
    {
      "Error": {
        "label": "A really bad error occurred",
        "msg": "I don't know, but it's over nine thousand!",
        "span": {
          "start": 9001,
          "end": 9007
        }
      }
    }
  ]
}
```

### `Signature` plugin call response

A successful response to a [`Signature` plugin call](#signature-plugin-call). The body is an array of [signatures](https://docs.rs/nu-protocol/0.90.1/nu_protocol/struct.PluginSignature.html).

Example:

```json
{
  "CallResponse": [
    0,
    {
      "Signature": [
        {
          "sig": {
            "name": "len",
            "usage": "calculates the length of its input",
            "extra_usage": "",
            "search_terms": [],
            "required_positional": [],
            "optional_positional": [],
            "rest_positional": null,
            "vectorizes_over_list": false,
            "named": [
              {
                "long": "help",
                "short": "h",
                "arg": null,
                "required": false,
                "desc": "Display the help message for this command",
                "var_id": null,
                "default_value": null
              }
            ],
            "input_type": "String",
            "output_type": "Int",
            "input_output_types": [],
            "allow_variants_without_examples": false,
            "is_filter": false,
            "creates_scope": false,
            "allows_unknown_args": false,
            "category": "Default"
          },
          "examples": []
        }
      ]
    }
  ]
}
```

### `PipelineData` plugin call response

A successful result with a Nu [`Value`](#value) or stream. The body is a map: see [`PipelineDataHeader`](#pipelinedataheader).

Example:

```json
{
  "CallResponse": [
    0,
    {
      "Value": {
        "Int": {
          "val": 42,
          "span": {
            "start": 12,
            "end": 14
          }
        }
      }
    }
  ]
}
```

## Stream messages

Streams can be sent by both the plugin and the engine. The agent that is sending the stream is known as the _producer_, and the agent that receives the stream is known as the _consumer_.

All stream messages reference a stream ID. This identifier is an integer starting at zero and is specified by the producer in the message that described what the stream would be used for: for example, [`Call`](#call) or [`CallResponse`](#callresponse). A producer **should not** reuse identifiers it has used once before. The most obvious implementation is sequential, where each new stream gets an incremented number. It is not necessary for stream IDs to be totally unique across both the plugin and the engine: stream `0` from the plugin and stream `0` from the engine are different streams.

### `Data`

This message is sent from producer to consumer. The body is a 2-tuple (array) of (`id`, `data`).

The `data` is either a `List` map for a list stream, in which case the body is the [`Value`](#value) to be sent, or `Raw` for a raw stream, in which case the body is either an `Ok` map with a byte buffer, or an `Err` map with a [`ShellError`](#shellerror).

Examples:

```json
{
  "Data": [
    0,
    {
      "List": {
        "String": {
          "val": "Hello, world!",
          "span": {
            "start": 40000,
            "end": 40015
          }
        }
      }
    }
  ]
}
```

```json
{
  "Data": [
    0,
    {
      "Raw": {
        "Ok": [72, 101, 108, 108, 111, 44, 32, 119, 111, 114, 108, 100, 33]
      }
    }
  ]
}
```

```json
{
  "Data": [
    0,
    {
      "Raw": {
        "Err": {
          "IOError": {
            "msg": "disconnected"
          }
        }
      }
    }
  ]
}
```

### `End`

This message is sent from producer to consumer. The body is a single value, the `id`.

**Must** be sent at the end of a stream by the producer. The producer **must not** send any more [`Data`](#data) messages after the end of the stream.

The consumer **must** send [`Drop`](#drop) in reply unless the stream ended because the consumer chose to drop the stream.

Example:

```json
{
  "End": 0
}
```

### `Ack`

This message is sent from consumer to producer. The body is a single value, the `id`.

Sent by the consumer in reply to each [`Data`](#data) message, indicating that the consumer has finished processing that message. `Ack` is used for flow control. If a consumer does not need to process a stream immediately, or is having trouble keeping up, it **should not** send `Ack` messages until it is ready to process more `Data`.

Example:

```json
{
  "Ack": 0
}
```

### `Drop`

This message is sent from consumer to producer. The body is a single value, the `id`.

Sent by the consumer to indicate disinterest in further messages from a stream. The producer **may** send additional [`Data`](#data) messages after `Drop` has been received, but **should** make an effort to stop sending messages and [`End`](#end) the stream as soon as possible.

The consumer **should not** consider `Data` messages sent after `Drop` to be an error, unless `End` has already been received.

The producer **must** send `End` in reply unless the stream ended because the producer ended the stream.

Example:

```json
{
  "Drop": 0
}
```

## Encoding

### JSON

The JSON encoding defines messages as JSON objects. No separator or padding is required. Whitespace within the message object as well as between messages is permitted, including newlines.

The engine is more strict about the format it emits: every message ends with a newline, and unnecessary whitespace and newlines will not be emitted within a message. It is explicitly supported for a plugin to choose to parse the input from the engine by parsing each line received as a separate message, as this is most commonly supported across all languages.

Byte arrays are encoded as plain JSON arrays of numbers representing each byte. While this is inefficient, it is maximally portable.

MessagePack **should** be preferred where possible if performance is desired, especially if byte streams are expected to be a common input or output of the plugin.

### MessagePack

[MessagePack](https://msgpack.org) is a machine-first binary encoding format with a data model very similar to JSON. Messages are encoded as maps. There is no separator between messages, and no padding character is accepted.

Most messages are encoded in the same way as their JSON analogue. For example, the following [`Hello`](#hello) message in JSON:

```json
{
  "Hello": {
    "protocol": "nu-plugin",
    "version": "0.90.2",
    "features": []
  }
}
```

is encoded in the MessagePack format as:

```
81                   // map, one element
  a5 "Hello"         // 5-character string
  83                 // map, three elements
    a8 "protocol"    // 8-character string
    a9 "nu-plugin"   // 9-character string
    a7 "version"     // 7-character string
    a6 "0.90.2"      // 6-character string
    a8 "features"    // 8-character string
    90               // array, zero elements
```

(verbatim byte strings quoted for readability, non-printable bytes in hexadecimal)

Byte arrays are encoded with MessagePack's native byte arrays, which impose zero constraints on the formatting of the bytes within. In general, the MessagePack encoding is much more efficient than JSON and **should** be the first choice for plugins where performance is important and MessagePack is available.

## Embedded Nu types

Several types used within the protocol come from elsewhere in Nu's source code, especially the [`nu-protocol`](https://docs.rs/nu-protocol) crate.

Rust enums are usually encoded in [serde](https://serde.rs)'s default format:

```javascript
"Variant"             // Variant
{ "Variant": value }  // Variant(value)
{ "Variant": [a, b] } // Variant(a, b)
{
  "Variant": {
    "one": 1,
    "two": 2
  }
}                     // Variant { one: 1, two: 2 }
```

Structs are encoded as maps of their fields, without the name of the struct.

### `Value`

[Documentation](https://docs.rs/nu-protocol/latest/nu_protocol/enum.Value.html)

This enum describes all structured data used in Nu.

Example:

```json
{
  "Int": {
    "val": 5,
    "span": {
      "start": 90960,
      "end": 90963
    }
  }
}
```

`CustomValue` for plugins **may** only contain the following content map:

| Field    | Type       | Usage                                                              |
| -------- | ---------- | ------------------------------------------------------------------ |
| **type** | string     | **Must** be `"PluginCustomValue"`.                                 |
| **name** | string     | The human-readable name of the custom value emitted by the plugin. |
| **data** | byte array | Plugin-defined representation of the custom value.                 |

Plugins will only be sent custom values that they have previously emitted. Custom values from other plugins or custom values used within the Nu engine itself are not permitted to be sent to or from the plugin.

Example:

```json
{
  "CustomValue": {
    "val": {
      "type": "PluginCustomValue",
      "name": "database",
      "data": [36, 190, 127, 40, 12, 3, 46, 83]
    }
  }
}
```

### `Span`

[Documentation](https://docs.rs/nu-protocol/latest/nu_protocol/span/struct.Span.html)

Describes a region of code in the engine's memory, used mostly for providing diagnostic error messages to the user with context about where a value that caused an error came from.

| Field     | Type    | Usage                                          |
| --------- | ------- | ---------------------------------------------- |
| **start** | integer | The index of the first character referenced.   |
| **end**   | integer | The index after the last character referenced. |

### `PipelineDataHeader`

Describes either a single value, or the beginning of a stream.

| Variant                                                | Usage                                         |
| ------------------------------------------------------ | --------------------------------------------- |
| [`Empty`](#pipelinedataheader-empty)                   | No values produced; an empty stream.          |
| [`Value`](#pipelinedataheader-value)                   | A single value                                |
| [`ListStream`](#pipelinedataheader-liststream)         | Specify a list stream that will be sent.      |
| [`ExternalStream`](#pipelinedataheader-externalstream) | Specify an external stream that will be sent. |

#### `Empty`

An empty stream. Nothing will be sent. There is no identifier, and this is equivalent to a `Nothing` value.

The representation is the following string:

```json
"Empty"
```

#### `Value`

A single value. Does not start a stream, so there is no identifier. Contains a [`Value`](#value).

Example:

```json
{
  "Value": {
    "Int": {
      "val": 2,
      "span": {
        "start": 9090,
        "end": 9093
      }
    }
  }
}
```

#### `ListStream`

Starts a list stream. Expect [`Data`](#data) messages with the referenced ID.

Contains <a name="liststreaminfo">`ListStreamInfo`</a>, a map:

| Field  | Type    | Usage                 |
| ------ | ------- | --------------------- |
| **id** | integer | The stream identifier |

Example:

```json
{
  "ListStream": {
    "id": 2
  }
}
```

#### `ExternalStream`

External streams are composed of optional `stdout` and `stderr` raw streams and/or an `exit_code` list stream.

| Field                | Type                                        | Usage                                                         |
| -------------------- | ------------------------------------------- | ------------------------------------------------------------- |
| **span**             | [`Span`](#span)                             | The source code reference that caused the stream.             |
| **stdout**           | [`RawStreamInfo`](#rawstreaminfo) or null   | Standard output of the command                                |
| **stderr**           | [`RawStreamInfo`](#rawstreaminfo) or null   | Standard error of the command                                 |
| **exit_code**        | [`ListStreamInfo`](#liststreaminfo) or null | The exit code (integer) of the command                        |
| **trim_end_newline** | boolean                                     | True if Nu **should** remove the last newline from the stream |

<a name="rawstreaminfo">`RawStreamInfo`</a> is a map as follows:

| Field          | Type            | Usage                                                                           |
| -------------- | --------------- | ------------------------------------------------------------------------------- |
| **id**         | integer         | The stream identifier                                                           |
| **is_binary**  | boolean         | True if the stream **should** be immediately treated as binary (non-text) data. |
| **known_size** | integer or null | If known, the size of the stream contents in bytes.                             |

The exit code stream is also optional, and is expected to send one integer representing a command exit code at any time, where `0` represents success and any other value is treated as a failure - but will not automatically cause an error to be generated within Nu.

Example:

```json
{
  "ExternalStream": {
    "stdout": {
      "id": 0,
      "is_binary": true,
      "known_size": 262144
    },
    "stderr": null,
    "exit_code": {
      "id": 1
    }
  }
}
```

### `ShellError`

[Documentation](https://docs.rs/nu-protocol/latest/nu_protocol/enum.ShellError.html)

This enum describes errors produced by the engine. As this enum is quite large and grows frequently, it is recommended to try to send this back to the engine without interpreting it, if possible.
