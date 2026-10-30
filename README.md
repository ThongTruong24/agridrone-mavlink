[![Build Status](https://github.com/mavlink/mavlink/workflows/Test%20and%20deploy/badge.svg)](https://github.com/mavlink/mavlink/actions?query=branch%3Amaster)

# MAVLink

MAVLink -- Micro Air Vehicle Message Marshalling Library.

MAVLink is a very lightweight, header-only message library for communication between drones and/or ground control stations. It consists primarily of message-set specifications for different systems ("dialects") defined in XML files, and Python tools that convert these into appropriate source code for [supported languages](https://mavlink.io/en/#supported_languages). There are additional Python scripts providing examples and utilities for working with MAVLink data.

> **Tip** MAVLink is very well suited for applications with very limited communication bandwidth. Its reference implementation in C is highly optimized for resource-constrained systems with limited RAM and flash memory. It is field-proven and deployed in many products where it serves as interoperability interface between components of different manufacturers.


## Quick start

### Generate C headers

To install the minimal MAVLink environment on Ubuntu LTS 20.04, 22.04, or 24.04, enter the following on a terminal:

```bash
# Dependencies
sudo apt install python3-pip

# Clone mavlink into the directory of your choice
git clone https://github.com/mavlink/mavlink.git --recursive
cd mavlink

python3 -m pip install -r pymavlink/requirements.txt
```

You can then build the MAVLink2 C-library for `message_definitions/v1.0/common.xml` from the `/mavlink` directory as shown:

```bash
python3 -m pymavlink.tools.mavgen --lang=C --wire-protocol=2.0 --output=generated/include/mavlink/v2.0 message_definitions/v1.0/common.xml
```

### Use from cmake

To include the headers in cmake, install them locally, e.g. into the directory `install`:

```
cmake -Bbuild -H. -DCMAKE_INSTALL_PREFIX=install -DMAVLINK_DIALECT=common -DMAVLINK_VERSION=2.0
cmake --build build --target install
```

Then use `find_package` to get the dependency in `CMakeLists.txt`:

```
find_package(MAVLink REQUIRED)

add_executable(my_program my_program.c)

target_link_libraries(my_program PRIVATE MAVLink::mavlink)
```

And pass the local install directory to cmake (adapt to your directory structure):

```
cd ../my_program
cmake -Bbuild -H. -DCMAKE_PREFIX_PATH=../mavlink/install
```

For a full example, check [examples/c](examples/c).

*Note: even though we use `target_link_libraries` in cmake, it doesn't actually "link" to MAVLink as it's just a header-only library.*

### Other instructions

Instructions for using the C libraries are then covered in [Using C MAVLink Libraries (mavgen)](https://mavlink.io/en/mavgen_c/).

> **Note:** [Installing the MAVLink Toolchain](https://mavlink.io/en/getting_started/installation.html) explains how to install MAVLink on other Ubuntu platforms and Windows, while [Generating MAVLink Libraries](https://mavlink.io/en/getting_started/generate_libraries.html) explains how to build MAVLink for the other programming languages [supported by the project](https://mavlink.io/en/#supported_languages).
> The sub-topics of [Using MAVLink Libraries](https://mavlink.io/en/getting_started/use_libraries.html) explain how to use the generated libraries.


## THACO vision following control

`CC_AI_VISION_CONTROL` (42015) retains the `bounding_box`, `tracking` and
`following` flags and adds backward-compatible MAVLink2 extensions:

| Field | Wire unit | Meaning |
| --- | --- | --- |
| `distance` | 0.1 m (`dm`) | Desired following distance (centre); `30` means 3 m. |
| `threshold` | 0.1 m (`dm`) | Tolerance radius (plus/minus); `5` means 0.5 m. |

Two float extensions, `vel_gain` and `acc_gain`, multiply target velocity and acceleration
feedforward respectively (dimensionless, finite0..2, defaults0). Position and yaw remain
direct setpoints. A final `uint8_t gains_valid` distinguishes explicit zero gains (`1`)
from legacy packets (`0`, keep both current gains). Validate/apply gains as one pair;
invalid gains/presence must not prevent a valid Following OFF. These are feedforward
multipliers, not FC motion limits. Gains do not enable Following or bypass admission.
Offsets: vel_gain5, acc_gain9, gains_valid13; payload min3/max14, CRC_EXTRA211 unchanged.
Old receivers ignore new extensions. New receivers preserve gains on old packets.

The acceptable band is inclusive: `[distance - threshold, distance + threshold]`. For example,
`distance=40, threshold=3` requests following around4m with tolerance0.3m,
meaning3.7..4.3m. Position goal remains4m. Following requires both `tracking=1` and
`following=1`, together with the flight controller's Offboard admission checks.

A zero byte leaves that setting unchanged, including when an older sender omits
both extensions. The receiver must validate the resulting pair against its flight
operating limits before applying either setting: distance and width must be
positive, and width must be less than distance. Invalid settings must leave both
current settings unchanged. These bytes request runtime settings; they do not
imply persistence or acknowledge that the receiver applied them.

The original three bytes remain at offsets 0–2; distance and threshold occupy
offsets 3 and 4. Payload minimum length stays 3, maximum length becomes14 and
CRC_EXTRA stays 211. Older receivers ignore the new fields. Regenerate consumers
from this XML; generated C pack functions take distance, threshold, vel_gain, acc_gain, gains_valid
(`0, 0, 0, 0, 0` preserves configured values). Applications must explicitly handle
these fields to support changing the hover band from the GCS.

In the THACO superproject, generate Companion headers through the parent helper:

```bash
python3 sync_mavlink.py --lang C++11 \
  --output Companion_Computer/build/cc_mavlink/generated/mavlink
```

## Key Links

* [Documentation/Website](https://mavlink.io/en/) (mavlink.io/en/)
* [Discussion/Support](https://mavlink.io/en/#support)
* [Contributing](https://mavlink.io/en/contributing/contributing.html)
* [License](https://mavlink.io/en/#license)
