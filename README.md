# Fennel Proto REPL Protocol

This repository hosts the protocol powering the `fennel-proto-repl` module in Emacs's [fennel-mode][1] package.
The protocol itself is editor-agnostic and can be configured to support various message formats.

The code of the protocol can be either loaded as a library or directly embedded in the client.
This library requires Fennel version 1.3.1+.
Sending and receiving messages is done by wrapping the `___repl___.readChunk` and `___repl___.onValues` functions, and thus should work for custom REPLs (like the one in the [min-love2d-fennel][2]).

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
                {:list data} (.. "[" (table.concat data " ") "]")
                {:string data} (env.fennel.view data)
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
{:id 1 :eval "(+ 1 2 3)"}
{"id": 1, "op": "accept"}
{"id": 1, "op": "eval", "values": ["6"]}
{"id": 1, "op": "done"}
```

The protocol then wraps all REPL's IO in such a way that the message format is always the same:

```fennel
{:id 2 :eval "(io.write :foo)"}
{"id": 2, "op": "accept"}
{"id": 2, "op": "print", "descr": "stdout", "data": "foo"}
{"id": 2, "op": "eval", "values": ["#<file (0x7f1afd149780)>"]}
{"id": 2, "op": "done"}
{:id 3 :eval "(io.read :n)"}
{"id": 3, "op": "accept"}
{"id": 3, "op": "read", "data": "/tmp/fennel-proto-repl.FIFO.1384dFmV"}
{"id": 3, "op": "eval", "values": ["42"]}
{"id": 3, "op": "done"}
```

## API

This protocol is based on a one-message-per-line approach, where each message is a serialized data structure.

Incoming messages are expected to be Fennel tables:

```fennel
{:id ID OP DATA}
```

Where the `ID` is a positive integer, `OP` is a string key describing the kind of operation to apply to the `DATA` string.
Each operation expects its own data format.
Supported operations:

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

Each message is then accepted by the REPL and processed accordingly to its op key.
After accepting the message the REPL responds with an ACCEPT message followed by several more messages, depending on the operation.
Once a message is fully processed the REPL responds with the "done" OP.

Each response message is formatted as specified by the format function and always includes the ID and OP keys, followed by other keys, which are OP-specific.
Possible messages formatted as JSON:

- `{"id": 0, "op": "init", "status": "done", "protocol": "ver", "fennel": "ver", "lua": "ver"}` - initialization of the REPL was successful.
  The protocol always responds with the first message having a 0 ID.
- `{"id": 0, "op": "init", "status": "fail", "data": "error message"}` - initialization of the REPL wasn't successful.
- `{"id": ID, "op": "accept", "data": "done"}` - message was accepted by the REPL process.
- `{"id": ID, "op": "print", "descr": "stdin", "data": "text"}` - the REPL process requested `"data"` to be printed.
- `{"id": ID, "op": "read", "data": "named-pipe"}` - the REPL process requested user input.
  The user input is handled by attaching it to the named pipe.
- `{"id": ID, "op": "eval", "values": ["value 1", "value 2"]}` - the REPL process returned the results of the evaluation.
  Values are stored in a list.
  The same format goes for all OPs listed above that weren't mentioned in this list.
- `{"id": ID, "op": "error", "type": "kind", "data": "message", "stacktrace": "trace"}` - the REPL process encountered an error.
  The `"stacktrace"` key is optional, and may not be present.
- `{"id": ID, "op": "done",}` - REPL process is done processing the message.

All messages are processed sequentially due to the limitation of the underlying runtime.
A client decides what to do with each message or not to process one at all.

The protocol can be further altered by sending code to the REPL.
Here's an example of changing the outgoing message format as Emacs plists:

```fennel
((fn to-plist-protocol []
   (fn protocol.format [data]
     "Format data as emacs-lisp plist."
     (:  "(%s)" :format
         (table.concat
          (icollect [_ [k v] (ipairs data)]
            (: ":%s %s" :format k
               (: (case v
                    {:list data} (: "(%s)" :format (table.concat data " "))
                    {:string data} (fennel.view data)
                    {:sym data} (case data true :t false :nil _ (tostring data))
                    _ (protocol.internal-error "wrong data kind" (fennel.view v)))
                  :gsub "\n" "\\\\n"))) " ")))
   {:id 1000 :nop ""}))
```

(Note that here we're modifying the protocol.format directly, so it already happens in the environment of the protocol.)

Protocol methods that are available to be changed:

- `(protocol.format data)` - how to format outgoing messages.
  Data is a message in the following format: `[KEY {KIND DATA}]`, where KEY is an arbitrary string, KIND is one of `"sym"`, `"string"`, or `"list"`, and DATA is an arbitrary data that corresponds to the KIND.
- `(protocol.mkfifo)` - creates a named FIFO pipe.
  Returns an absolute path to the file.
- `(protocol.read mode callback op)` - how to read user input.
  Must call callback with the following message:
  ```fennel
  [[:id {:sym protocol.id}]
   [:op {:string OP}]
   [:data {:string FIFO}]]
  ```
  The FIFO in the message is a file that then is read by the REPL,
  and to which the client will write the data.  The OP in the
  message is an argument for this function.

Additionally, `(protocol.internal-error cause message)` can be called to signal an internal error, that the client may handle in a special way.
This method must not be changed, it is exposed only to be used in other methods.
The `protocol.id` variable provides the ID of the currently processed request, and must not be altered.
There's also `protocol.op` which provides the OP of the currently processed message.

[1]: https://git.sr.ht/~technomancy/fennel-mode
[2]: https://gitlab.com/alexjgriffith/min-love2d-fennel/-/blob/ecce4e3e802b3a85490341e13f8c562315f751d2/lib/stdio.fnl
