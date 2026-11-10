# Changelog

## 0.1.0

The first release: a driver for Bosch Sensortec's BMI270 six-axis IMU on an
I2C or an SPI bus.

- `Bmi270` (I2C) and `Bmi270Spi` (SPI), the types a board's device table
  builds from the part's wiring, implementing devices-nv's `Accelerometer`,
  `Gyroscope`, `Imu`, `Thermometer` and `Identifiable`.
- The start-up of data sheet section 4.4: the SPI wake-up read, the soft
  reset, advanced power save off, the 8192-byte configuration file in
  bursts of 32 bytes on I2C and 30 on SPI with the base address set before
  each, and the poll of INTERNAL_STATUS. The file is in the module
  `bmi270_config`, unchanged from Bosch Sensortec's BMI270 Sensor API
  v2.86.1 (BSD 3-Clause; the notice is in the module and in `LICENSE`), as
  a read-only table.
- The power modes of Table 6, with a 450 µs wait after every register
  write, which advanced power save needs.
- The accelerometer in milli-g and the gyroscope in milli-degrees per
  second at the sensitivities of Tables 2 and 3, the ranges (the gyroscope
  also at 125 dps) and the rates by filter mode, the data-ready flags, the
  interrupt pin set-up and the data-ready map, and the temperature.
- The headerless FIFO, the accelerometer's and the gyroscope's
  self-tests, and the readback of the configuration file.
- The device program `examples/imu_reader.nv`, a reading loop. Built for the
  nRF52840-DK it is 31,816 B of flash and for the nRF54LM20-DK 33,440 B.
  When the package was written the two Nordic boards answered for I2C as
  absent hardware and the images were 27,560 B and 29,200 B, without the
  reading path; the TWIM drivers landed on 2026-11-10.
- Not verified on hardware: no board on the bench carries the chip. A reply
  from a BMI270 over the Nordic boards' TWIM buses is NOT VERIFIED until
  the driver rig.
- Not implemented: the auxiliary and OIS interfaces, the built-in motion
  features, the gyroscope's CRT and the offset compensation, the FIFO in
  header mode, three-wire SPI, and the async form of the traits.
- `novo pkg build --bin <name> --target=<board>` hands the board's slug to
  clang as a target triple (`bugs/pkg/build-bin-board-target.md` in the
  novo-lang repository), and a `[[bin]]` entry makes the registry list a
  library as a program. The device program is therefore built by
  `tests/device_build.sh`, as the main of a scratch package, and the
  manifest has no `[[bin]]` entry.
