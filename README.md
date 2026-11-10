# bmi270-nv

A driver for the BMI270, a six-axis inertial measurement unit (IMU) from
Bosch Sensortec, specified in the data sheet "BMI270: 6-axis, smart, low
power Inertial Measurement Unit for high-performance applications",
document BST-BMI270-DS000-08, revision 1.6, March 2026
([bosch-sensortec.com/products/motion-sensors/imus/bmi270](https://www.bosch-sensortec.com/products/motion-sensors/imus/bmi270/)).
The driver speaks I2C and SPI and implements the `Accelerometer`,
`Gyroscope`, `Imu`, `Thermometer` and `Identifiable` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv).

**Hardware verification: NOT VERIFIED.** No board on the bench carries
this chip, so the driver has run only against a double of the part, in the
host suite. The device program below builds for the nRF52840-DK and the
nRF54LM20-DK and has not been flashed.

## The device program

`examples/imu_reader.nv` reads a BMI270 on a breakout board ten times a
second and prints the acceleration in milli-g, the rate of turn in
milli-degrees per second and the temperature in thousandths of a degree
Celsius over the board's UART:

```
acc mg 12 -4 1003  gyr mdps 8 -15 3  temp mC 26125
```

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_imu.{ Imu }
use dev_bus
use bmi270.{ Bmi270 }

@tier(embedded)
fn main() [hw]
    bsp.board.init()
    // The part answers at 0x68 on the board's first I2C bus (SDO low).
    let imu = Bmi270.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 },
                            dev_bus.wiring_i2c(0, 0x68))
    // Writes the 8192-byte configuration file and enables the sensors;
    // 0 means the part took it.
    let code = imu.begin_code()
    while true
        match imu.imu_read()
            Some(s) =>
                // s.accel.z is close to 1000 with the board flat.
                hal.uart.write(str.from_int(s.accel.z))
                hal.uart.write("\n")
            None => hal.uart.write("no answer\n")
        hal.timer.delay_ms(100)
```

The full program, with the gyroscope and the temperature, is
`examples/imu_reader.nv`. A board package that depends on this package
builds it as its `main`. `tests/device_build.sh` does so from a scratch
package for the two boards below and for the emulated board, where it runs
the image and checks that the program reports that the part did not start:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which program | Of which tables and data | RAM (`.data` and `.bss`) |
|---|---|---|---|---|
| `nrf52840-dk` | 31,816 B | 8,148 B | 10,132 B | 3,748 B |
| `nrf54lm20-dk` | 33,440 B | 8,770 B | 11,078 B | 3,748 B |

The numbers are from the build's memory report. The tables hold the
8192-byte configuration file, read-only in flash. Flash also holds the
runtime and the compiler's support library. The arena, 65,536 B on both
boards, is the runtime's heap and is reserved whether or not the program
uses it. The nRF52840-DK's bus is TWIM0 on SDA P0.26 and SCL P0.27, and
the nRF54LM20-DK's is TWIM22 on SDA P1.03 and SCL P1.04
(`docs/hal/README.md` in the novo-lang repository, "The Nordic DKs'
buses"), so both images carry the reading path. Before the Nordic buses
landed on 2026-11-10 the two board packages answered for I2C as absent
hardware, and the images were 27,560 B and 29,200 B of flash with 2,644 B
of RAM. A reply from a BMI270 over those buses is NOT VERIFIED until the
driver rig. The program's effect is `[hw]`, for the I2C bus, the UART and the
timer.

`BoardI2c` reaches the bus that the board's package names with
`@hal(i2c)`. On a board that names none, the bus is absent hardware, every
transfer is refused, and the program prints `bmi270 did not start: -1`.

## What it is

The BMI270 holds a 16-bit three-axis accelerometer, a 16-bit three-axis
gyroscope and a temperature sensor in a 2.5 x 3.0 mm package. The
accelerometer measures within a **full scale** of ±2, ±4, ±8 or ±16 g
(1 g is the acceleration of gravity, 9.81 m/s²); the gyroscope within
±125, ±250, ±500, ±1000 or ±2000 degrees per second (dps). Each axis is a
signed 16-bit count, so the sensitivities are the full scale over 2^15:
16384, 8192, 4096 and 2048 LSB/g, and 262.144, 131.072, 65.536, 32.768 and
16.384 LSB/dps (data sheet Tables 2 and 3). The driver answers
acceleration in milli-g and rotation in milli-degrees per second.

The part holds no firmware of its own. After every power-up and every soft
reset the host writes it a **configuration file** of 8192 bytes, and the
sensors do not work before that. Bosch Sensortec publishes the file in its
BMI270 Sensor API; this package carries it, unchanged, in the module
`bmi270_config`, under the licence stated at the end of this page. The
driver writes it in bursts to the register INIT_DATA, setting the base
address in INIT_ADDR_0 and INIT_ADDR_1 (in words of two bytes) before each
burst, and then waits for INTERNAL_STATUS to read 1 (data sheet section
4.4).

The part has two buses. It is on I2C after power-up, at the address 0x68
(SDO to ground) or 0x69 (SDO to the supply). A rising edge on the
chip-select pin CSB switches it to SPI until the next reset. On SPI a read
returns one dummy byte before the register bytes, and the first read after
power-up or a reset must be made and its answer discarded (section 6.3 and
6.4).

## Install

```sh
novo pkg add bmi270-nv
```

## What the package contains

| Module | What is in it |
|---|---|
| `bmi270` | The register map, the conversions between a count and a physical quantity, the link to an I2C or an SPI bus, the register access over any link, and the types `Bmi270` and `Bmi270Spi`, which hold a board's bus |
| `bmi270_config` | The 8192-byte configuration file as a table in read-only memory, and its length |

## How to choose an entry point

A program on a board uses `Bmi270` for a part on I2C or `Bmi270Spi` for a
part on SPI, as the board's device table builds them:

```novo norun:fragment
    devices
        imu  Bmi270     from "bmi270-nv"  on i2c0  addr 0x68  irq imu_int1
        imu  Bmi270Spi  from "bmi270-nv"  on spi0  cs imu_cs  irq imu_int1
