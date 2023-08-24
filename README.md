# Fennel Proto REPL Protocol

This repository hosts the protocol powering the `fennel-proto-repl` module in Emacs's [fennel-mode][1] package.
The protocol itself is editor-agnostic and can be configured to support various message formats.

The code of the protocol can be either loaded as a library or directly embedded in the client.
This library requires Fennel version 1.3.1+.
Sending and receiving messages is done by wrapping the `___repl___.readChunk` and `___repl___.onValues` functions, and thus should work for custom REPLs (like the one in the [min-love2d-fennel][2]).

Up until the 1.0.0 version everything is a subject to breaking changes.

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

If the initialization was successful, the protocol will respond with a message of ID `0` with the OP `"init"` and a  `"done"` status, followed by protocol and environment versions.
After that the editor can send and receive messages:

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

Additionally, if the Fennel module located elsewhere and it's impossible to require it with `(require :fennel)` call, the second parameter to the procol function can be used to specify the location:

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
- `{"id": ID, "op": "read", "pipe": "path"}` - the REPL process requested user input.
  By default the user input is handled by attaching it to the named pipe.
  See [the input-handling section](#handling-user-input) for more info.
- `{"id": ID, "op": "eval", "values": ["value 1", "value 2"]}` - the REPL process returned the results of the evaluation.
  Values are stored in a list.
  The same format goes for all OPs listed above that weren't mentioned in this list.
- `{"id": ID, "op": "error", "type": "kind", "data": "message", "stacktrace": "trace"}` - the REPL process encountered an error.
  The `"stacktrace"` key is optional, and may not be present.
- `{"id": ID, "op": "done",}` - REPL process is done processing the message.

All messages are processed sequentially due to the limitation of the underlying runtime.
A client decides what to do with each incoming message or not to process one at all.

All messages must be complete - no partial expressions are supported.
The protocol will discard any unfinished expression and return a `parse` error.
This is done because the REPL process must never be blocked to wait for more input while processing a message, since more messages can come from different sources.

For example, if the user were to send an unfinished expression, it would be stored in the REPL `readChunk` method.
While not technically a problem, as the user can just finish their expression in the next message, the client has to keep track of the ID, and it also can try to send a different request, and this would mix up two messages in the `readChunk` buffer.
This can happen if the automatic completion is enabled, and the client sends a message querying the REPL for completions on every key press.

## Modifying the protocol at runtime

The protocol can be further altered by sending code to the REPL after the connection is established.
Here's an example of changing the outgoing message format to Emacs property lists:

```fennel
(do (fn protocol.format [data] "Format data as emacs-lisp plist." (:  "(%s)" :format (table.concat (icollect [_ [k v] (ipairs data)] (: ":%s %s" :format k (: (case v {:list data} (: "(%s)" :format (table.concat data " ")) {:string data} (fennel.view data) {:sym data} (case data true :t false :nil _ (tostring data)) _ (protocol.internal-error "wrong data kind" (fennel.view v))) :gsub "\n" "\\\\n"))) " "))) {:id 1000 :nop ""})
```

Few things to note:

1. Contrary to the format function we're supplying during the upgrade, here we're modifying the `protocol.format` itself, so it has a different argument list.
2. The code above is formatted as a single line, because the protocol operates on a single line messages.
   Code minification is performed by the client.
   The client should remove all newlines, escape newlines in strings, and remove all of the comments from the code being sent to the process.
3. Finally, the code is wrapped in a `(do ... {id 1 :nop ""})` structure to inject the code and evaluate it in the protocol environment instead of the urer environment, and still conform to the message specification.
   This is a general mechanism for modifying the protocol at runtime.

Protocol methods that are available to be changed:

- `(protocol.format data)` - how to format the outgoing messages.
  Data is a message in the following format: `[KEY {KIND DATA}]`, where KEY is an arbitrary string, KIND is one of `"sym"`, `"string"`, or `"list"`, and DATA is an arbitrary data that corresponds to the KIND.
- `(protocol.mkfifo)` - creates a named FIFO pipe.
  Returns an absolute path to the file.
- `(protocol.read callback ...)` - how to read user input.
  Further notes on this method are below.

Additionally, `(protocol.internal-error cause message)` can be called to signal an internal error, that the client may handle in a special way.
This method must not be changed, it is exposed only to be used in other methods.
The `protocol.id` variable provides the ID of the currently processed request, and must not be altered.
There's also `protocol.op` which provides the OP of the currently processed message, it should not be altered as well.

## Handling the user input

The protocol defines a custom `read` function that works in place of `io.read` and the default implementation relies on named pipes, meaning that the server and the client are on the same machine, which is usually the case for Fennel.
Input handling is an open interface, meaning that any client is free to implement its own way of communication and process `read` messages in the way meaningful for the implementation.

The `protocol.read` method accepts a `message` callback followed by the modes same as for `io.read`.
The `message` callback is used to tell the client how and where to pass the input.
The message itself must contain the ID, OP, and all the other necessary information for client to send the data back, which the client is free to interpret however it needs to.

An example message can look like this:

```fennel
[[:id {:sym protocol.id}]
 [:op {:string :read}]
 [:pipe {:string "/path/to/the/pipe"}]]
```

This method is not portable, and will not work under Windows because the named pipes can't be created in the same way (unless the REPL is running in Cygwin or WSL (not tested)).
The client can redefine `protocol.mkfifo` in a system-dependent way or provide it's own implementation of `protocol.read` which doesn't rely on pipes at all.

For example, here's an implementation of `protocol.read` which uses [luasocket][3] to starts a small socket server and wait for data from the client.
Instead of providing a `"pipe"` parameter in the message, it provides the port number:

```fennel
(fn protocol.read [message ...]
  "Start a new socket server, and wait for a client to be attached.
Then reads data for every given format."
  (let [socket (require :socket)
        server (socket.bind :localhost 0)
        (_ port) (server:getsockname)
        _ (message [[:id {:sym protocol.id}]
                    [:op {:string :read}]
                    [:port {:sym port}]])
        client (server:accept)
        data []]
    (for [i 1 (select :# ...)]
      (let [fmt (select i ...)]
        (tset data i (client:receive fmt))))
    (client:close)
    (server:close)
    (table.unpack data 1 (select :# ...))))
```

The client may implement a read handler that, given a message with the `read` OP and the `port` key will connect to that port over a socket on the `localhost` (or a different host) and write the data into it.
Here's the example session, assuming we've already redefined the `protocol.read` method:

```fennel
>> {:id 2 :eval "(io.read :*l 1 1 1)"}
{"id": 2, "op": "accept"}
{"id": 2, "op": "read", "port": 40013}
```

Now we can attach to the given port with `telnet` acting as a client:

```
$ telnet localhost 40013
Trying ::1...
Connected to localhost.
Escape character is '^]'.
first line
abc
Connection closed by foreign host.
```

We've written two lines into the socket, and after closing the connection we should see that reflected in the REPL process:

```fenel
{"id": 2, "op": "eval", "values": ["\"first line\\r\"", "\"a\"", "\"b\"", "\"c\""]}
{"id": 2, "op": "done"}
```

I hope this displays the idea of customization possibilities this protocol allows.

[1]: https://git.sr.ht/~technomancy/fennel-mode
[2]: https://gitlab.com/alexjgriffith/min-love2d-fennel/-/blob/ecce4e3e802b3a85490341e13f8c562315f751d2/lib/stdio.fnl
[3]: https://w3.impa.br/~diego/software/luasocket/home.html
[4]: https://git.sr.ht/~technomancy/fennel-mode/tree/main/item/fennel-proto-repl.el
