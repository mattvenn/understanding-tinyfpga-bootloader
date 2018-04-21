# Understanding the TinyFPGA bootloader.

This bootloader was written by Luke Valenty. The TinyFPGA website is
http://tinyfpga.com/

I am learning verilog with the new open source Icestorm tools from Clifford Wolf.
http://www.clifford.at/icestorm/

I was very interested in the bootloader and it seemed a good way to further my
understanding of Verilog, FPGAs and USB to study it. 

Thanks to Luke for publishing his work and Clifford for the tools for us to
learn and use this amazing hardware.

The bootloader repo is here https://github.com/tinyfpga/TinyFPGA-Bootloader

# Aim

Aim of the bootloader is to allow easy and cheap transmission of FPGA configuration files
over USB to get stored in FPGA connected SPI FLASH memory. Data is sent over an
emulated serial port. 

On other boards a moderately expensive serial to usb chip
(like the FTDI ones) is used. One exception is the BlackIce board that uses an
ARM chip to provide USB comms. The FPGA configuration is stored on the ARM
chip's EEPROM.

# Overview

USB is a very complex protocol, and implementing it all on a small FPGA would be
difficult. However, by limiting the requirements it becomes more approachable.

Limitations of the bootloader:

* full speed (12MHz) - means the FPGA can communicate directly. High speed requires current drivers.
* limited number of endpoints - configured with NUM_OUT_EPS and NUM_IN_EPS
* 2 types of endpoint - control and serial.
* after configuring is finished, don't support changes to things like baud and line encoding
* limited data buffer length: set to 32 bytes - MAX_IN_PACKET_SIZE parameter

How it works:

The top module is tinyfpga_bootloader.v. It controls the LED and sets up the USB
engine. It instaniates:

* the usb protocol engine - connected to the below pair of in and out endpoints
* the usb serial and control endpoint (ctrl in and out endpoint)
* the spi bridge (serial in and out endpoint)

The USB protocol engine is defined in usb_fs_pe.v. It handles the low level USB
and ends up with endpoint interfaces.

Endpoints can be thought of as buffers. They have a limited amount of space and
need to be able to deal with running out of space or not having data to read.

It uses a number of modules to:

* decode and encode the data onto the wires, checksums: usb_fs_rx/tx
* data transfer from/to endpoints: usb_fs_out/in_pe
* endpoint arbitration:usb_fs_in/out_arb

USB is configured by the host communicating with endpoint id 0. The control transfer state machine
in usb_serial_ctrl_ep.v sets up the USB device as an emulated serial port with 3 endpoints: 

* ACM (would be used for configuring serial port)
* TX
* RX

After configuration is finished only RX and TX endpoints are supported. 

The SPI memory controller is defined in usb_spi_bridge_ep.v. It is connected to
the serial RX and TX endpoints for reading and writing the FLASH. It uses 2
state machines.

The SPI state machine manages reading and writing to FLASH.

The command sequencer state machine receives commands and data from the
serial_out endpoint. It can start a transfer to FLASH or boot the FPGA. 

SPI MISO from FLASH is put straight into the serial_in endpoint.

# Resources

* verilog: https://github.com/tinyfpga/TinyFPGA-Bootloader
* schematic: https://github.com/tinyfpga/TinyFPGA-BX/blob/master/board/TinyFPGA-BX-Schematic.pdf
* usb in a nutshell http://www.beyondlogic.org/usbnutshell/usb1.shtml
  * control transfers: https://beyondlogic.org/usbnutshell/usb4.shtml#Control
* final year project of a USB stack implemented in FPGA http://engineering.biu.ac.il/files/engineering/shared/PE_project_book_0.pdf
* another good overview of USB https://www.keil.com/pack/doc/mw/USB/html/index.html

# Notes on coding style

* most important modules further up in top
* consistent naming convention
* parameterised (num endpoints and buffer data length)
* great test suite