```

and reads it as `bsp.board.imu()`. The two types are one driver with a
different bus handle. A program that holds its own bus, or a test with a
double of the part, implements `BmiLink` over the bus (four methods and
the chunk length) and calls the module's functions, which take the link as
their first argument. The functions `i2c_read`, `i2c_write`, `spi_read`,
`spi_write` and `spi_wake` are the transports a link calls.

| Method of `Bmi270` and `Bmi270Spi` | What it does |
|---|---|
| `attach(bus, pins, w)` | the part at `w.addr`, or selected by `w.cs`; nothing is sent |
| `begin()`, `begin_code()` | bring the part up (the sequence below); the result is 0, or -1 no part, -2 a refused write, -3 the file not taken |
| `accel_read()`, `gyro_read()`, `imu_read()` | `Accelerometer`, `Gyroscope`, `Imu`: milli-g, milli-degrees per second, and both from one read |
| `accel_set_range(r)`, `gyro_set_range(r)` | the full scale |
| `accel_set_rate_hz(hz)` | the slowest rate at or above `hz` that the filter mode has |
| `accel_ready()`, `gyro_ready()` | the data-ready flags of STATUS |
| `accel_fifo_count()`, `accel_fifo_read(dst)` | the FIFO, in headerless mode |
| `temp_read()` | `Thermometer`: the temperature |
| `device_id()` | `Identifiable`: CHIP_ID, 0x24 |
| `set_power_mode(mode)` | suspend, configuration, low power, normal or performance |
| `fifo_enable(acc, gyr)` | set the FIFO to collect the accelerometer, the gyroscope or both |
| `map_data_ready(pin)` | map the data-ready interrupt to INT1 or INT2 |
| `accel_self_test()`, `gyro_self_test()` | the self-tests of section 4.12 |
| `as_link()` | the link, for the functions of the module that take one |

## The rules a user needs

1. The part does nothing before the file is written. `begin` runs the
   sequence of data sheet section 4.4: on SPI a read of CHIP_ID, answer
   discarded; CHIP_ID must read 0x24; a soft reset (CMD 0xB6) and 2 ms; on
   SPI the read again; PWR_CONF.adv_power_save to 0 and 450 µs; INIT_CTRL
   to 0; the file; INIT_CTRL to 1; INTERNAL_STATUS polled until it reads 1,
   which takes at most 20 ms. It then enables the accelerometer, the
   gyroscope and the temperature sensor and waits 45 ms, the gyroscope's
   start-up time from suspend (Table 3). The data sheet forbids the upload twice
   after one power-up or reset, so `begin` resets the part first and may
   run again at any time.
2. The file goes in bursts. A burst is at most 32 bytes on I2C and 30 on
   SPI, because the driver's buffer is 32 bytes and an SPI frame also
   holds the register byte. Before each burst the driver writes the byte
   index divided by two to INIT_ADDR_0 (the low four bits) and INIT_ADDR_1
   (the next eight). `verify_config` reads the file back and counts the
   bytes that differ.
3. With advanced power save on, which is the state after power-up and in
   the suspend and low power modes, a register write needs 450 µs before
   the next one; a write that comes sooner is lost (Table 19, section
   4.5). The driver waits 450 µs after every write it makes and writes
   one register at a time, so none is lost in any mode. The upload is the
   one exception, and it runs with advanced power save off.
4. On SPI the address byte of a read has bit 7 set, the first byte read is
   a dummy, and the part ignores writes until it has seen a read after
   power-up or a reset. `identify` and `begin` make that read. A soft
   reset puts the interface back on I2C, so `soft_reset` makes it again.
5. The registers reset to ±8 g at 100 Hz for the accelerometer and
   ±2000 dps at 200 Hz for the gyroscope. `begin` leaves them so. A
   change of range or rate takes effect at the next sample of the current
   rate (section 4.6.16).
6. The gyroscope's sensitivity in the register description is rounded
   (16.4 LSB/dps); Table 3 gives 16.384. The driver uses the table: the
   full scale over 2^15 in both sensors, so a count of 16384 at ±2000 dps
   is exactly 1000 dps.
7. The rates the filter modes allow differ (section 4.6). In the
   performance filter mode the accelerometer runs at 12.5 Hz to 1600 Hz and
   the gyroscope at 25 Hz to 3200 Hz. In the low power filter mode the
   accelerometer runs at 25/32 Hz to 400 Hz and the gyroscope at 25, 50 and
   100 Hz. `set_power_mode` holds the rates in range, and
   `accel_set_rate_hz` picks the slowest rate at or above the request
   within the mode the part is in.
8. The paired sample is one read of 12 bytes from DATA_8. The ranges are
   read in a second transaction, three bytes from ACC_RANGE, so a range
   written between the two would scale the sample wrongly.
9. The part has +-125 dps, which `GyroRange` has no value for.
   `gyr_range_write_125` sets it, and `gyro_range` then answers
   `GyroRange250dps`, the narrowest value of the trait that holds it. The
   readings are converted at 125 dps.
10. The temperature word is signed, 512 LSB per kelvin, and 0x0000 is
    23 °C. The word 0x8000 is invalid: it is what the register holds
    until the first measurement, and `temp_read` answers `None` for it.
11. A FIFO frame is 6 bytes for one sensor and 12 for both, in headerless
    mode (section 4.7.1): the gyroscope's bytes first, then the
    accelerometer's. A frame read in part is repeated whole at the next
    read, and a read past the fill level returns an uninitialised frame, so
    `accel_fifo_read` reads the fill level first and takes whole frames
    only. Headerless mode needs the sensors to run at one rate.
12. The self-tests need the board still. The accelerometer's excites the
    part with a force of each sign and compares the difference of the
    readings with 16 g on x, -15 g on y and 10 g on z. The gyroscope's
    sends the g_trigger command and reads the result from
    GYR_SELF_TEST_AXES and GYR_GAIN_STATUS. The data sheet recommends a
    reset after either, which is `soft_reset` and `begin`.

## Running on a microcontroller

Every function of the module builds at the embedded tier, and the manifest
declares that tier, so a program of the embedded, real-time or system tier
may call it. The driver's own code builds no list: its bytes move through
devices-nv's `DevBytes`, a 32-byte buffer held in the caller's frame, and
the configuration file is a read-only table indexed one byte at a time.
devices-nv's register access allocates one frame per transaction and
releases it before it returns, because `I2cBus` takes and answers lists.
The registry shows the package on the host layer, the layer of the trait
packages it depends on.

## What is not included

- The auxiliary interface (a magnetometer on the part's second bus) and
  the OIS interface. NOT IMPLEMENTED.
- The part's built-in features: the step counter, any-motion and
  no-motion, significant motion, the activity and wrist-gesture
  recognition. They are configured through the feature pages and are NOT
  IMPLEMENTED; `FEAT_PAGE` and the feature registers are reachable
  through `reg_get` and `reg_set`.
- The component retrimming of the gyroscope (CRT) and the fast offset
  compensation. NOT IMPLEMENTED.
- The FIFO in header mode, and the gyroscope's own FIFO read. The FIFO is
  headerless, and `accel_fifo_read` answers the accelerometer's frames.
- Three-wire SPI, and the NVM-backed settings (NV_CONF, the drive
  strength).
- The BMI260, which has another chip id and configuration file.
- The async form of the traits, which an `async fn` cannot take as a value
  type.

## Reference drivers read

Each of these was read for its fixes, and none was ported. The fix is
named at its site in `src/bmi270.nv`.

| Driver | What was taken from it |
|---|---|
| Bosch Sensortec BMI270 Sensor API v2.86.1 (`bmi2.c`) | the dummy read that moves the part to SPI, again after the soft reset; the file written in chunks with the address registers set before each; 450 µs after each write in advanced power save and one register at a time; the 2 ms soft reset; the 20 ms wait for INTERNAL_STATUS |
| Zephyr `drivers/sensor/bosch/bmi270` | polling INTERNAL_STATUS instead of one fixed wait; the 450 µs between PWR_CONF and INIT_CTRL; the start-up times of the two sensors |

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  traits this driver implements, the wiring record a board's device table
  builds, and the register access it uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus`, `SpiBus`, `GpioOut` and `Timer` traits and the board
  handles.
