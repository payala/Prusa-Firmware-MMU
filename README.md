# Prusa-Firmware-MMU

## About this fork: MMU3 firmware for MMU2S boards

This is an unofficial fork of [prusa3d/Prusa-Firmware-MMU](https://github.com/prusa3d/Prusa-Firmware-MMU). It runs the MMU3 firmware on **original MMU2S control boards**, which have a known power supply design flaw. The work is on branch `mmu2s-no-uv-check`, based on upstream `v3.0.3`.

### The problem

The MMU2S board gets its 5V from the printer's Einsy board through a series diode (D1). The diode drops the MMU's supply to about 4.3V, and the rail dips further under load. Many owners bridge D1 to get closer to 5V, but the rail still sits around 4.65-4.8V and has short dips. Prusa's own fix for this on newer units is the MMU3 PD-board.

Since firmware 3.0.0, the MMU checks its own supply voltage on every main loop pass. It takes a single ADC sample of VCC (measured against the internal bandgap), with no filtering or debounce. If that one sample is below about 4.1V, the MMU panics with error **#04307 "MMU MCU undervoltage"** (`MCU_UNDERVOLTAGE_VCC`). It then disables all motors and stops the print.

On an MMU2S board, a brief dip is enough to trigger this, even when the unit would have kept working fine. The halt doesn't protect anything; it just ruins the print.

### What this fork changes

- **Undervoltage check removed.** `RuntimeHWChecks()` in `src/main.cpp` no longer checks VCC. If the supply really does get too low, the MMU simply resets, as it did before 3.0.0, instead of halting on purpose.
- **CI updated.** GitHub Actions now builds the firmware on every push to `mmu2s-*` branches, or on demand. The `.hex` is published as the `Firmware` artifact of each run. `BUILD_NUMBER` is pinned to the upstream release (896), so the printer sees the normal 3.0.3 version.

Everything else is identical to upstream.

### Who should use it

Use it if you run MMU3 firmware on an MMU2S board and keep getting #04307 with a supply you know is good enough. If you have an MMU3 or an MMU2S with the PD-board, use the official firmware instead.

### Flashing

1. Download the `.hex` from the latest successful run in the **Actions** tab, under the `Firmware` artifact.
2. Flash it with PrusaSlicer's firmware updater, the same way as an official MMU firmware.

No changes are needed on the printer side.

### Updating to a new upstream release

Rebase `mmu2s-no-uv-check` onto the new upstream tag, keep the `RuntimeHWChecks()` change, and update `-DBUILD_NUMBER` in `.github/workflows/build.yml` to the matching upstream build number.

---

This repository includes source code and firmware releases for the Original Prusa Multi Material Unit based on 8-bit ATMEL microcontroller.

The currently supported models are:
- Original Prusa MMU3
- Original Prusa MMU2S

## Introduction
This is the new firmware for the Multi Material Unit (MMU).

### Motivation
The key motivation for developing a new firmware structure were as follows:

- adding a possibility of reporting the MMU's state even during running commands - the architecture of the original MM-control-01 project didn't allow to achieve this requirement
- while being able to report the internal state of the MMU, the printer should be able to describe the error states clearly on its LCD without leaving the user to rely on some blinking LEDs
- modular design prepared for possible future upgrades

### Firmware architecture
The whole firmware is composed of simple state machines which run all at once - it is a kind of simple cooperative multi-tasking while not eating up any significant resources by deploying generic task switching solutions like RTOS or similar. The general rule is to avoid waiting inside these state machines, no state machine is allowed to block execution of others. That implies making separate waiting states which only check for some condition to be true before proceeding further.

The firmware is separated into 4 layers:

- HAL is responsible for talking to the physical hardware, in our case an AVR processor and its peripherals, TMC2130 stepper drivers, shift registers etc.
- modules are the components abstracted of the real hardware and/or connection. A typical example are the buttons, LEDs, Idler, Selector etc.
- logic layer is the application logic - this layer provides the sequences and logical relations between modules thus forming the behavior of the MMU.
- main is the top layer, it is responsible for initialization of the whole firmware and performing the main loop, where the stepping of all the automata is located.

## Getting Started

### Requirements

- Python 3.6 or newer (with pip)

### Cloning this repository

Run `git clone https://github.com/prusa3d/Prusa-Firmware-MMU.git`.

### How to prepare build env and tools
Run `./utils/bootstrap.py`

`bootstrap.py` will now download all the "missing" dependencies into the `.dependencies` folder:
- clang-format-9.0.0-noext
- cmake-3.22.5
- ninja-1.10.2
- avr-gcc-7.3.0

### How to build the preliminary project so far:
Now the process is the same as in the Buddy Firmware:
```
./utils/build.py
```

builds the `MMU3_<major>.<minor>.<revision>+<commit nr>`` in build/release folder

In case you'd like to build the project directly via cmake you can use an approach like this:
```
mkdir build
cd build
cmake .. -G Ninja -DCMAKE_TOOLCHAIN_FILE=../cmake/AvrGcc.cmake
ninja
```

It will produce a `MMU3_<version>.hex` file.

### Development

The build process of this project is driven by CMake and `build.py` is just a high-level wrapper around it. As most modern IDEs support some kind of CMake integration, it should be possible to use almost any editor for development. Below are some documents describing how to setup some popular text editors.

- Visual Studio Code
- Vim
- Other LSP-based IDEs (Atom, Sublime Text, ...)

#### Formatting

All the source code in this repository is automatically formatted:

- C/C++ files using [clang-format](https://clang.llvm.org/docs/ClangFormat.html),
- Python files using [yapf](https://github.com/google/yapf),
- and CMake files using [cmake-format](https://github.com/cheshirekow/cmake_format).

If you want to contribute, make sure to install [pre-commit](https://pre-commit.com) and then run `pre-commit install` within the repository. This makes sure that all your future commits will be formatted appropriately. Our build server automatically rejects improperly formatted pull requests.

## License

The firmware source code is licensed under the GNU General Public License v3.0 and the graphics and design are licensed under Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0). Fonts are licensed under different license (see [LICENSE](LICENSE.md)).
