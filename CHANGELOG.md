# Changelog

## 0.1.0

The first release: a driver for the MAX32664 biometric sensor hub of Maxim
(Analog Devices), the A variant (MaximFast firmware, MAX30101 sensor), on an I2C
bus.

- `Max32664`, the type a board's device table builds from the board's I2C bus,
  timer and GPIO handles, the wiring (RSTN is `en`, MFIO is `irq`) and a
  `Max32664Config`, implementing devices-nv's `HeartRate` and `Identifiable`.
  `Max32664Port` is the trait a test or a program implements to run the module's
  functions over its own bus.
- The command protocol of the User Guide (UG6806 Rev 0): the family and index
  frame, the status byte of Table 4 as its own value, the 2 ms and 45 ms waits, a
  retry of a busy hub, and a check of the count of bytes the bus returned.
- The RSTN and MFIO sequences into application and bootloader mode, with MFIO
  released as the interrupt in application mode, and a poll of the mode read for
  up to 1.7 s while the firmware initialises.
- The sensor hub status bits, the device mode, the sensor hub, algorithm and
  bootloader versions, the bootloader page size and the microcontroller type.
- The output mode and FIFO threshold, the MAX30101 and automatic gain control
  and algorithm enables, MAX30101 register reads and writes, and the start and
  stop of sampling in the order of the guide's trace.
- The output FIFO of the A variant: the four LED counts and the report (heart
  rate in 0.1 bpm, confidence, SpO2 in 0.1 per cent, state), the extended report
  of the 10.x firmware, one sample per transaction, decoded against the guide's
  trace, and `HeartRate`'s drain with time stamps.
- The device program `examples/hr_reader.nv`, a heart rate and SpO2 reader.
  Built for the nRF52840-DK it is 21,192 B of flash and for the nRF54LM20-DK
  22,576 B; before the Nordic buses landed, with the reading path left out, it
  was 13,872 B and 15,088 B.

### Known

- Hardware verification: NOT VERIFIED. No board on the bench carries the hub.
  The nRF52840 DK drives I2C with TWIM0 and the nRF54LM20 DK with TWIM22, so
  the device program's images carry the reading path; a reply from the hub over
  either bus is NOT VERIFIED.
- The B, C and D variants are NOT IMPLEMENTED; `config_ok` refuses them. Their
  names and algorithms are from the titles of Analog Devices' application notes,
  which could not be fetched.
- Not implemented: flashing the application through the bootloader, the
  algorithm tuning commands (families 0x50 and 0x51), the sample counter output
  modes, the external accelerometer, the other sensors' commands, the register
  dump, the sleep modes and the async forms.
- The extended report layout is from SparkFun's library and is not in User Guide
  Rev 0; the order of the LED fields against the MAX30101's slots and the read of
  a single sample from the A firmware's FIFO are NOT VERIFIED on a part.
- `devices-nv` is the registry range `^0.3.3`, the first release with the
  `HeartRate` trait. `embedded-hal-nv` is the registry range `^0.3.2`.
- The `[tier]` of the manifest is `embedded`, the strictest, so the module also
  checks at `rt` and `system`.
