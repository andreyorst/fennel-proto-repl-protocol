# Fennel Proto REPL Protocol

This repository hosts the protocol powering the `fennel-proto-repl` module in Emacs's [fennel-mode][1] package.
The protocol itself is editor-agnostic and can be configured to support various message formats.

The code of the protocol can be either loaded as a library or directly embedded in the client.
This library requires Fennel version 1.3.1+.
Sending and receiving messages is done by wrapping the `___repl___.readChunk` and `___repl___.onValues` functions, and thus should work for custom REPLs (like the one in the [min-love2d-fennel][2]).

Up until the 1.0.0 version everything is subject to breaking changes.

## Usage overview

The communication is based on structured messages - the client sends Fennel-formatted tables and receives messages in a specified format.
To specify a format, the client must provide a format function to the protocol upon start.
For example, here's a format function for JSON-based communication:

```fennel
(fn format-json [env data]
  (: "{%s}" :format
     (table.concat
      (icollect [_ [k v] (ipairs data)]
        (: "%s: %s" :format (env.fennel.view k)
           (: (case v
                {:list data} (.. "[" (table.concat data ", ") "]")
                {:string data} (env.fennel.view data {:byte-escape #(: "\\\\x%2x" :format $)})
                {:sym data} (tostring data)
                _ (env.protocol.internal-error "Wrong data kind" (env.fennel.view v)))
              :gsub "\n" "\\\\n"))) ", ")))
```

By loading the `protocol.fnl` file in the REPL and immediately calling the result with the format function the REPL will be *upgraded* to a message-based communication format:

```fennel
>> ((require :protocol) format-json)
{"id": 0, "op": "init", "status": "done", "protocol": "0.1.0", "fennel": "1.3.1-dev", "lua": "PUC Lua 5.4"}
```

If the initialization is successful, the protocol will respond with a message of ID `0` with the OP `"init"` and a  `"done"` status, followed by protocol and environment versions.
After that, the editor can send and receive messages:

```fennel
>> {:id 1 :eval "(+ 1 2 3)"}
{"id": 1, "op": "accept"}
{"id": 1, "op": "eval", "values": ["6"]}
{"id": 1, "op": "done"}
```

The protocol then wraps all REPL's IO in such a way that the message format is always the same:

```fennel
>> {:id 2 :eval "(io.write :foo)"}
{"id": 2, "op": "accept"}
{"id": 2, "op": "print", "descr": "stdout", "data": "foo"}
{"id": 2, "op": "eval", "values": ["#<file (0x7f1afd149780)>"]}
{"id": 2, "op": "done"}
>> {:id 3 :eval "(io.read :n)"}
{"id": 3, "op": "accept"}
{"id": 3, "op": "read", "data": "/tmp/fennel-proto-repl.FIFO.1384dFmV"}
{"id": 3, "op": "eval", "values": ["42"]}
{"id": 3, "op": "done"}
```

Additionally, if the Fennel module is located elsewhere and it's impossible to require it with the `(require :fennel)` call, the second parameter to the protocol function can be used to specify the location:

```fennel
>> ((require :protocol) format-json :lib.fennel)
{"id": 0, "op": "init", "status": "done", "protocol": "0.1.0", "fennel": "1.3.1-dev", "lua": "PUC Lua 5.4"}
```

It's useful when the application embeds a specific version of Fennel as a library and uses it internally.
The client may use this to match fennel versions in case there's an ambiguity.

## API

This protocol is based on a one-message-per-line approach, where each message is a serialized data structure.

Incoming messages are expected to be Fennel tables:

```fennel
{:id ID OP DATA}
```

Where the `ID` is a positive integer, `OP` is a string key describing the kind of operation to apply to the `DATA` string.
Zero, and negative integer IDs are reserved for internal use.

Supported OPs:

- `eval`: evaluate a string of Fennel code.
- `complete`: produce all possible completions for a given input symbol.
- `doc`: produce documentation of a symbol.
- `reload`: reload the module.
- `find`: print the filename and line number for a given function.
- `compile`: compiles the expression into Lua and returns the result.
- `apropos`: produce all functions matching a pattern in all loaded modules.
- `apropos-doc`: produce all functions that match the pattern in their docs.
- `apropos-show-docs`: produce all documentation matching a pattern in the function name.
- `help`: show REPL message in the REPL.
- `reset`: erase all REPL-local scope.
- `exit`: leave the REPL.
- `nop`: ignore the operation.

Each OP expects its own DATA format.

A message is then accepted by the REPL and processed accordingly to its OP key.
After accepting the message the REPL responds with an ACCEPT message followed by several more messages, depending on the operation performed.
Once a message is fully processed the REPL responds with the "done" OP.

Each response message is formatted as specified by the format function and always includes the ID and OP keys, followed by other keys, which are OP-specific.
Possible messages formatted as JSON:

- `{"id": 0, "op": "init", "status": "done", "protocol": "ver", "fennel": "ver", "lua": "ver"}` - initialization of the REPL was successful.
  The protocol always responds with the first message having a 0 ID.
