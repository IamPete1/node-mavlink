# Node.js MavLink library

This package is the implementation of serialization and parsing of MavLink messages for v1 and v2 protocols.

It consists partially of code generated from the XML documents in the original MavLink repository and a few pieces that define the parser and serializer.

## Getting started

It is extremely easy to get started using this library. First you need to install the required packages. For demonstration purposes we'll be using the `serialport` package to read the data from serial port.

### Reading messages

```
$ npm install --save node-mavlink serialport
```

Once you've done it you can start using it. First you'll need a serial port that can parse messages one by one. Please note that since we're using ECMAScript modules the file name should end with `.mjs` extension (e.g. `test.mjs`)

```
import * as SerialPort from 'serialport'
import { MavLinkPacketSplitter, MavLinkPacketParser } from 'node-mavlink'

// substitute /dev/ttyACM0 with your serial port!
const port = new SerialPort('/dev/ttyACM0')

// constructing a reader that will emit each packet separately
const reader = port
  .pipe(new MavLinkPacketSplitter())
  .pipe(new MavLinkPacketParser())

reader.on('data', packet => {
  console.log(packet)
})
```

That's it! That is all it takes to read the raw data. But it doesn't end there - in fact this is only the beginning of what this library can do for you.

Each message consists of multiple fields that contain specific data. Parsing the data is also very easy.

```
import { minimal, common, ardupilotmega, uavionix, icarous } from 'node-mavlink'

// create a registry of mappings between a message id and a data class
const REGISTRY = {
  ...minimal.REGISTRY,
  ...common.REGISTRY,
  ...ardupilotmega.REGISTRY,
  ...uavionix.REGISTRY,
  ...icarous.REGISTRY,
}

reader.on('data', packet => {
  const clazz = REGISTRY[packet.header.msgid]
  if (clazz) {
    const data = packet.protocol.data(packet.payload, clazz)
    console.log('Received packet:', data)
  }
})
```

### Sending messages

Sending messages is also very easy. One example that is very useful is to send the `REQUEST_PROTOCOL_VERSION` to switch to protocol version 2.

```
import { MavLinkProtocolV2 } from 'node-mavlink'

// MavLink requires that the messages are indexed
let seq = 0

// Create an instance of of the `CommandInt` class that will be the vessel
// for containing the command data
const msg = new common.CommandInt()
msg.command = common.MavCmd.REQUEST_PROTOCOL_VERSION
msg.param1 = 1

// Serialize the message using the v2 protocol
const buffer = new MavLinkProtocolV2().serialize(msg, seq++)

// Limit the sequence to be within the 0..255 range
seq &= 255

port.write(buffer)
```

## Interacting with other communication mediums

The splitter and parser work with generic streams. Of course the obvious choice for many use cases will be a serial port but the support doesn't end there.

There are options for streams working over network (TCP or UDP), GSM network - pretty much anything that sends and receives data over Node.js `Stream`s.

## Closing thoughts

The original generated sources lack one very important aspect of a reusable library: documentation. Also, most of the time the names are more C-like than JavaScript-like.

When generating sources for data classes a number of things happen:

- `enum` values are trimmed from common prefix; duplicating enum name in its value when its value cannot be spelled without the enum name is pointless and leads to unnecessairly vebose code
- enum values, whenever available, contain JSDoc describing their purpose
- data classes are properly named (PascalCase)
- data classes fields are properly named (camelCase)
- both data classes and their fields whenever available also contain JSDoc
- if a particular data class is deprecated that information is also available in the JSDoc
- some class names are modified compared to their original values (e.g. `Statustext` => `StatusText`)

This leads to generated code that contains not only raw types but also documentation where it is mostly useful: right at your fingertips.

I hope you'll enjoy using this library! If you have any comments, find a bug or just generally want to share your thoughts you can reach me via emial: padcom@gmail.com

Peace!
