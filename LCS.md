# Light-O-Rama Compressed Sequence File Format
The _[Light-O-Rama](http://www.lightorama.com) Compressed Sequence_ file format (".lcs") is a compressed version of the XML based Light-O-Rama sequence file format (".lms" and ".las"). The Light-O-Rama Compressed Sequence file format uses a custom compression scheme and file structure, with the majority undocumented.

See the ["Compressed Sequences"](http://www.lightorama.com/help/index.html?concept_compressed_sequences.htm) Light-O-Rama help guide page for more information.

## Header
The file format is identified by a decorative 415 byte ASCII header starting at index 0. Besides this header, there are no other magic bytes to identify the file format.

```
***************************************************
*                                                 *
* This file is a compressed sequence for use with *
*        the Light-O-Rama software suite.         *
*                                                 *
*           http://www.lightorama.com/            *
*                                                 *
***************************************************
```

The header will be followed by one `0x0A` NL character (for a total of 416 bytes).

A `0x02` STX character marks the start of a series of seemingly always NUL bytes, with a variable length. An additional `0x01` SOH character marks the start of the file's metadata fields. As such, it's recommended software implementions simply seek to the first occurance of `0x01` to begin reading.

## Metadata Fields
Following the header, the metadata fields structure is used to store authorship and timestamp information.

Each metadata field starts with a `uint16` value indicating its *character* length (not byte length) encoded as UTF-16 LE, followed by a `uint16` NUL value, and then the encoded metadata field value.

The metadata field values are not null terminated and instead continue instantly to the next `uint16` character length value (if any).

### Encoding Example
The following encodes a value of "HELLO" in 14 bytes.

```
0x05 0x00	// uint16 character length value (5)
0x00 0x00	// uint16 NUL value
0x72 0x00	// UTF-16 LE "H"
0x69 0x00	// ..."E"
0x76 0x00	// ..."L"
0x76 0x00	// ..."L"
0x79 0x00	// ..."O"
```

A metadata field may be "omitted" by encoding a character length value of 0. Keep in mind, the `uint16` NUL value is still required. This means each empty metadata field will use 4 bytes.

### Metadata Field Order
The metadata fields are not keyed, and instead a fixed encoding order is used in order to interpret the value of each metadata field. Unfortunately this means the purpose of each field is static, and the addition of new fields is unsupported and dangerous. 

#### Encoding Order
1. File path of the media file, if any
2. File author
3. File creation timestamp
4. Last file update author
5. Song album
6. Song artist
7. Song title
8. Unknown, always empty, likely reserved for future use

Empty metadata field values must still be written, otherwise the official Light-O-Rama software will misinterpret the values and potentially attempt to read sequence data as metadata fields.

## Sequence Data
The sequence data appears to be aligned in 40 byte blocks (this may be a result of my configuration), prefixed by a `uint16` value containing the centiseconds (0.01 seconds) value used by the Light-O-Rama [timing grid](http://www.lightorama.com/help/index.html?concept_compressed_sequences.htm).
