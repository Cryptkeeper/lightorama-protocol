# Light-O-Rama Protocol
This documentation covers the Light-O-Rama (LOR) communication protocol for [AC lighting control units](http://www1.lightorama.com/pro-ac-light-controllers/). It has been manually reverse engineered using the [LOR Hardware Utility](http://www.lightorama.com/help/index.html?hardware_utility.htm), serial port monitoring software and `LOR1602Wg3` & `CTB16PCg3` units. Given the reverse engineered nature, this documentation should be considered incomplete, incorrect and outdated. 

# Useful Links
### Reference Implementations
* [xLights](https://github.com/smeighan/xLights) is a LOR-like C++ program which offers support for LOR units. 
* [fpp](https://github.com/FalconChristmas/fpp/blob/master/src/channeloutput/LOR.cpp) is a sequence scheduling and player program which supports LOR output.
* [liblightorama](https://github.com/Cryptkeeper/liblightorama) is a C90 library implementing the LOR protocol and brightness curves.
* [go-lightorama](https://github.com/Cryptkeeper/go-lightorama) is a Go library implemented given this documentation.

### Light-O-Rama Resources
* [Firmware updates](http://www1.lightorama.com/firmware-updates/)
* [User guides and datasheets](http://www1.lightorama.com/documentation/)
* [Network configuration examples](http://www1.lightorama.com/typical-setups/)
* [Baud rate selection](http://www1.lightorama.com/network-speeds/)
* [LOR Hardware Utility guide](http://www.lightorama.com/help/index.html?hardware_utility.htm)
* [LOR_DeviceFile.txt](http://www1.lightorama.com/downloads/LOR_DeviceFile.txt)

# Test Environment
From my testing, the protocol seems consistent across firmware versions and is unlikely to change. You can view your current firmware version in the [LOR Hardware Utility](http://www.lightorama.com/help/index.html?hardware_utility.htm). I have provided my firmware and unit models for reference.

| Unit Model | Firmware Version | Channel Count |
| --- | --- | --- |
| 1x `LOR1602Wg3` | 1.12 | 16 |
| 1x `CTB16PCg3` | 1.09 | 16 |
| 3x `CTB16PCg3` | 1.11 | 16 |

Each of my units have 16 channels. I have kept alternate channel count configurations in mind while documenting the protocol and as such you should have no issues applying it to other network configurations.

I am using a 57.6K baud rate and the Light-O-Rama provided [USB-RS485 adapter](http://store.lightorama.com/uscoad.html).

# Network
## Fault Tolerance
Messages are generally "fire and forget". Your software implementation SHOULD avoid resending previous messages since it may interrupt ongoing effect executions (like restarting a timer), resulting in channel output glitches.

Some software implementations may choose to ocassionally [repeatedly resend messages](https://github.com/smeighan/xLights/blob/master/xLights/outputs/LOROutput.cpp#L107) as a "sanity" measure. However this mostly wastes network bandwidth and risks the aforementioned glitches. If you choose to repeatedly resend messages, avoid sending messages which execute over a period of time.

## Heartbeat Behavior
Every 500ms the LOR Hardware Utility broadcasts a heartbeat message onto the network. This message is not unit specific and should only be sent once per "heartbeat". 

If a unit has not received a heartbeat message within 2 seconds, it will mark itself not connected and become inactive. This is indicated by the hardware with a message on its LCD (if any), or a blinking LED.

To "re-activate" the units, simply restart sending the heartbeat message.

### Notes
* Units do not mind changes in heartbeat message frequency as long as it is within the timeout interval.
* Some units may require several heartbeats to pass before re-activating. Software implementations MAY wish to wait 1-5 seconds when starting to ensure all units have been activated.

## Empty Bytes
An empty byte (`0x00`) appears to be used to execute each unit's incoming network buffer. **You must send these empty bytes, otherwise units will never execute the messages you've sent.** 

It is RECOMMENDED you prefix messages with an empty byte to ensure the buffer is flushed prior to the incoming message, improving fault tolerance by preventing corrupt/invalid messages from remaining in the buffer. Appending an additional empty byte to the end of the message means it will be executed as soon as the empty byte is received.

### Notes
* `0x00` does not otherwise appear in the protocol, even as a value.
* To reduce bandwidth usage, you may be able to omit the empty byte in some places. The LOR Sequencer appears to automatically do this when sending multiple messages. If you find your unit is not executing messages properly, consider introducing additional empty bytes.
* In my monitoring I have seen empty bytes consistently sent over the network. This may be either to either flush an outbound queue, or to simply ensure that even with data loss, the message buffer is still being flushed.

# Data Types & Encoding Formats
## Unit IDs
The LOR protocol supports up to 240 *unique* unit IDs. Given that `0x00` is effectively unavailable namespace, the valid unit ID range is `0x01` to `0xF0` (240). Any unit ID above `0xF0` is reserved, often used for routing information.

Since messages are not consumed by units, multiple units may have the same unit ID. They would simply act as clones of each other when receiving messages. This is however discouraged since software implementations may not properly handle multiple responses to messages direct at a single unit ID.

| Value | Purpose |
| --- | --- |
| `0xFF` | "Any unit ID", used for broadcasting messages |
| `0xFE` | Represents the controlling program, used for replying to messages |
| `0xFB` | Used by the LOR Hardware Utility when uploading MP3 & sequence files |
| `0xFA` | Used within an unknown message sent by the LOR Hardware Utility when the program is first launched |
| `0xF1` | Used by the LOR Hardware Utility when setting "Any" unit's ID |

**Tip:** Wherever a unit ID is specified, you may choose to subsitute it for the `0xFF` broadcast flag. This makes it easy to avoid manually sending the same message to multiple units.

## Channel IDs
While channel IDs start at 1 within LOR software and user guides, they start at 0 within the protocol. This documentation will also consider channel IDs as starting at 0. The following code sample shows how to calculate a protocol equivalent of a channel ID by ORing it, as well as reversing this by taking its AND of NOT `0x80`. Channel numbers greater than or equal to 127 (the max value that can fit a `uint8` with the `0x80` offset) should be encoded as multiple bytes in [big endian format](https://en.wikipedia.org/wiki/Endianness). Each byte should still be offset by `0x80`.

```
uint8_t to_protocol_channel(uint8_t channel) {
	return channel | 0x80;
}

uint8_t from_protocol_channel(uint8_t protocol_channel) {
	return protocol_channel & ~0x80;
}
```

## Brightness
Brightness is encoded as a `uint8` between `0x01` (100% brightness) and `0xF0` (0% brightness). Values outside these min/max bounds seem to result in indeterminate and unreliable behavior.

You can convert a normalized value [0, 1] into a brightness value like so:

`brightness_value = normalized_value * (0x01 - 0xF0) + 0xF0`

### Notes
* The provided code sample results in a linear brightness curve. Some program implementations, such as xLights, may use a [custom curve](https://github.com/smeighan/xLights/blob/master/xLights/outputs/LOROutput.cpp#L66) function. The brightness curve's behavior is up to the developer and is not restricted by the hardware beyond the previously specified min/max values.
* These values have been captured as output of the LOR Hardware Utility. 
* The [datasheet](http://www1.lightorama.com/PDF/LOR160xWg3_Datasheet.pdf) for my `LOR1602Wg3` unit specifies "100 intensity levels for dimming". So while additional brightness precision is supported within the protocol, your hardware may disregard it.

## Durations
As displayed within the LOR Hardware Utility (and cross validated in the [datasheet](http://www1.lightorama.com/PDF/LOR160xWg3_Datasheet.pdf) for my `LOR1602Wg3` unit), durations have a minimum value of 0.1s and a maximum of 25s with 256 available levels between those values.

Durations are encoded as a `uint16` as a scaled value against the max and min limits. For durations that fit within the lower byte, the upper byte MUST be set to `0x80` so that it is ignored by the unit.

The following code sample accomplishes this by ORing the `uint16` value against `0x80` shifted left by 8 bits. The magic value `5099` is derived from the encoded representation of the minimum duration, `0.1s`.

```
// This example assumes seconds value is already within the acceptable range
uint16_t lor_duration(float seconds) {
    const uint16_t scaled_value = (uint16_t) roundf(5099 / (seconds / 0.1F));
    if (scaled_value > 0xFF) {
        return scaled_value;
    } else {
        return 0x8000 | scaled_value;
    }
}
```

### Examples
| Duration | Scaled Value | Encoded Value |
| --- | --- | --- |
| 0.1s | 5099 | `0x13EB` |
| 0.5s | 1020 | `0x03FC` |
| 1s | 510 | `0x01FE` |
| 2s | 255 | `0x80FF` |
| 25s | 20 | `0x8014` |

### Notes
- Any duration greater than 2 seconds fits within a single byte.
- The scale used is exponential*ish* and offers higher precision for lower value durations. For example, the delta of the scaled values of 0.1s and 0.2s is 232x larger than the delta between the scaled values of 2.1s and 2.2s.
- Any scale could technically be applied atop this behavior assuming the resulting encoded values stay within the value range. Your unit however may disregard or round this value - [consult your unit's datasheet](http://www1.lightorama.com/documentation/).

## Actions
Actions are predefined lighting effects, typically built into the hardware. These are what make up the majority of the network traffic since they are the effective purpose of the software and hardware. An action defines what a target should do, but does NOT specify the target directly. Instead, actions are wrapped into messages, which specify the target, the action, and any metadata required by the action.

| Name | Value | Description | Has Metadata |
| --- | --- | --- | --- |
| On | `0x01` | Sets a channel to 100% brightness | No |
| Set Brightness | `0x03` | Sets a channel's brightness | Yes |
| Fade | `0x04` | Fades a channel between two brightness values | Yes |
| Set Twinkle | `0x06` | Sets a channel to twinkling mode | No |
| Set Shimmer | `0x07` | Sets a channel to shimmer mode | No |

### Metadata Structures
#### Set Brightness
| Name | Data Type |
| --- | --- |
| Brightness | `uint8` |

#### Fade
| Name | Data Type |
| --- | --- |
| Start Brightness | `uint8` |
| End Brightness | `uint8` |
| Duration | `uint16` |

# Messages
## Heartbeat
| Field | Data Type | Value | Notes |
| --- | --- | --- | --- |
| Unit ID | `uint8` | `0xFF` | Denotes a broadcast message |
| Configuration Update | `uint8` | `0x81` | Commonly seen used in messages relating to configuration data |
| Heartbeat | `uint8` | `0x56` | |

## Get Unit Version
Fetching the firmware version requires a message to be sent and a response received. The response message does not contain a unit ID. As a result, for software implementations looking to scan the network, you must wait a period of time after each sent message for a response, before moving on to the next unit ID.

### Request
| Field | Data Type | Value | Notes |
| --- | --- | --- | --- |
| Unit ID | `uint8` | `0xFF` | |
| Firmware Request Type | `uint8` | `0x88` | Seem in firmware upload messages |
| Data Request Type | `uint8` | `0x29` | Variations of this message have been seen with values `0x2A` and `0x2B` when changing/setting unit IDs |
| | `uint8` | `0x2D` | |

### Response
| Field | Data Type | Value | Notes |
| --- | --- | --- | --- |
| Response | `uint8` | `0xFE` | `0xFE` is used for routing messages back to the controlling software |
| Data Response Type | `uint8` | `0x29` | Matches value in request message |
| Unit Type | `uint8` | | Unit hardware model, correponds to the "Type" column of the [LOR_DeviceFile.txt](http://www1.lightorama.com/downloads/LOR_DeviceFile.txt) | 
| | `uint8` | `0x81` | |
| | `uint8` | `0xFF` | |
| Firmware Version 1 | `uint8` | | ASCII value of the firmware's 1st minor version number |
| Firmware Version 2 | `uint8` | | ASCII value of the firmware's 2nd minor version number |
| | `uint8` | `0x80` | |
| | `uint8` | `0x03` | Potentially unit ID |
| | `uint8` | `0x03` | Potentially unit ID |
| | `uint8` | `0xF0` | |

For example firmware version "1.09" has minor version numbers of 0 and 9. These are sent as their ASCII values (0x30 & 0x39). Only the minor version numbers seem to be included in the response message. This is supplemented by the fact that the [LOR_DeviceFile.txt](http://www1.lightorama.com/downloads/LOR_DeviceFile.txt) hardcodes the major version into the "Description" column.

### Notes
* This is used by the LOR Hardware Utility when determining if a unit ID is in use for ID changes, or when hitting the "Refresh" button and scanning the network.
* The LOR Hardware Utility will send this 5 times, with a delay between each request, for each unit ID it is checking until it gets a response. If it has not received a response after all 5 requests, it considers the unit ID unused or absent.

## All Off
Turns off all channels attached to the unit ID without any filtering capability. 

**Tip:** Unit ID `0xFF` will broadcast this to all units. This allows you to easily turn off EVERYTHING with a single message.

| Field | Data Type | Value |
| --- | --- | --- |
| Unit ID | `uint8` |
| Action | `uint8` | `0x41` |

## Background Fade
Background Fade enables applying a foreground action atop a background action. This allows execution of multiple single channel actions simultaneously on the same channel ID. The structure is equivalent to the two actions joined with a `0x81` magic number.

| Field | Data Type | Value | Notes |
| --- | --- | --- | --- |
| Unit ID | `uint8` | | |
| Foreground Action | `uint8` | `0x07` | Only works with `Set Shimmer` |
| Channel ID | `uint8` | | |
| Magic Number | `uint8` | `0x81` | Denotes extended message? |
| Background Action | `uint8` | `0x04` | Only works with `Fade` |
| Start Brightness | `uint8` | | |
| End Brightness | `uint8` | | |
| Duration | `uint16` | | |

### Notes
* This has not been tested with broadcast or channel masking behavior.
* While other action values may be used, only `Set Shimmer` and `Fade` produce results without clashing.

## Single Channel Action
Executes an action (and its metadata, if any) on the single specified channel. If broadcasted, this will apply to the specified channel ID of *each* unit (assuming it has enough channels).

| Name | Data Type |
| --- | --- |
| Unit ID | `uint8` |
| Action | `uint8` |
| Action Metadata | |
| Channel ID | `uint8` |

## Multi Channel Action
Multi channel actions utilize the same actions and metadata structures as their single channel equivalents with the exception that instead of sending a `uint8` channel ID a channel bitmask is sent and the channel mask length is included as part of the action value. For units with more than 16 channels, multiple messages are sent instead, each providing a portion of the full channel mask.

| Field | Data Type | Notes |
| --- | --- | --- |
| Unit ID | `uint8` | |
| Action | `uint8` | Action value OR'd against a magic number |
| Action Metadata | | |
| Chain Index | `uint8/-` | Only sent when chaining multi channel messages due to the full channel configuration being unable to fit within a single channel mask |
| Channel Mask | `uint8[]` | Length is dependent on channel count, channels with a 0 value bit are ignored and maintain their previous state |

### Channel Masking
The channel mask for channels 1, 7 and 14 in binary is `0010 0000 0100 0001`.

```
uint16_t mask = 0;
mask |= 1 << 0 // set bit index 0 (channel 1) to 1
mask |= 1 << 6 // set bit index 6 (channel 7) to 1
mask |= 1 << 13 // set bit index 13 (channel 14) to 1
```

The resulting channel mask is 16 bits long and is encoded as `0x2041`.

To determine the magic number used, round the length of your channel mask up to the nearest multiple of 8, select the magic number from the table below, and OR it against your action value.

| Channel Mask Length | Mask | Notes |
| --- | --- | --- |
| 8 bits | `0x30` | |
| 16 bits | `0x10` | **Tip:** `0x10` has a decimal value of 16 |
| 16 bits | `0x50` | Only used when chaining messages |

```
uint8_t get_multi_channel_action(uint8_t action, uint8_t channel_mask_length) {
	if (channel_mask_length == 8) {
		return 0x30 | action;
	} else if (channel_mask_length == 16) {
		return 0x10 | action;
	} else {
		// For channel mask lengths greater than 16, multiple messages are sent instead.
		// A magic number of 0x50 is used, regardless of channel mask length
		return 0x50 | action;
	}
}
```

In the previous example, 16 is already a multiple of 8, so a `0x10` magic number is selected. A "Fade" action (`0x04`) would be OR'd against the selected magic number (`0x10`), resulting in a final action value of `0x14`.

### Chaining Example
Given a unit ID of `0x01` with 64 channels, we can set every channel to 0% brightness using 4 multi channel messages. The chain index is a reverse index, starting at the highest index plus `0x01`, ending at `0x01`. It must not be `0x00`, which marks the end of a message.

```
# Set Brightness of channels 0-15 (padding bytes omitted)
0x01 (unit ID)
0x53 (Chaining channel mask magic number | 0x03 Set Brightness action)
0xF0 (0% brightness)
0x03 (chain index of 3)
0xFF (channel mask pt. 1)
0xFF (channel mask pt. 2)
```

Resending this message with a chain index of 2 and 1 would provide channel masks for channels 16-31 and 32-47 respectively. To end the chain, a final message is sent with an _unchained_ action value offset and without a chain index.

```
# Set Brightness of channels 48-63 (padding bytes omitted)
0x01 (unit ID)
0x13 (16-bit channel mask magic number | 0x03 Set Brightness action)
0xF0 (0% brightness)
0xFF (channel mask pt. 1)
0xFF (channel mask pt. 2)
```

#### Notes
- Channel mask lengths can be mixed. For example, 40 channels could be sent as 2x 16 bit masks and 1x 8 bit mask.
- Units safely handle channel masks larger in length than the channel count, however they should be avoided to optimize bandwidth and processing speed.

# Undocumented
These messages have been captured, and are noted here to help cross reference recurring magic numbers.

```
# LOR Hardware Utility Boot
Sent 5 times by the LOR Hardware Utility when booting.
0xFA 0x88 0x31 0x2D

# Edit Mode
Appears whenever modifying a unit's configuration or updating firmware.
$UNIT_ID 0x8A 0x56

# Firmware upload
Various captured stages of the firmware update process

fe cb da 01 ff ff ff ff ff 01 <- Initial channel configuration
fe cb da 02 ff 0b ff ff ff 01 <- Firmware blob (n...)
fe cb da 03 ff 08 ff ff ff 01 <- Last firmware blob

0x01, 0x02, 0x03 denote upload stage

0x01 = Start
0x02 = Data part
0x03 = Finish

Each firmware blob part is sent with the message header 0xFF 0x89 0x02.
Units respond to each blob part with:

fe cb da 02 ff 0b ff ff ff 01

0xDA = 218, Bootloader version?
```

## Notes
* `#FILE` marker within firmware files is the last 4 bytes of Fletcher-64 checksum of the re-assembled firmware file.
* Firmware files get split at carriage returns (`0x0D 0x0A`). Heartbeats are sent in their place. Each carriage return denotes a "blob", with the last blob placed first in the file.
* Firmware values have some sort of bitpacking encoding. They do not contain values [0,32][35][251-255].
