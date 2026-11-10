# max32664-nv

A driver for the MAX32664, the biometric sensor hub of Maxim (now Analog
Devices): a microcontroller pre-programmed with a bootloader that loads Maxim's
firmware, which reads a pulse-oximetry sensor (the MAX30101 on the A variant),
runs an algorithm on its samples, and hands the host a heart rate, an SpO2
value and a confidence level over I2C. The document is the "MAX32664 User
Guide", UG6806; Rev 0; 01/19, which gives the pin sequences that start the
application or the bootloader, the I2C message protocol (its Table 5), the
status byte, the sensor hub status byte, the output FIFO format and an
annotated trace of a MAX32664GWEA streaming data. The driver implements the
`HeartRate` and `Identifiable` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv) and, beside them, reads
the hub's own report: heart rate, confidence, SpO2 and state.

**Hardware verification: NOT VERIFIED.** No board on the bench carries this
part, so the driver has run only against a double of the hub, in the host
suite. The device program below builds for the nRF52840 DK and the nRF54LM20 DK
and has not been flashed.

## Install

```sh
novo pkg add max32664-nv
```

The device program is `examples/hr_reader.nv`.  Make it the `main` of a package that
depends on this one and build it for a board:

```sh
novo build --target=nrf52840-dk src/main.nv
novo build --target=nrf54lm20-dk src/main.nv
```

## The device program

`examples/hr_reader.nv` resets the hub into its application, starts the sensor
and the MaximFast algorithm, and prints the report once a second over the
board's UART:

```
hr 63.0 bpm  confidence 99  spo2 99.6  state 3
```

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio, BoardTimer }
use dev_bus
use max32664.{ Max32664, Max32664Config, Max32664Variant, Max32664Report }

// Writes `n`, 0 or more, in decimal.  Formatting into a string would
// allocate, which @tier(embedded) refuses, so each digit goes out as a
// byte.  An Int has at most 19 decimal digits, which bounds both loops.
@tier(embedded)
fn digits(n: Int) [hw]
    var div = 1
    for _k in 0..18
        if div <= n / 10
            div = div * 10
    for _k in 0..19
        if div > 0
            hal.uart.write_byte(48 + n / div % 10)
            div = div / 10

// Writes a value in tenths as a decimal with one place: 630 is 63.0.
@tier(embedded)
fn tenths(n: Int) [hw]
    digits(n / 10)
    hal.uart.write_byte(46)
    digits(n % 10)

@tier(embedded)
fn report(r: Max32664Report) [hw]
    hal.uart.write("hr ")
    tenths(r.heart_rate_x10)
    hal.uart.write(" bpm  confidence ")
    digits(r.confidence)
    hal.uart.write("  spo2 ")
    tenths(r.spo2_x10)
    hal.uart.write("  state ")
    digits(r.state)
    hal.uart.write("\n")

@tier(embedded)
fn main() [hw]
    bsp.board.init()
    // The algorithm report only, an interrupt at one sample, the automatic
    // gain control on.
    let cfg = Max32664Config { variant: Max32664Variant.VariantA,
        output: 2,
        algo_mode: 1,
        threshold: 1,
        agc: true }
    let w = dev_bus.wiring_en(dev_bus.wiring_irq(dev_bus.wiring_i2c(0, 0x55), 29), 28)
    let hub = Max32664.attach(BoardI2c { id: 0 }, BoardTimer { id: 0 }, BoardGpio { id: 0 }, w, cfg)
    if not hub.begin()
        hal.uart.write("no answer from the hub at 0x55\n")
    while true
        match hub.fifo_count()
            Some(n) =>
                if n > 0
                    match hub.report_now()
                        Some(r) => report(r)
                        None    => hal.uart.write("no answer\n")
            None    => hal.uart.write("no answer\n")
        hal.timer.delay_ms(1000)
