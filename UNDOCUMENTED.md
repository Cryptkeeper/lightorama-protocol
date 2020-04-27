# Undocumented Packet Captures
These are packet captures with a given context that have not yet been fully documented.

### Call System
The protocol contains a broadcast and call request system, which can be found when using the LOR Hardware Utility to configure unit IDs or search for connected units. Presumably there is an acknowledgement response to many of these packets, but it is currently undocumented.

**WARNING: This behavior is _especially_ underdocumented.**

#### Is Unit Connected
Requests an acknowledgement response from a controller with the specified ID. When refreshing connected units, the LOR Hardware Utility will send this packet to each connected serial port, iterating from unit ID `0x01` to the "Max Unit ID" as specified in the LOR Hardware Utility. Each packet will be sent 5 times, per unit ID, per connected serial port.

| Field | Data Type | Notes |
| - | - | - |
| Header | `uint8` | Always `0x00` |
| Controller ID | `uint8` | |
| Magic Number | `uint8` | Always `0x88` |
| Magic Number | `uint8` | Always `0x29` |
| Magic Number | `uint8` | Always `0x2D` |
| End | `uint8` | Always `0x00` |

#### Set Unit ID
Sets _ALL_ connected unit IDs to a specified value. Given unit IDs have a maximum value of `0xF0` (240), the usage of `0xF1` (241) as a controller ID implies it may have a usage as an "any" unit ID. Conflictingly however, `0xFF` is also used by the heartbeat payload as a broadcast flag.

| Field | Data Type | Notes |
| - | - | - |
| Header | `uint8` | Always `0x00` |
| Controller ID | `uint8` | Always `0xF1` |
| Magic Number | `uint8` | Always `0x88` |
| Magic Number | `uint8` | Always `0x2A` |
| Magic Number | `uint8` | Always `0x32` |
| End | `uint8` | Always `0x00` |

#### Change Unit ID
Changes the ID of a unit, given its current ID and the new ID. This is used to safely change the ID of a specific unit, even with multiple controllers connected.

| Field | Data Type | Notes |
| - | - | - |
| Header | `uint8` | Always `0x00` |
| Current Controller ID | `uint8` | |
| Magic Number | `uint8` | Always `0x88` |
| Magic Number | `uint8` | Always `0x2A` |
| Magic Number | `uint8` | Always `0x2D` |
| End | `uint8` | Always `0x00` |

## Toggle?
```
Unit: 		0x01
Channel: 	17
On: 		0x00 0x01 0x51 0x41 0x01 0x00 0x00
Off: 		0x00 0x01 0x51 0xc1 0x00 0x00
```

```
Unit:		0x01
Channel: 	18
On:			0x00 0x01 0x51 0x41 0x02 0x00 0x00
Off:		0x00 0x01 0x51 0xc1	0x00 0x00
```

```
Unit:		0x01
Channel:	129
On:			0x00 0x01 0x51 0x48 0x01 0x00 0x00
Off:		0x00 0x01 0x51 0xc8 0x00 0x00
```
128 (`0x80` channel ID base) + 129 (channel) = 257 - 1 (index offset) = 256

256 = `0x100` = `[0x01, 0x00]`

```
Unit:		0x01
Channel:	130
On:			0x00 0x01 0x51 0x48 0x02 0x00 0x00
Off:		0x00 0x01 0x51 0xc8 0x00 0x00
```

`0x51` appears to denote the usage of a `0x01` (On) command in a multi channel usage (currently only `0x30` and `0x10` are documented, but since they only support up to 16 bits, this likely means `0x50` denotes a higher bit count.)

Each `Off` command does not appear to contain channel IDs, instead a value of `0xc7` or `0xc8`. Additionally, if the usage of `0x51` is believed to be a multi channel command usage of `0x01`, then we should not be seeing these commands turn the channels off, and instead maintain their _on_ state.

## All Off
`0x00 0x01 0x41 0x00 0x00`
Used to turn off all 8 channels in an 8 channel configuration.

## All On?
`0x00 0x0 0x31 0xff 0x00 0x00`
Used to turn on all 8 channels in an 8 channel configuration.

