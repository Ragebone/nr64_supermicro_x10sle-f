# How to flash the IC of the x10sle-f BIOS and BMC-Firmware

## Hardware requirements 

* At least one SPI programmer, preferrably a CH341A spi programmer, as this howto will base on that. 
* SOP8 Chip Grabber 
* SOP16 Chip Grabber 
* A few female to female dupont wire jumper cable 
* A fiber cleaning pin 
* a computer with linux (can be a Raspberry pi or similar types as well) 

If you like, you could probably get away with just an SPI programmer and a linux computer. 
(Heck, there is even the chance to use a [raspberry pi to flash via SPI](https://libreboot.org/docs/install/rpi_setup.html)!), but I would recommend to buy these chip grabbers while ordering the CH341A spi programmer, as they are very low cost and can help you in other situation with firmware/bios issue on notebooks, tablets and PCs also very well. 

Lets start with the CH341A spi programmer:

