# 0.6.1 / 12.03.2025

- fix a bug with OP being `nil` in some cases.

# 0.6.0 / 24.12.2024

- add `protocol.receive` method for handling client input
- rework `protocol.read` in terms of `protocol.receive`.

# 0.5.0 / 22.10.2023

- add a new `,return` OP to the protocol

# 0.4.2 / 24.08.2023

- provide formats as additional data in the `read` op

# 0.4.1 / 24.08.2023

- Allow overriding the Fennel module location when setting up the protocol

# 0.4.0 / 24.08.2023

- allow `io.read` to read multiple patterns as separate values

# 0.3.0 / 08.05.2023

- Change `read` OP message format, allowing client to interpret it in a way that's meaningful for it.

# 0.2.0 / 04.11.23

- Change `read` OP message format, providing a type of communication.

# 0.1.0 / 04.06.23

- Initial protocol release.