- lsm303agr-nv and l3gd20-nv drive the accelerometer and gyroscope pair of
  the STM32F3DISCOVERY; they have no file to upload.

## Tests

`novo test tests` runs the driver over a double of the part. The double
models the part at register level behind an I2C bus, an SPI bus and the
chip-select pin, so the driver's own transactions run against it: the I2C
register address, the SPI address bit and dummy byte, and the bursts. It
does what the data sheet says: CHIP_ID reads 0x24; INTERNAL_STATUS reads 1
only if INIT_CTRL was written 0, every byte of the file arrived at its
index, and INIT_CTRL was written 1; the part on SPI ignores writes before
its first read; and in advanced power save a write inside 450 µs of the
last is dropped and counted, so a test fails if the driver ever writes too
soon.

The suite checks the upload in 32-byte and 30-byte bursts against the
file, the conversions against Tables 2 and 3 and section 4.6.14, the rates
and the filter-mode ranges, every power mode, the samples at each range,
the FIFO, the self-tests, and each path of a part that stops answering.
`bash tests/coverage.sh` merges the suites' line coverage over `src/`.

## Licence

The package is under the Apache License, Version 2.0 (`LICENSE`).

The module `src/bmi270_config.nv` holds the bytes of `bmi270_config_file`
from Bosch Sensortec's BMI270 Sensor API
([github.com/boschsensortec/BMI270_SensorAPI](https://github.com/boschsensortec/BMI270_SensorAPI),
`bmi270.c` v2.86.1, Copyright (c) 2023 Bosch Sensortec GmbH), which Bosch
Sensortec publishes under the BSD 3-Clause licence. The module is
distributed under that licence, whose text is in the header of the module
and in `LICENSE`. The conditions are: the copyright notice, the list of
conditions and the disclaimer are kept in the source and reproduced in
documentation that accompanies a binary, and the name of Bosch Sensortec
and of its contributors is not used to endorse a product made from this
software without written permission. The manifest's `license` field is
`Apache-2.0 AND BSD-3-Clause`.
