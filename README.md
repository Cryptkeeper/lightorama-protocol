# Light-O-Rama Protocol
This documentation covers the Light-O-Rama (LOR) communication protocol for [AC lighting control units](http://www1.lightorama.com/pro-ac-light-controllers/). It has been manually reverse engineered using the [LOR Hardware Utility](http://www.lightorama.com/help/index.html?hardware_utility.htm), serial port monitoring software and a `LOR1602WG3` unit.

Given the reverse engineered nature, this documentation should be considered incomplete, incorrect and outdated. 

It is provided as is. 

## Configuration
I am using a `LOR1602WG3` unit at 19.2k with 16 channels. It has a controller ID of 0x01. The baud rate for a LOR network is heavily dependant on its usage and hardware. Check out [LOR's documentation](http://www1.lightorama.com/network-speeds/) for selecting a baud rate.

## Network Protocol
The protocol appears to be [little-endian](https://en.wikipedia.org/wiki/Endianness#Little-endian) in its custom encoding formats. 

### Expected Traffic 
Controller units maintain their state internally. As such only _changed_ channel states need to be sent as they change. Some program implementations may choose to occasionally [resend existing state commands](https://github.com/smeighan/xLights/blob/master/xLights/outputs/LOROutput.cpp#L107) as a "sanity" measure. Resending state may result in visual glitches caused by resetting effect timers (such as fading) and increases bandwidth use.

## Heartbeat
Every 0.5s the LOR Hardware Utility sends a heartbeat payload onto the network. The exact value does not seem to matter as long as it within a 2 second timeout (this timeout is approximate).

If the controller unit has not recently received a heartbeat payload, it will consider itself not connected and become inactive. On my unit, this results in it not processing additional commands.

The heartbeat payload is a constant set of 5 magic bytes: `[0x00, 0xFF, 0x81, 0x56, 0x00]`

## Magic Numbers & Encoding Formats
Whether by design or by obfuscation the protocol contains several magic numbers and domain-specific encoding formats. Avoid duplicating these in code implementations as they may change.

### Brightness
Brightness is encoded as an unsigned byte between `0x01` (100% brightness) and `0xF0` (0% brightness). These values have been captured as output of the LOR Hardware Utility. Values outside these min/max bounds seem to result in indeterminate and unreliable behavior.

You can convert a given value (between [0, 1]) using:

`byteBrightness = floatBrightness * (maxBrightness - minBrightness) + minBrightness` 

...where `maxBrightness` and `minBrightness` are `0xF0` and `0x01` respectively. 

This example results in a linear brightness curve. Some program implementations, such as xLights, may use a [custom curve](https://github.com/smeighan/xLights/blob/master/xLights/outputs/LOROutput.cpp#L66). The brightness curve's behavior is up to the developer and is not restricted by the hardware beyond the previously specified min/max values.

### Durations
Durations are encoded in a unique [2]byte format. The method for encoding into this format has been derived from the assumption that durations have a minimum value of 0.1s and a maximum of 25s (as offered by the LOR Hardware Utility). While the LOR Hardware Utility only offers durations in increments by 0.1s (10 FPS), the encoding format does allow (while variable) additional precision however your mileage will vary depending on its usage.

Before being encoded, the value must be scaled. The scale used is exponential*ish* and offers higher precision for lower value durations. For example, the delta of the scaled values of 0.1s and 0.2s is 231.8x larger than the delta between the scaled values of 2.1s and 2.2s.

Given `timeSeconds` as a duration in seconds (such as 0.1s or 5s), you can calculate its scaled value as:

`scaledValue = round(maxDuration / (timeSeconds / minDuration)))` 

...where `maxDuration` and `minDuration` have values of `5099` (the decimal value of the captured hex encoded form of 0.1s, `[0x13, 0xEB]`) and `0.1` respectively. If `scaledValue <= 0xFF`, then it is encoded as `[0x80, scaledValue]` where `0x80` is a magic number representing an "empty" byte ignored by the unit. Scaled values larger than `0xFF` must be encoded in big endian format as a uint16.

#### Example Encodings
| Timing | Scaled Value | Encoded Format |
| - | - | - |
| 0.1s | 5099 | `[0x13, 0xEB]` |
| 0.5s | 1020 | `[0x03, 0xFC]` |
| 1s | 510 | `[0x01, 0xFE]` |
| 2s | 255 | `[0x80, 0xFF]` |
| 25s | 20 | `[0x80, 0x14]` |

As a reference, 2s is encoded as `[0x80, 0xFF]` (with a decimal value of `255`) which defines the break point in the previously mention encoding logic.

Any scale could technically be applied atop this behavior assuming the resulting encoded values stay within the assumed boundaries of `5099` (0.1s) and `20` (25s).

### Command IDs
Command IDs represent a predefined action for the controller to execute.

| Value | Name | Description |
| - | - | - |
| 0x03 | Set | Sets a channel's brightness |
| 0x04 | Fade | Fades a channel between two brightness values |
| 0x06 | Twinkle | Sets a channel to twinkling mode |
| 0x07 | Shimmer | Sets a channel to shimmer mode |

### Channel IDs
Channel IDs can be determined by taking the index of the channel (starting at 0, not 1) plus 128 (`0x80`). 

This can be easily implemented in code as `channelId = 0x80 | channelIndex`. 

## Single Channel Commands
Single channel commands exist within a parent structure containing routing data, a command ID, and **possibly** additional metadata relative with a format relative to the command ID. Commands which do not have additional metadata should omit the section entirely and should not send empty values.

### Parent Structure
| Field | Data Type | Notes |
| - | - | - |
| Header | byte | Always `0x00` |
| Controller ID | byte | |
| Command ID | byte | |
| Metadata | | Command ID specific metadata structure |
| Channel ID | byte | |
| End | byte | Always `0x00` |


### Metadata Structures
#### Set Brightness
| Name | Data Type |
| - | - |
| Brightness | byte |

#### Fade
| Name | Data Type |
| - | - |
| Start Brightness | byte |
| End Brightness | byte |
| Duration | [2]byte |

#### Set Twinkle
None.

#### Set Shimmer
None.

## Extended Commands
### Background Fade
Background Fade enables applying a foreground command atop a background command. This allows execution of multiple single channel commands simultaneously on the same channel ID.

| Field | Data Type | Notes |
| - | - | - |
| Header | byte | Always `0x00` |
| Controller ID | byte | |
| Foreground Command ID | byte | Only accepts `Twinkle` and `Shimmer` |
| Channel ID | byte | 
| Magic Number | byte | Always `0x81` (denotes extended command statement?) |
| Background Command ID | byte | Only accepts `Fade` |
| Start Brightness | byte | |
| End Brightness | byte | |
| Duration | [2]byte | |
| End | byte | Always `0x00` |

## Reference Implementations
- [xLights](https://github.com/smeighan/xLights) is a LOR-like C++ program which offers support for LOR controller units. 
- [go-lightorama](https://github.com/Cryptkeeper/go-lightorama) is a Go library implemented given this documentation.
