# AGENTS.md

This file provides guidance to AI coding assistants (Claude Code, Copilot, Cursor, Codex, etc.) when working with code in this repository.

## Project Overview

HeadsetControl is a cross-platform C++20 application to control USB-connected headsets on Linux, macOS, and Windows. It enables controlling features like sidetone, battery status, LEDs, inactive time, equalizers, and more for various gaming headsets (Logitech, SteelSeries, Corsair, HyperX, Roccat, Audeze).

## Build System

The project uses **CMake** as its build system.

### Standard build commands:
```bash
# Initial setup
mkdir build && cd build
cmake ..
make

# Install globally (includes udev rules on Linux)
sudo make install

# On Linux after install, reload udev rules:
sudo udevadm control --reload-rules && sudo udevadm trigger
```

### Testing:
```bash
# Enable + build + run unit tests
cmake -DBUILD_UNIT_TESTS=ON ..
make check          # Builds dependencies and runs ctest --output-on-failure
# Or just:
ctest
```

### Code formatting:
**IMPORTANT:** The CI uses clang-format version 18. To avoid formatting conflicts, install the matching version:
```bash
# On macOS:
brew install llvm@18

# On Linux (Ubuntu/Debian):
sudo apt-get install clang-format-18

# Format all code
cmake -DENABLE_CLANG_FORMAT=ON ..
make format
```

### Code analysis:
```bash
cmake -DENABLE_CLANG_TIDY=ON ..
make tidy
```
The `tidy` target prefers `clang-tidy-9` if installed, otherwise falls back to the system `clang-tidy` (typically the same version as your formatter).

## Architecture

### Project Structure

```
HeadsetControl/
├── lib/                          # Core library (libheadsetcontrol)
│   ├── devices/                  # Device implementations (header-only)
│   │   ├── hid_device.hpp        # Base class for all devices
│   │   ├── protocols/            # CRTP protocol templates
│   │   │   ├── hidpp_protocol.hpp
│   │   │   ├── steelseries_protocol.hpp
│   │   │   ├── logitech_centurion_protocol.hpp
│   │   │   └── logitech_calibrations.hpp
│   │   └── *.hpp                 # Device-specific implementations
│   ├── device.hpp                # Capability enums and structs
│   ├── device_registry.hpp       # Device lookup singleton
│   ├── result_types.hpp          # Result<T> error handling
│   ├── capability_descriptors.hpp # CAPABILITY_DESCRIPTORS metadata array
│   ├── feature_handlers.hpp      # FeatureHandlerRegistry dispatch table
│   ├── headsetcontrol.hpp        # Public C++ API
│   ├── headsetcontrol_c.h        # Public C API
│   ├── dev.hpp / dev.cpp         # Dev-mode HID exploration helpers
│   └── utility.hpp               # Helper functions
├── cli/                          # Command-line interface
│   ├── main.cpp                  # Entry point
│   ├── argument_parser.hpp       # CLI argument parsing
│   ├── dev.cpp                   # Developer/debug mode (--dev)
│   └── output/                   # Serialization (JSON, YAML, ENV)
├── tests/                        # Unit and integration tests
├── docs/                         # Documentation (ADDING_A_DEVICE, ADDING_A_CAPABILITY,
│                                 #   ADDING_A_CORSAIR_DEVICE, DEVELOPMENT, LIBRARY_USAGE)
├── cmake_modules/                # Findhidapi.cmake and related
├── assets/                       # Icons and Windows resource file
├── .github/                      # CI workflows, issue and PR templates
├── CMakePresets.json             # CMake presets
└── vcpkg.json                    # vcpkg manifest (Windows builds)
```

### Device Registration System

The core architecture uses a **device registry pattern** with modern C++20:

1. **Device Registry** (`lib/device_registry.hpp`):
   - Singleton pattern for device management
   - `DeviceRegistry::instance().initialize()` registers all devices
   - `getDevice(vendor_id, product_id)` looks up devices by USB IDs
   - `getAllDevices()` returns all registered implementations

2. **HIDDevice Base Class** (`lib/devices/hid_device.hpp`):
   - Pure virtual interface for all headset devices
   - Virtual methods for each capability (e.g., `setSidetone()`, `getBattery()`)
   - Returns `Result<T>` types for proper error handling
   - Provides HID communication helpers (`writeHID()`, `readHIDTimeout()`)

3. **Device Implementations** (`lib/devices/*.hpp`):
   - Each headset is a class inheriting from `HIDDevice` (or a CRTP protocol template)
   - Header-only implementations
   - CRTP protocol templates under `lib/devices/protocols/` reduce boilerplate (HID++, SteelSeries, Logitech Centurion). Inherit via e.g. `protocols::HIDPPDevice<MyDevice>`.

### Result<T> Error Handling

All device methods return `Result<T>` (similar to `std::expected`):

```cpp
Result<BatteryResult> getBattery(hid_device* handle) override {
    auto result = readHIDTimeout(handle, buffer, timeout);
    if (!result) {
        return result.error();  // Propagate error
    }
    return BatteryResult { .level_percent = 85, .status = BATTERY_AVAILABLE };
}
```