```

The program is `examples/hr_reader.nv`. A board package that depends on this
package builds it as its `main`. `tests/device_build.sh` does so from a scratch
package for the two boards below and for the emulated board, where it runs the
image and checks that the program reports that the hub does not answer:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 13,872 B | 1,904 B | 2,644 B |
| `nrf54lm20-dk` | 15,088 B | 2,120 B | 2,644 B |

The numbers are from the build's memory report. Flash also holds the runtime,
the compiler's support library and the program's tables. The arena, 65,536 B on
both boards, is the runtime's heap and is reserved whether or not the program
uses it; the driver allocates nothing in it.

Neither Nordic board package names a driver for its two-wire interface
(`@hal(i2c)`), so `BoardI2c` is absent hardware on both: every transfer is
refused, the program prints `no answer from the hub at 0x55`, and the build
removes the code that follows the first bus read, which makes the rows above
smaller than the image of a board with a bus.

The wiring: SDA and SCL to the board's first I2C bus (with pull-ups), the
breakout's supply to 3.3 V, RSTN to a GPIO pin of the board (the wiring's
`en`, pin 28 here) and MFIO to another (the wiring's `irq`, pin 29). Both pins
are needed: the hub chooses between its application and its bootloader by the
level of MFIO while RSTN is released, so a board that leaves them to a
jumper cannot use `begin`.

### The effects

Every function of the driver that reaches the bus, a pin or the timer
declares `[hw]`, and nothing else. The driver performs no input or output of
its own and allocates nothing of its own. The bus functions of devices-nv build
one short frame per transaction inside a `system` block and release it when the
call returns. The module checks at every tier, `embedded` included. `begin`
spends 60 ms in the pin sequence and then polls the hub every 10 ms for up to
1.7 s while the firmware initialises; each command after it waits 2 ms, and the
three that enable the sensor and the algorithm 45 ms each.

## What the hub is

The MAX32664 is a MAX32660 microcontroller (an Arm Cortex-M4 with FPU in a 1.6
by 1.6 mm, 16-bump WLP package) with a bootloader in its flash. Maxim supplies
the application as an encrypted `.msbl` file, one for each variant, and the
bootloader accepts only the file whose key its part number names (User Guide,
Table 1). The application drives the sensors on a second I2C bus, runs the
algorithm, and keeps the results in an output FIFO the host reads over the
first. The host never sees the sensor's registers except through the hub's
register commands.

### The variants

The last letter of the part number names the firmware and its key. The letter
cannot be read from the part; the host says which hub it has.

| Variant | Part number | Firmware | Source |
|---|---|---|---|
| A | MAX32664GWEA | MaximFast finger heart rate and SpO2 at 100 Hz, with a MAX30101 and optionally a KX-122 accelerometer; automatic gain control | User Guide Table 1; MAXREFDES220# |
| B | MAX32664GWEB | wrist heart rate | ADI application note "Measuring heart rate using the MAX32664B" |
| C | MAX32664GWEC | wrist heart rate and SpO2 | ADI application note "Measuring SpO2 and heart rate using the MAX32664C" |
| D | MAX32664GWED | blood pressure trending with heart rate and SpO2 | ADI application note "Measuring blood pressure, heart rate and SpO2 using the MAX32664D" |

This package implements the A variant: its commands, its report layout and
its sensor. **B, C and D are NOT IMPLEMENTED**; `config_ok` refuses them and
`begin` answers false for them without a transaction. The B, C and D rows of
the table come from the titles of the application notes, which could not be
fetched; the User Guide's Table 1 lists only the A variant, and its Table 7
gives the report layouts of the wrist heart rate, blood pressure and wrist
SpO2 algorithms, which this driver does not read. The LCSC listing of the
hub's WLP part names the D variant only, so a board built from that part needs the B, C or D
support before this driver can use it; the SparkFun breakout carries the A variant.

### The two modes and their pins

The hub starts in the bootloader or in its application by the level of MFIO
as RSTN is released (User Guide, Figures 4 and 5):

| Mode | Sequence |
|---|---|
| application | RSTN low for 10 ms; MFIO high while RSTN is low; RSTN high; 50 ms |
| bootloader | RSTN low for 10 ms; MFIO low while RSTN is low; RSTN high; 50 ms |

`reset_into(p, mfio)` runs it. `enter_app_mode` then lets MFIO go (an input with
a pull-up, because the application drives it as the interrupt) and polls the
mode read every 10 ms for up to 1.7 s, because the guide's 50 ms covers the
switch into the mode and the application initialises for about 1.5 s more
(Zephyr's comment on the C variant), during which the hub may not answer. `enter_bootloader` polls
for 200 ms and leaves MFIO a low output. In bootloader mode the driver reads
the bootloader version and page size, and the mode command (`exit_bootloader`)
leaves it. **Flashing the application through the bootloader is NOT
IMPLEMENTED**: the sequence of User Guide Table 10 (set the number of pages, the
initialisation vector, the authentication bytes, erase, send the pages) is
described there and is the work of a package of its own, with the `.msbl` files
that Maxim supplies under its terms.

### The I2C protocol

The 7-bit address is 0x55 (the guide gives the 8-bit forms 0xAA and 0xAB). A
command is a write of a family byte, an index byte and the data bytes, a stop,
a wait, and a read whose first byte is the status of the command and whose
other bytes are the answer.

| Status | Name | Meaning |
|---|---|---|
| 0x00 | success | the command was taken |
| 0x01 | ERR_UNAVAIL_CMD | an illegal family or index byte |
| 0x02 | ERR_UNAVAIL_FUNC | the function is not implemented |
| 0x03 | ERR_DATA_FORMAT | the wrong number of bytes for that family |
| 0x04 | ERR_INPUT_VALUE | an illegal value |
| 0x05 | ERR_TRY_AGAIN | the hub is busy, try again |
| 0x80 to 0x83 | ERR_BTLDR_* | a bootloader error: general, checksum, authorisation, invalid application |
| 0xFF | ERR_UNKNOWN | an unknown error |

The wait between the write and the read is the guide's `CMD_DELAY`, 60 µs. The
later firmware and the drivers read for this one wait 2 ms after an ordinary
command and 45 ms after the commands that enable a sensor or an algorithm, and
this driver does the same. `xfer` answers the status of the command as an
`Int`, -1 for a bus failure, and the bytes of the answer only for status 0; a
status of 5 sends the command again, up to three times in all, and a bus
failure is not retried.

The commands the driver uses:

| Family, index | Command | Function |
|---|---|---|
| 0x00 0x00 | read the sensor hub status (Table 6) | `status_read` |
| 0x01 0x00 | set the device mode: 0 exit the bootloader, 2 reset, 8 enter the bootloader | `mode_set` |
| 0x02 0x00 | read the device mode: 0 application, 8 bootloader | `mode_read` |
| 0x10 0x00 | set the output mode | `output_mode_set` |
| 0x10 0x01 | set the FIFO threshold | `threshold_set` |
| 0x12 0x00 | read the number of samples in the output FIFO | `samples_available` |
| 0x12 0x01 | read the output FIFO | `sample_read`, `report_read`, `drain` |
| 0x40 0x03, 0x41 0x03 | write, read a MAX30101 register | `afe_reg_write`, `afe_reg_read` |
| 0x44 0x03 | enable the MAX30101 | `afe_enable` |
| 0x52 0x00 | enable the automatic gain control | `agc_enable` |
| 0x52 0x02 | enable the WHRM or MaximFast algorithm | `algo_enable` |
| 0x81 0x00, 0x81 0x01 | bootloader version, page size | `bootloader_version`, `bootloader_page_size` |
| 0xFF 0x00 | the microcontroller type: 0 MAX32625, 1 MAX32660 or MAX32664 | `mcu_type` |
| 0xFF 0x03, 0xFF 0x07 | the sensor hub version, the algorithm version | `hub_version`, `algo_version` |

The sensor hub status byte has DataRdyInt at bit 3 (the FIFO is at its
threshold, which is also what MFIO signals), FifoOutOvrInt at bit 4 (the output
FIFO overflowed and data was lost), FifoInOvrInt at bit 5, DevBusy at bit 6 and
Err0 at bit 0 (a problem talking to the sensor); `status_data_ready` and its
neighbours read them.

### The output mode and the report

`output_mode_set` takes 1 (sensor data), 2 (algorithm data) or 3 (both), and 0
or 4 to pause the FIFO. The modes 5 to 7 add a sample counter byte to the
samples, and the guide does not say where it sits in the sample; they are NOT
IMPLEMENTED and the function refuses them. A sample is the bytes of the output
FIFO format (Table 7) in this order:

| Part | Bytes | Content |
|---|---|---|
| sensor data | 12 | four LED fields of 3 bytes, 24 bits, most significant byte first |
| algorithm report | 6 | heart rate in 0.1 bpm (2), confidence in per cent (1), SpO2 in 0.1 per cent (2), the MaximFast state (1) |
| extended report (`algo_mode` 2) | 11 | the 6 bytes, then the SpO2 r value in 0.1 (2), an extended state (1) and 2 bytes the firmware does not use |

The MaximFast state is 0 for no object, 1 for an object, 2 for an object other
than a finger and 3 for a finger. A reading is worth using when the state is 3
and the confidence is high; `report_good(r, min_conf)` says so.

The extended report is not in User Guide Rev 0. It comes from SparkFun's
library, which enables it with `algo_mode` 2 on the firmware 10.x; its layout is
taken from that library's comments and is NOT VERIFIED here. The guide's trace
(Table 9) is the only sample in the document, and the tests decode it:

```
03 6A 43   03 04 92   00 00 00   00 2E 15   02 76   63   03 E4   03
IR 223811  red 197778 LED3 0     LED4 11797  63.0    99   99.6    3
```

The guide prints the red count as 19778; the bytes 0x03 0x04 0x92 are 197778,
and the tests use that. The guide names the first LED field the IR count and the
second the red, and the driver follows it: `Max32664Sample.led1` is the first
field, and the PPG sample's `ir` is the first and `red` the second. The MAX30101
datasheet numbers its red LED first, so whether the hub's field order is the
sensor's slot order or its own is NOT VERIFIED without a part.

## What the package contains

| Module | What is in it |
|---|---|
| `max32664` | The type `Max32664` that holds a board's bus, timer and pins, the trait `Max32664Port`, the command transport, the pin sequences, the configuration, the FIFO readers, the report decoders and the `HeartRate` implementation |

## How to choose an entry point

A program on a board uses `Max32664`, built by `attach` with the board's I2C bus,
timer and GPIO, the wiring and a `Max32664Config`. The wiring's `en` is RSTN and
its `irq` is MFIO.

`Max32664Config` has five fields: `variant` (A), `output` (1 to 3), `algo_mode`
(0 none, 1 the report of Table 7, 2 the extended report; not 0 when the output
carries algorithm data), `threshold` (the FIFO level that raises DataRdyInt, 1
to 255) and `agc` (whether the algorithm's automatic gain control sets the LED
currents). `config_default()` is the guide's trace with the gain control on.

| Method | What it does |
|---|---|
| `attach(bus, timer, pins, w, cfg)` | the hub at `w.addr` (0x55); nothing is sent |
| `begin()` | resets the hub into the application and starts sampling (`bring_up`); takes up to 1.7 s; whether it came up |
| `begin_bootloader()` | resets the hub into the bootloader; whether it came up there |
| `mode()`, `status()` | the device mode, and the sensor hub status byte |
| `fifo_count()` | the samples waiting |
| `report_now()` | the next algorithm report, oldest first |
| `sample_now()` | the next whole sample |
| `hub_version()`, `algo_version()` | the firmware and algorithm versions (the guide's trace reads 1.9.0 and 1.8.2) |
| `bootloader_version()`, `bootloader_page_size()` | in bootloader mode |
| `exit_bootloader()`, `reset()` | by the mode command |
| `stop()` | the algorithm off, the sensor off, the output paused |
| `set_agc(on)` | the automatic gain control |
| `afe_reg(r)`, `set_afe_reg(r, v)` | one MAX30101 register through the hub |
| `ppg_set_rate_hz(hz)` | `HeartRate`: starts sampling and answers 100, the firmware's one rate; 0 when the hub refused |
| `ppg_set_led_ma(led, ma)` | `HeartRate`: writes the LED's pulse amplitude register, 0.2 mA a step; the current set, 0 when refused |
| `ppg_drain(out)` | `HeartRate`: up to 32 raw samples, one transaction each; -1 when the hub did not answer |
| `ppg_lost()` | `HeartRate`: 1 when the status byte reports an overflow of the output FIFO, else 0 |
| `device_id()` | `Identifiable`: the microcontroller type 1 as `who`, the firmware's major version as `rev` |
| `mfio()` | the level of MFIO |

`report_now` reads one sample and answers its report; it does not look at the
FIFO first, so a program asks `fifo_count` and reads while it is above 0. A
program that holds its own bus, or a test with a double of the hub, implements
`Max32664Port` over its bus and calls the module's functions, which take the
port: `max32664.bring_up(port, cfg)`, `max32664.report_read(port, cfg)`.

## The rules a user needs

1. The hub is slow to start. After the pin sequence the firmware initialises for
   about 1.5 s (Zephyr's comment); the commands in that time may not be taken.
   `begin` polls the mode read; a program that drives the sequence itself waits
   or polls the same way.
2. A raw sample needs the output mode to carry sensor data. With the algorithm
   output only (`output` 2) the FIFO holds reports, and `ppg_drain` answers 0
   without taking them.
3. The algorithm's automatic gain control sets the LED currents and the pulse
   width. `ppg_set_led_ma` writes the MAX30101's register through the hub and
   the algorithm overwrites it, so switch the control off (`agc: false` or
   `set_agc(false)`) for a current the program picks.
4. The A firmware samples at 100 Hz. `ppg_set_rate_hz` answers 100 whatever it is
   asked.
5. One report a second is typical at 100 Hz; the confidence needs several
   seconds with a finger in place to rise.
6. Do not enable the accelerometer if the board has none (User Guide, Table 1).
   The driver does not enable it; the commands for an external accelerometer
   (families 0x13, 0x14, 0x44 0x04) are NOT IMPLEMENTED.
7. A read of the FIFO moves the hub's read pointer by the bytes read. The driver
   reads one whole sample per transaction. A drain that fails part-way has taken
   the samples before the failure out of the hub and answers -1 with `out`
   empty.
8. The hub gives an overflow flag, not a count: `ppg_lost` answers 0 or 1.

## What is not included

NOT IMPLEMENTED:

- The B, C and D variants and their algorithms (the wrist heart rate, the wrist
  SpO2 and the blood pressure trending, with their configuration commands of
  families 0x50 and 0x51 and their report layouts).
- Flashing the application through the bootloader (User Guide Table 10 and
  Figure 13), and in-application programming.
- The MaximFast and AGC tuning commands (families 0x50 and 0x51): the SpO2
  coefficients, the AGC target, step and sensitivity, the sample counter modes.
- The external accelerometer (families 0x13, 0x14 and the enable 0x44 0x04) and
  the other sensors' enable and register commands (the MAX86141, MAX30205,
  MAX30001).
- The register dump (family 0x43) and the register attributes (0x42).
- The hub's sleep modes and the wake-up by MFIO that the Zephyr driver uses
  (to be implemented in the firmware, the guide says).
- The async form of `HeartRate`: a handle implements no trait spelled with
  `async fn` (SPEC section 14.5).

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the `HeartRate`
  and `Identifiable` traits this driver implements, the wiring record and the
  register helpers.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names the
  `I2cBus`, `GpioOut` and `Timer` traits and the board handles.
- [max30102-nv](https://novo-lang.org/packages/max30102-nv) drives the MAX30102
  directly, without a hub, and leaves the algorithm to the program. This package
  is the choice when the hub's firmware is wanted.

## Tests

`novo test` runs the module over a double of the hub: a parser of the command
frames of Table 5 with a table of answers by family and index, a status byte
first in every answer, a NACK for its first reads (as the starting hub does),
"busy" for a number of reads, an error status for a command, and a read one
byte short; and over a second double of the pins that stamps every event with
the clock. The suite checks the framing of the bus, every status of Table 4,
the retry of a busy hub and the absence of a retry on a bus failure, the delays
(2 ms and 45 ms), the sensor hub status bits, the mode read and set, the version
reads against the guide's traces (1.9.0, bootloader 3.0.0, page size 8192), the
two pin sequences against time (RSTN low 10 ms, 50 ms after), the polling of a
starting hub, a hub that answers in the other mode, every refusal of the
configuration, the commands of the guide's Table 9 in order, the decoding of the
trace sample byte by byte, the algorithm-only, sensor-only and extended layouts,
the drain with its time stamps, its cap of 32 and its failures, and the type
over the host's board handles. Each of the ten commands of `sampling_start` runs
once with its transaction failing, and must answer that it failed, so no bus error
is read as data. The suite covers every line of `src/`:

```sh
NOVO=bin/novo bash tests/coverage.sh
```

**Hardware: NOT VERIFIED.** See the device program section.

## Sources

The sources read for this driver, for their fixes and not ported, are the User
Guide above, SparkFun's `SparkFun_Bio_Sensor_Hub_Library` (the A variant's
host library, Beerware) and Zephyr's `drivers/sensor/adi/max32664c` (the C
variant's driver, Apache-2.0). Embassy, esp-hal and the Rust crates have no
driver for this hub (the crates.io search found none). The pitfalls taken from
them are named in `src/max32664.nv` at the sites that avoid them:

- SparkFun's `readByte` returns the status byte when it is not zero and the data
  byte otherwise, so an error code and a value are one kind of number; here the
  status is an `Int` of its own and the data is apart.
- SparkFun waits a second after the reset and Zephyr 1.6 s ("50 ms to switch
  into app mode plus 1500 ms of initialisation", its comment says, citing the C
  variant's guide), while the A guide gives 50 ms. Here the mode read is polled
  every 10 ms for up to 1.7 s, so the hub is used as soon as it answers.
- SparkFun's accelerometer switch tests `!= 0 || != 1`, which is always true.
  Here the parameters are checked before anything is sent.
- SparkFun reads a fixed count and trusts the bytes; here the count the bus
  returned is checked.
- SparkFun leaves MFIO an output after the bootloader sequence; here the
  application sequence releases it as the interrupt.
- Zephyr's worker reads the whole FIFO in one transaction, because, its comment
  says, the C variant does not support the read of a single message and an
  uncleared FIFO overruns. The A guide says the read pointer moves by the sample
  size read, and SparkFun reads one report at a time, so this driver reads one
  sample per transaction, which also keeps each frame within the 31 data bytes of
  a `DevBytes`. That the A firmware behaves so is NOT VERIFIED on a part; `drain`
  reads up to 32 samples, so the FIFO is emptied when it holds that many or fewer.
- The status 5 of the guide ("try again") is not retried by SparkFun; here it is,
  three times at the command wait.

## Licence

Apache-2.0.
