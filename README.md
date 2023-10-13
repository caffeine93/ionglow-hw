# IonGlow Nixie Tube Clock 

The idea behind development of this Nixie Clock is to have a modular system in which
the common *Control board* has the ability to control a wide range of swappable
*Display Tube boards* as well as daisy-chaining multiple of those (limited by the
current supplying ability of the power supply). This repo will only contain the
hardware design documents, the software for the clock will be provided as a separate
repo.

## Control Board

The Control board is separated into multiple functional blocks described in the
following subchapters.

### Power Supply

The power supply circuitry starts with the barrel jack connector through which the
input voltage of +12V DC is provided onto the board. The crowbar circuit is used
to limit the damage caused by possible overvoltage, either due to improper charger
plugin or a power surge, while the PTC (resettable) fuse also doubles to avoid
an overcurrent condition such a short circuit.

The +12V is used to provide voltage to the HV5522 Serial-In Parallel-Out High Voltage
Drivers on the *Display board* but also used to generate all the other needed voltages
for other subsystems.

The HV DC Boost converter uses the MAX1771 to generate the anode voltage needed to
pass the treshold voltage of the Nixie tubes in order for them to illuminate. The
precise value of voltage being generated can be fine tuned using the potentiometer
to allow for optimal selection for different Nixie tube models.

The +5V DC Buck converter uses the TPS62140 to produce voltage for the LEDs rail as well as
the LED Control drivers.

The +3.3V DC Buck converter uses the TPS62140 to produce the voltage for almost all the
digital logic subsystems on the board (ESP32, RTC, EEPROM, etc.)

### MCU

The ESP32 is the main controlling block of the clock, it's connected to a number of bus
devices for support functions as well as an antenna connector (with the matching Pi-network
circuitry) to allow for WiFi and Bluetooth connection to the clock. There's a button
connected for basic clock control as well as a buzzer for the alarm functionality.

### RTC

The DS3231 Real-Time Clock is used to keep track of time. It's connected to the ESP32
via I2C and also an IRQ line in order to allow for alarms and similar events to be
immediately available to the ESP32. The battery backup is not added since the clock
always synchronizes over NTP via WiFi on startup. The RTC chip is also internally
temperature-compensated, thus it won't run slower or faster not matter the temperature
of the clock nor the environment.

### EEPROM

There's an EEPROM chip added to allow for long-term storing of configuration of
the clock as well as production-related information. It's also connected to ESP32
via I2C.

### USB

The ESP32's UART is connected via CP2102 USB-UART bridge to the USB connector.
The purpose is both to allow UART for updating of the firmware on the ESP32
itself as well as connection via PC for configuration purposes. The USB connector
itself is protected by diodes against static shocks.

### Level Shifters

Different logic subsystems of the clock operate on different frequencies as explained
above, thus the level-shifting circuitry is used for in-spec commuinication. The
CD40109 is used for 3.3V->12V conversion for ESP32->HV5522 communication via SPI
while the 74VHC1GT125 is used for ESP32->TLC59731 communication over one-wire interface
utilizing the EasySet protocol.

## Display Board

The Display board contains the Nixie tubes themselves as well as LED background
illuminating circuitry. This board is much simpler than the Control board and has
different configurations depending on the tube type.

### Nixie Tube control

The Nixie tubes themselves are supplied with anode voltage via line feeding the
high voltage generated on the control board. The cathodes are connected to the
HV5522 Serial-In Parallel-Out HV driver which will either connect the cathode
to the GND or not, depending on the shift-register being fed over SPI by ESP32.
The chips are daisy-chained via the connector and when the internal shift
register "overflows", it's fed out of the *DATAOUT* pin into the next HV5522
on the next Display board. The tubes are directly driven (non-multiplexed)
to extend their lifetime. The brightness of the tube is a function of the
anode current, calculated as I = (Ua - Ut)/Ra where:

- I = anode current
- Ua = anode voltage
- Ut = threshold voltage of the Nixie tube (specified in its datasheet)
- Ra = anode resistor value

The value of the anode resistor is fixed, but the brightness can still be controlled
by applying PWM pulsing over the */BL* (blanking) pin on the HV5522 which
disconnects all the cathodes from GND, thus turning them off. The duty cycle
of blanking interval ON/OFF can thus control the tube's brightness.

### LEDs control

The LEDs are controlled by the TLC59731 LED driver chip which is controlled
over a one-wire interface using the EasySet protocol from the ESP32. The brightness
of each channel (red, green, blue) can be set in increments of one in interval of
0-255. These chips are also daisy-chain compatible and, having an internal signal
amplifier, do not suffer even when very long chains are connected. The large
electrolytic capacitor serves to provide current when the sudden sink by LEDs
appears on the +5V rail either on power-on or due to rapid intensity change.
