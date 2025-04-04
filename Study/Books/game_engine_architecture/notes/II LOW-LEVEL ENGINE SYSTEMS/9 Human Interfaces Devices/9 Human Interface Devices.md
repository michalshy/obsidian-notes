Chapter provided for description about different ways in which engines process and utilize inputs from human interface devices.
# Types of devices
Many of them, including joypads, controllers, keyboard and mouse, wiimote.
# Interfacing with HID
Design and device dependent implementation.
Usually manages input, but might provide output for user.
## Polling
Reading state of device periodically (every frame?).
Conducted via reading hardware registers or reading memory-mapped I/O port, or via higher-level software interface.
Likewise, output by writing to those registers etc.
## Interrupts
Sometimes communication of state change should be conducted only on change. For example in case of mouse, there is no reason to send constant data stream.
This is implemented via *hardware interrupt.*
## Wireless devices
In case of wireless devices, there is no way to write into registers. Devices must talk via Bluetooth protocol.
This communication is usually handled by separate thread, or at least by encapsulated simple interface which may be called from main thread.
582