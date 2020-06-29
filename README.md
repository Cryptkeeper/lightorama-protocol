# lightorama-protocol
This repository contains manually reverse engineered documentation for various layers of the [Light-O-Rama](http://www.lightorama.com) software & hardware ecosystem, including the communication protocol and some file formats.

The goal of this project is to improve open-source support for these file formats, protocols and devices and prevent data rot. Contributions are welcomed.

It is incomplete and should be considered unsupported. Although risky, this documentation is largely up to date due to the annual software release cycle and the difficulty in releasing new versions of hardware products.

* [Light-O-Rama communication protocol](PROTOCOL.md)
* [Light-O-Rama "Compressed Sequence (.lcs)" file format](LCS.md)

## Useful Links
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
