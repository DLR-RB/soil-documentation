# SOIL Encodings

SOIL provides an abstraction layer to convert SOIL messages to binary data, and vice versa.
Different encodings exist, depending on the use case, and they can be mixed and matched with different frame encodings and transport layers.
The following encodings are currently defined for SOIL:

- [JSON](JSON.md): Standard JSON encoding, useful for debugging, fast integration, and simple devices
- [Compact JSON](Compact%20JSON.md): Like JSON, but with short field names and integer enums, for smaller messages.
- [Binary](Binary.md): A compact binary format specifically designed for SOIL.
- [Protobuf](protobuf/Protobuf.md): Binary format using Google Protocol Buffers, with good cross-platform support.

It's also possible to define custom encodings to match existing implementations, by directly implementing a `MessageEncoding` in the language of choice.
