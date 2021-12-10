# Libre Solar BMS Firmware

![build badge](https://github.com/LibreSolar/bms-firmware/actions/workflows/zephyr.yml/badge.svg)

This repository contains the firmware based on [Zephyr RTOS](https://www.zephyrproject.org/) for the different Libre Solar Battery Management Systems.

## Development and release model

The `main` branch contains the latest release of the firmware plus some cherry-picked bug-fixes. So you can always pull this branch to get a stable and working firmware for your BMS.

New features are developed in the `develop` branch and merged into `main` after passing tests with multiple boards. The `develop` branch is changed frequently and may even be rebased to fix previous commits. Use this branch only if you want to try out most recent features and be aware that something might be broken.

Releases are generated from `main` after significant changes have been made. The year is used as the major version number. The minor version number starts at zero and is increased for each release in that year.

## Supported boards

The software is easily configurable to support different BMS boards with STM32F072 and STM32L452 MCUs:

| Board                                      | MCU | IC | Default revision | Older revisions |
|--------------------------------------------|-----|----|------------------|-----------------|
| Libre Solar [BMS-5S50-SC](https://github.com/LibreSolar/bms-5s50-sc)   | STM32F072 | bq76920 | 0.1  |  |
| Libre Solar [BMS-15S80-SC](https://github.com/LibreSolar/bms-15s80-sc) | STM32F072 | bq76930/40 | 0.1 |     |
| Libre Solar [BMS-16S100-SC](https://github.com/LibreSolar/bms-16s100-sc) (under development) | STM32G0B1 | BQ76952 | 0.1 |  |
| Libre Solar [BMS-8S50-IC](https://github.com/LibreSolar/bms-8s50-ic) * | STM32L452 | ISL94202 | 0.2  | 0.1 |

(*) Revision 0.1 of this board is also available with STM32F072 MCU.

## Supported ICs

The firmware allows to use different BMS monitoring ICs, which usually provide single-cell voltage monitoring, balancing, current monitoring and protection features.

The chips currently supported by the firmware are listed below.

| Manufacturer       | Chip     | # Cells | Status       | Datasheet |
|--------------------|----------|---------|--------------|-----------|
| Texas Instruments  | bq76920  |   3s-5s | full support | [link](https://www.ti.com/lit/ds/symlink/bq76920.pdf)
| Texas Instruments  | bq76930  |  6s-10s | full support | [link](https://www.ti.com/lit/ds/symlink/bq76930.pdf)
| Texas Instruments  | bq76940  |  9s-15s | full support | [link](https://www.ti.com/lit/ds/symlink/bq76940.pdf)
| Texas Instruments  | BQ76952  |  3s-16s | under development | [link](https://www.ti.com/lit/ds/symlink/bq76952.pdf)
| Renesas / Intersil | ISL94202 |   3s-8s | full support | [link](https://www.renesas.com/us/en/document/dst/isl94202-datasheet)


## Building and flashing the firmware

This repository contains git submodules, so you need to clone (download) it by calling:

```
git clone --recursive https://github.com/LibreSolar/bms-firmware
```

Unfortunately, the green GitHub "Clone or download" button does not include submodules. If you cloned the repository already and want to pull the submodules, run `git submodule update --init --recursive`.

Building the firmware requires the native Zephyr build system.

This guide assumes you have already installed the Zephyr SDK and the `west` tool according to the [Zephyr documentation](https://docs.zephyrproject.org/latest/getting_started/).

The repository root directory is the `west` workspace. The `app` subfolder contains all application source files, the CMake entry point and the `west` manifest, so we have to move into this folder:

        cd app

The following command initializes the west workspace:

        west init -l .

This command will create a `.west/config` file in the workspace root and set up the repository as specified in the `west.yml` file. Afterwards the following command pulls the Zephyr source and all necessary modules, which might take a while:

        west update

Initial board selection (see `boards` subfolder for correct names):

        west build -b <board-name>@<revision>

The appended `@<revision>` specifies the board version according to the above table. It can be omitted if only a single board revision is available or if the default (most recent) version should be used. See also [here](https://docs.zephyrproject.org/latest/application/index.html#application-board-version) for more details regarding board revision handling in Zephyr.

Flash with specific debug probe (runner), e.g. J-Link:

        west flash -r jlink

User configuration using menuconfig:

        west build -t menuconfig

Report of used memory (RAM and flash):

        west build -t rom_report
        west build -t ram_report

## Firmware customization

This firmware is developed to allow easy addition of new boards and customization of the firmware.

### Hardware-specific changes

In Zephyr, all hardware-specific configuration is described in the [devicetree](https://docs.zephyrproject.org/latest/guides/dts/index.html).

The file `boards/arm/board_name/board_name.dts` contains the default devicetree specification (DTS) for a board. It is based on the DTS of the used MCU, which is included from the main Zephyr repository.

In order to overwrite the default devicetree specification, so-called overlays can be used. An overlay file can be specified via the west command line. If it is stored as `board_name.overlay` in the `app` subfolder, it will be recognized automatically when building the firmware for that board.

### Application firmware configuration

For configuration of the application-specific features, Zephyr uses the [Kconfig system](https://docs.zephyrproject.org/latest/guides/kconfig/index.html).

The configuration can be changed using `west build -t menuconfig` command or manually by changing the prj.conf file (see `Kconfig` file for possible options).

Similar to DTS overlays, Kconfig can also be customized per board. Create a folder `app/boards` and a file `board_name.conf` in that folder. The configuration from this file will be merged with the `prj.conf` automatically.

#### Change battery capacity, cell type and number of cells in series

By default, the charge controller is configured for LiFePO4 cells (`CONFIG_CELL_TYPE_LFP`). Possible other pre-defined options are `CONFIG_BAT_TYPE_NMC`, `CONFIG_BAT_TYPE_NMC_HV` and `CONFIG_BAT_TYPE_LTO`.

The number of cells is automatically selected by Kconfig to get 12V nominal voltage. It can also be manually specified via `CONFIG_NUM_CELLS_IN_SERIES`.

To compile the firmware with default settings e.g. for a 24V LiFePO4 battery with a nominal capacity of 100Ah, add the following to `prj.conf` or the board-specific `.conf` file:

```
CONFIG_BAT_CAPACITY_AH=100
CONFIG_CELL_TYPE_LFP=y
CONFIG_NUM_CELLS_IN_SERIES=8
```

#### Configure serial for ThingSet protocol

By default, the BMS uses the serial interface in the UEXT connector for the [ThingSet protocol](https://libre.solar/thingset/). This allows to use WiFi modules with ESP32 without any firmware change.

Add the following configuration if you prefer to use the serial of the additional debug RX/TX pins present on many boards:

```
CONFIG_UEXT_SERIAL_THINGSET=n
```

To disable regular data publication in one-second interval on ThingSet serial at startup add the following configuration:

```
CONFIG_THINGSET_SERIAL_PUB_DEFAULT=n
```

## API documentation

The documentation auto-generated by Doxygen can be found [here](https://libre.solar/bms-firmware/).

## Unit tests

Writing the tests is still work in progress. New functions should be implemented in test-driven development fashion. Tests for old functions will be added step by step.

Build and run the tests with the following command:

    cd tests
    west build -p -b native_posix -t run

By default, the unit tests are run for the ISL94202 IC, but you can select other ICs by specifying an overlay located in `tests/boards`.

    west build -p -b native_posix -t run -- -DDTC_OVERLAY_FILE="boards/bq769x0.overlay"