- `{"id": 0, "op": "init", "status": "fail", "data": "error message"}` - initialization of the REPL wasn't successful.
- `{"id": ID, "op": "accept", "data": "done"}` - message was accepted by the REPL process.
- `{"id": ID, "op": "print", "descr": "stdin", "data": "text"}` - the REPL process requested `"data"` to be printed.
- `{"id": ID, "op": "read", "formats": ["L"]}` - the REPL process requested user input.
- `{"id": ID, "op": "eval", "values": ["value 1", "value 2"]}` - the REPL process returned the results of the evaluation.
  Values are stored in a list.
  The same format goes for all OPs listed above that weren't mentioned in this list.
- `{"id": ID, "op": "error", "type": "kind", "data": "message", "stacktrace": "trace"}` - the REPL process encountered an error.
  The `"stacktrace"` key is optional, and may not be present.
- `{"id": ID, "op": "done",}` - REPL process is done processing the message.
- `{"id": ID, "op": "retry", "message": "{...}"}` - a special response, indicating that the client has to retry sending the message.
  See [the input-handling section](#handling-user-input) for more info.

All messages are processed sequentially due to the limitation of the underlying runtime.
A client decides what to do with each incoming message or not to process one at all.

All messages must be complete - no partial expressions are supported.
The protocol will discard any unfinished expression and return a `parse` error.
This is done because the REPL process must never be blocked to wait for more input while processing a message since more messages can come from different sources.

For example, if the user were to send an unfinished expression, it would be stored in the REPL `readChunk` method.
While not technically a problem, as the user can just finish their expression in the next message, the client has to keep track of the ID, and it also can try to send a different request, and this would mix up two messages in the `readChunk` buffer.
This can happen if the automatic completion is enabled, and the client sends a message querying the REPL for completions on every key press.

## Protocol functions

Several protocol functions are available to the client:

- `(protocol.receive id)` - receive one message of specific `id`.
- `(protocol.read format)` - read user input according to the `format`.
- `(protocol.message message)` - send a `message` to the client.
- `(protocol.env-set! key value)` - set a `key` to a `value` so it is visible in the user program's environment (`_G`).
- `(protocol.internal-error cause message)` - signal an internal error, that the client may handle in a special way.

## Modifying the protocol at runtime

The protocol can be further altered by sending code to the REPL after the connection is established.
Here's an example of changing the outgoing message format to Emacs property lists:

```fennel
(do
  (fn protocol.format [data]
    "Format data as emacs-lisp plist."
    (: "(%s)" :format
       (table.concat
        (icollect [_ [k v] (ipairs data)]
          (: ":%s %s" :format k
             (: (case v
                  {:list data} (: "(%s)" :format (table.concat data " "))
                  {:string data} (fennel.view data)
                  {:sym data} (case data true :t false :nil _ (tostring data))
                  _ (protocol.internal-error "wrong data kind" (fennel.view v)))
                :gsub "\n" "\\\\n")))
        " ")))
  {:id 1000 :nop ""})
```

A few things to note:

1. Contrary to the format function we're supplying during the upgrade, here we're modifying the `protocol.format` itself, so it has a different argument list.
2. The code above must be formatted as a single line because the protocol operates on single-line messages.
   Code minification is performed by the client.
   In general, the client should remove all newlines, escape newlines in strings, and remove all of the comments from the code being sent to the process.
3. Finally, the code is wrapped in a `(do ... {id 1000 :nop ""})` structure to inject the code and evaluate it in the protocol environment instead of the user environment, and still conform to the message specification.
   This is a general mechanism for modifying the protocol at runtime.

It is not recommended to change any other protocol function other than `protocol.format`

## Handling IO

The protocol defines a custom `read` function that overrides `io.read` and other related functions, such as `io.stdin:read`.
When encountering a `read` call, the protocol sends a message with the `read` OP and a list of formats:

```fennel
[[:id {:sym protocol.id}]
 [:op {:string :read}]
 [:formats {:list ["L"]}]]
```

Then waits for a response.

The client has to implement Lua's `io.read` function by some means, handling the following formats:

- `n` - read a number and parse it;
- `l`/`*l` - read a line without the line end character;
- `L`/`*L` - read a line including the line end character.
- `a`/`*a` - read everything;

The client then sends a normal message with the same `protocol.id` as received with the `read` OP.
The message has to contain the `id` and the `data` fields.
Contents of the `data` field are passed back to the application.

If another operation occurs during the read process, the protocol will send a `retry` OP.
When client receives a `retry` OP, it should re-send the message as is without any modification to the REPL process, after a short delay if possible.

[1]: https://git.sr.ht/~technomancy/fennel-mode
[2]: https://gitlab.com/alexjgriffith/min-love2d-fennel/-/blob/ecce4e3e802b3a85490341e13f8c562315f751d2/lib/stdio.fnl
[3]: https://w3.impa.br/~diego/software/luasocket/home.html
[4]: https://git.sr.ht/~technomancy/fennel-mode/tree/main/item/fennel-proto-repl.el