Error types: `DeviceError::timeout()`, `hidError()`, `protocolError()`, `notSupported()`

### Adding New Device Support

See `docs/ADDING_A_DEVICE.md` for detailed instructions. Quick overview:

1. Create `lib/devices/vendor_model.hpp`
2. Inherit from `HIDDevice` or a protocol template
3. Implement required virtual methods
4. Register in `lib/device_registry.cpp`

### Using as a Library

See `docs/LIBRARY_USAGE.md` for integration guide. The library provides:

- `headsetcontrol_lib` static library target
- `headsetcontrol_shared` shared library target (with `-DBUILD_SHARED_LIBRARY=ON`)
- Public headers in `lib/` directory
- C++ API (`headsetcontrol.hpp`) and C API (`headsetcontrol_c.h`)
- Device discovery and control API

```bash
# Build shared library for FFI (Python, Rust, etc.)
cmake -DBUILD_SHARED_LIBRARY=ON ..
make
```

### Capability System

Capabilities are enumerated in `lib/device.hpp` via the `CAPABILITIES_XLIST` macro. The full set:

- `CAP_SIDETONE` — Microphone sidetone level
- `CAP_BATTERY_STATUS` — Battery level and charging status
- `CAP_NOTIFICATION_SOUND` — Play a notification tone
- `CAP_LIGHTS` — LED on/off
- `CAP_INACTIVE_TIME` — Auto-shutoff timer
- `CAP_CHATMIX_STATUS` — Game/chat audio balance
- `CAP_VOICE_PROMPTS` — Toggle spoken prompts
- `CAP_ROTATE_TO_MUTE` — Rotate boom mic to mute
- `CAP_EQUALIZER_PRESET` — Select preset EQ
- `CAP_EQUALIZER` — Custom EQ bands
- `CAP_PARAMETRIC_EQUALIZER` — Parametric EQ (gain, Q-factor, frequency)
- `CAP_MICROPHONE_MUTE_LED_BRIGHTNESS`
- `CAP_MICROPHONE_VOLUME`
- `CAP_VOLUME_LIMITER`
- `CAP_BT_WHEN_POWERED_ON` — Bluetooth-on-power-on behavior
- `CAP_BT_CALL_VOLUME`
- `CAP_NOISE_FILTER`

When adding a capability, update both `CAPABILITIES_XLIST` and the descriptor/handler tables (see Data-Driven Feature System below).

### Data-Driven Feature System

The codebase uses a data-driven approach for feature handling:

1. **Capability Descriptors** (`lib/capability_descriptors.hpp`):
   - Single source of truth for all capability metadata
   - CLI flag names, descriptions, value ranges
   - Used for validation and help text generation

2. **Feature Handler Registry** (`lib/feature_handlers.hpp`):
   - Dispatch table for feature execution
   - Eliminates giant switch statements
   - One handler per capability

```cpp
// Adding a new capability only requires:
// 1. Add descriptor to CAPABILITY_DESCRIPTORS array
// 2. Register handler in FeatureHandlerRegistry::registerAllHandlers()
```

See `docs/ADDING_A_CAPABILITY.md` for a step-by-step guide.

## Code Style

- **C++20 standard** with modern features
- `.clang-format` uses WebKit base style
- RAII for resource management
- `Result<T>` is `[[nodiscard]]` at the class level — you do **not** need to mark virtual override methods returning `Result<T>` with `[[nodiscard]]`; the enforcement is on the type
- `std::format` for string formatting
- `std::optional`, `std::span`, `std::string_view`
- Designated initializers for struct initialization
- Device files are **header-only** (`.hpp`); the only `.cpp` in `lib/devices/` is `hid_device.cpp`
- Naming: methods `camelCase`, member variables `snake_case_` (trailing underscore), constants `ALL_CAPS`
- `using namespace std::string_view_literals;` at file scope is the convention in device headers
- Use `[[maybe_unused]]` on unused parameters in virtual overrides

## Dependencies

- **HIDAPI** — USB HID communication library (required)
- **CMake** — Build system (minimum 3.12)
- **C++20 compiler** — GCC 13+, Clang 16+, Apple Clang 15+, MSVC 19.29+ (VS 2019 16.10+) as enforced in `CMakeLists.txt`

## Platform-Specific Notes

### Linux
- Generates udev rules: `headsetcontrol -u > /etc/udev/rules.d/70-headset.rules`
- Reload rules: `sudo udevadm control --reload-rules && sudo udevadm trigger`

### macOS
- No special permissions needed
- Homebrew: `brew install sapd/headsetcontrol/headsetcontrol --HEAD`

### Windows
- Uses SetupAPI for HID access
- Some devices require `usagepage`/`usageid` instead of `interface`

## Testing

- **Unit tests**: `tests/test_*.cpp` using a lightweight test framework
- **Test device**: `HeadsetControlTest` (0xF00B:0xA00C) implements all capabilities
- Run with: `./headsetcontrol --test-device -b`
- **Dev mode**: `./headsetcontrol --dev -- --list` for HID exploration
