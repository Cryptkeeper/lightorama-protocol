# Light-O-Rama Protocol
This documentation covers the Light-O-Rama (LOR) communication protocol for [AC lighting control units](http://www1.lightorama.com/pro-ac-light-controllers/). It has been manually reverse engineered using the [LOR Hardware Utility](http://www.lightorama.com/help/index.html?hardware_utility.htm), serial port monitoring software and a `LOR1602WG3` unit.

Given the reverse engineered nature, this documentation should be considered incomplete, incorrect and outdated. 

It is provided as is. 

## Test Environment
I am using a `LOR1602WG3` unit with 16 channels (see the [datasheet](http://www1.lightorama.com/PDF/LOR160xWg3_Datasheet.pdf)), an ID of `0x01`, and a 19.2K baud rate. The baud rate for a LOR network is heavily dependant on its usage. Check out [LOR's documentation](http://www1.lightorama.com/network-speeds/) for selecting a baud rate.

Light-O-Rama, Inc. provides datasheets and manuals for many of their products at [http://www1.lightorama.com/documentation/](http://www1.lightorama.com/documentation/)

## Network
### Stateful Protocol
Only changed channel states should be sent. Some program implementations may choose to occasionally [resend existing state commands](https://github.com/smeighan/xLights/blob/master/xLights/outputs/LOROutput.cpp#L107) as a "sanity" measure. However, this may result in visual glitches caused by resetting effect timers and increases bandwidth use.

### Heartbeat Behavior
Every 0.5s the LOR Hardware Utility sends a heartbeat payload onto the network. 

If the unit has not recently received a heartbeat payload, it will mark itself not connected and become inactive. The timeout value seems to be around 2 seconds.

### Empty Bytes
An empty byte (`0x00`) appears to be used to drive the controller's command execution system and network framing/flushing behavior.

An empty byte marks the end of a command which instructs the controller to execute its command buffer. Prefixing commands with an empty byte ensures the buffer is flushed prior to the new command, improving fault tolerance by preventing corrupt/invalid commands from remaining in the buffer.

Additionally, empty bytes are consistently sent over the network. This may be either additional padding to maintain expected frame lengths, flush an outbound queue, or to simply ensure that even with data loss, the command buffer is still being flushed.

#### Notes
* `0x00` does not otherwise appear in the protocol, even in values.
* To reduce bandwidth usage, you may be able to omit the empty byte command prefix. The LOR Sequencer appears to automatically do this when sending multiple commands.

## Magic Numbers & Formats
Whether by design or by obfuscation the protocol contains several magic numbers and domain-specific encoding formats. Avoid duplicating these in code implementations as they may change.

### Heartbeat Payload
The heartbeat payload is a constant set of 5 magic bytes: `[0x00, 0xFF, 0x81, 0x56, 0x00]`

`0xFF` likely represents a broadcast since its index is typically used for routing information.

### Brightness
Brightness is encoded as a `uint8` between `0x01` (100% brightness) and `0xF0` (0% brightness). These values have been captured as output of the LOR Hardware Utility. Values outside these min/max bounds seem to result in indeterminate and unreliable behavior.

You can convert a given value (between [0, 1]) using:

`byteBrightness = floatBrightness * (maxBrightness - minBrightness) + minBrightness` 

...where `maxBrightness` and `minBrightness` are `0xF0` and `0x01` respectively. 

This example results in a linear brightness curve. Some program implementations, such as xLights, may use a [custom curve](https://github.com/smeighan/xLights/blob/master/xLights/outputs/LOROutput.cpp#L66). The brightness curve's behavior is up to the developer and is not restricted by the hardware beyond the previously specified min/max values.

The [datasheet](http://www1.lightorama.com/PDF/LOR160xWg3_Datasheet.pdf) for my `LOR1602WG3` unit specifies "100 intensity levels for dimming". So while you may adjust the brightness curve, you are unable to produce new brightness _levels_.

### Durations
As determined from the LOR Hardware Utility (and the [datasheet](http://www1.lightorama.com/PDF/LOR160xWg3_Datasheet.pdf) for my `LOR1602WG3` unit), durations have a minimum value of 0.1s and a maximum of 25s with 256 available levels between those values.

Before being encoded, the value must be scaled. Given `timeSeconds` as a duration in seconds (such as 0.1s or 5s), you can calculate its scaled value as:

`scaledValue = round(maxDuration / (timeSeconds / minDuration)))` 

...where `maxDuration` and `minDuration` have values of `5099` (the decimal value of the captured hex encoded form of 0.1s, `[0x13, 0xEB]`) and `0.1` respectively. If `scaledValue <= 0xFF`, then it is encoded as `[0x80, scaledValue]` where `0x80` is a magic number representing an "empty" byte ignored by the unit. Scaled values larger than `0xFF` must be encoded in big endian format as a uint16.

#### Examples
| Timing | Scaled Value | Encoded Format |
| - | - | - |
| 0.1s | 5099 | `[0x13, 0xEB]` |
| 0.5s | 1020 | `[0x03, 0xFC]` |
| 1s | 510 | `[0x01, 0xFE]` |
| 2s | 255 | `[0x80, 0xFF]` |
| 25s | 20 | `[0x80, 0x14]` |

#### Notes
- As a reference, 2s is encoded as `[0x80, 0xFF]` (with a decimal value of `255`) which defines the break point in the previously mention encoding logic.
- The scale used is exponential*ish* and offers higher precision for lower value durations. For example, the delta of the scaled values of 0.1s and 0.2s is 232x larger than the delta between the scaled values of 2.1s and 2.2s.
- Any scale could technically be applied atop this behavior assuming the resulting encoded values stay within the assumed boundaries of `5099` (0.1s) and `20` (25s).
- While the LOR Hardware Utility only offers durations in increments by 0.1s, the encoding format does allow (while variable) additional precision. Your unit however may disregard or round this value ([consult your unit's datasheet](http://www1.lightorama.com/documentation/)).

### Command IDs
Command IDs represent a predefined action for the controller to execute.

| Name | Value | Description |
| - | - | - |
| On | `0x01` | Sets a channel to 100% brightness |
| Set Brightness | `0x03` |Sets a channel's brightness |
| Fade | `0x04` | Fades a channel between two brightness values |
| Set Twinkle | `0x06` | Sets a channel to twinkling mode |
| Set Shimmer | `0x07` | Sets a channel to shimmer mode |

### Unit IDs
Unit IDs begin at `0x01` and end at `0xF0` (240). A unit ID of `0xF1` (241) seems to be used by the search request behavior as an "any" selector.

### Channel IDs
Channel IDs can be determined by taking the index of the channel (starting at 0, not 1) plus 128 (`0x80`).

This can be easily implemented in code as `channelId = 0x80 | channelIndex`.

Channel numbers greater than or equal to 127 (the max value that can fit a `uint8` with the `0x80` offset) should be encoded as multiple bytes in [big endian format](https://en.wikipedia.org/wiki/Endianness). Each byte should still be offset by `0x80`.


## Single Channel Commands
Single channel commands exist within a parent structure containing routing data, a command ID, and **possibly** additional metadata with a format relative to the command ID. Commands which do not have additional metadata should omit the section entirely and should not send empty values.

### Parent Structure
| Field | Data Type | Notes |
| - | - | - |
| Header | `uint8` | Always `0x00` |
| Controller ID | `uint8` | |
| Command ID | `uint8` | |
| Metadata | | Command ID specific metadata structure |
| Channel ID | `uint8` | |
| End | `uint8` | Always `0x00` |

### Metadata Structures
#### On
None.

#### Set Brightness
| Name | Data Type |
| - | - |
| Brightness | `uint8` |

#### Fade
| Name | Data Type |
| - | - |
| Start Brightness | `uint8` |
| End Brightness | `uint8` |
| Duration | `[2]uint8` |

#### Set Twinkle
None.

#### Set Shimmer
None.

## Multi Channel Commands
Multi channel commands utilize the same command IDs and metadata structures as their single channel command equivalents with the exception that instead of sending a `uint8` channel ID a channel bitmask is sent.

The bit length of the channel mask corresponds to the amount of channels being updated rounded to the nearest multiple of 8. Channels with a 0 value bit are ignored and maintain their previous state.

### Parent Structure
| Field | Data Type | Notes |
| - | - | - |
| Header | `uint8` | Always `0x00` |
| Controller ID | `uint8` | |
| Command ID | `uint8` | Command ID offset by magic number |
| Metadata | | Command ID specific metadata structure |
| Chain Index | | Only sent when chaining multi channel commands due to the full channel configuration being unable to fit within a single channel mask |
| Channel Mask | | Length is dependent on channel count |
| End | `uint8` | Always `0x00` |

### Command ID Offsets
Command IDs are offset by a magic number dependent on the length of the channel mask. You can determine the multi channel equivalent of a command ID using `multiCommandId = commandIdOffset | commandId`.

| Channel Mask Length | Command ID Offset | Notes |
| - | - | - |
| 8 bits | `0x30` | |
| 16 bits | `0x10` | Tip: `0x10` has a decimal value of 16 |
| 16 bits | `0x50` | Only used when chaining commands |

For configurations for more than 16 channels, multiple commands are sent instead, each providing a portion of the full channel mask. Multiple lengths may be mixed together.

#### Notes
- Channel mask lengths can be mixed. For example, 40 channels would be sent as 2x 16 bit masks and 1x 8 bit mask.
- Controllers safely handle channel masks larger in length than the channel count, however they should be avoided to optimize bandwidth and processing speed.

### Channel Masking Example
The channel mask for channels 1, 7 and 14 in binary is `0010 0000 0100 0001`.

This can be calculated in code like so:
```
var mask uint16
mask |= 1 << 0 // set bit index 0 (channel 1) to 1
mask |= 1 << 6 // set bit index 6 (channel 7) to 1
mask |= 1 << 13 // set bit index 13 (channel 14) to 1
```

This mask is then encoded as: `[0x20, 0x41]`

### Chaining Example
Given a controller ID of `0x01` with 64 channels, we can set every channel to 0% brightness using 4 multi channel commands. The chain index is a reverse index, starting at the highest index plus `0x01`, ending at `0x01`. It must not be `0x00`, which marks the end of a command.

```
# Set Brightness of channels 0-15 (padding bytes omitted)
0x01 (controller ID)
0x53 (0x50 chaining command ID offset | 0x03 Set Brightness command ID)
0xF0 (0% brightness)
0x03 (chain index of 3)
0xFF (channel mask pt. 1)
0xFF (channel mask pt. 2)
```

Resending this command with a chain index of 2 and 1 would provide channel masks for channels 16-31 and 32-47 respectively. To end the chain, a final command is sent with an _unchained_ command ID offset and without a chain index.

```
# Set Brightness of channels 48-63 (padding bytes omitted)
0x01 (controller ID)
0x13 (16-bit command ID offset | 0x03 Set Brightness command ID)
0xF0 (0% brightness)
0xFF (channel mask pt. 1)
0xFF (channel mask pt. 2)
```

## Controller Commands
Controller commands impact all channels on a given controller ID without any filtering capability. They are used for full state changes and are primarily a bandwidth/execution optimization.

### All Off
Turns off configured channels for the given controller ID.

| Field | Data Type | Notes |
| - | - | - |
| Header | `uint8` | Always `0x00` |
| Controller ID | `uint8` | |
| Command ID | `uint8` | Always `0x41` |
| End | `uint8` | Always `0x00` |

## Extended Commands
### Background Fade
Background Fade enables applying a foreground command atop a background command. This allows execution of multiple single channel commands simultaneously on the same channel ID.

| Field | Data Type | Notes |
| - | - | - |
| Header | `uint8` | Always `0x00` |
| Controller ID | `uint8` | |
| Foreground Command ID | `uint8` | Only accepts `Set Twinkle` and `Set Shimmer` |
| Channel ID | `uint8` | 
| Magic Number | `uint8` | Always `0x81` (denotes extended command statement?) |
| Background Command ID | `uint8` | Only accepts `Fade` |
| Start Brightness | `uint8` | |
| End Brightness | `uint8` | |
| Duration | `[2]uint8` | |
| End | `uint8` | Always `0x00` |

## Reference Implementations
- [xLights](https://github.com/smeighan/xLights) is a LOR-like C++ program which offers support for LOR controller units. 
- [fpp](https://github.com/FalconChristmas/fpp/blob/master/src/channeloutput/LOR.cpp) is a sequence scheduling and player program which supports LOR output.
- [go-lightorama](https://github.com/Cryptkeeper/go-lightorama) is a Go library implemented given this documentation.
