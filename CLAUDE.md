# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

BlackHole is a macOS virtual audio loopback driver implementing the CoreAudio Hardware Abstraction Layer (HAL) plugin interface. The entire driver is a single C file (~4600 lines) that creates virtual audio devices allowing zero-latency audio routing between applications.

## Essential Development Commands

### Building the Driver

```bash
# Debug build (with logging enabled)
xcodebuild -project BlackHole.xcodeproj -configuration Debug -target BlackHole

# Release build
xcodebuild -project BlackHole.xcodeproj -configuration Release -target BlackHole

# Custom build with 64 channels
xcodebuild \
  -project BlackHole.xcodeproj \
  -configuration Release \
  -target BlackHole \
  PRODUCT_BUNDLE_IDENTIFIER=audio.existential.BlackHole64ch \
  GCC_PREPROCESSOR_DEFINITIONS='$GCC_PREPROCESSOR_DEFINITIONS kNumber_Of_Channels=64'
```

### Testing

```bash
# Build and run unit tests
xcodebuild -project BlackHole.xcodeproj -scheme BlackHoleTests build
./build/Release/BlackHoleTests

# Tests validate AudioServerPlugIn interface implementation
# Tests include: OwnedObjects, Streams, Controls for each device
```

### Development Installation

```bash
# 1. Build the driver (creates build/BlackHole.driver)
xcodebuild -project BlackHole.xcodeproj -configuration Debug -target BlackHole CONFIGURATION_BUILD_DIR=build

# 2. Install to system location (requires sudo)
sudo cp -R build/BlackHole.driver /Library/Audio/Plug-Ins/HAL/

# 3. Restart CoreAudio to load the driver
sudo killall -9 coreaudiod

# 4. Verify installation in Audio MIDI Setup.app or:
system_profiler SPAudioDataType
```

### Creating Distribution Installers

```bash
# Creates signed and notarized installers for 2ch, 16ch, 64ch, 128ch, 256ch
# Requires: Developer Team ID and notarytool keychain profile configured
./Installer/create_installer.sh
```

## Core Architecture

### Single-File Driver Design

BlackHole is intentionally implemented as a monolithic C file (`BlackHole/BlackHole.c`) implementing the `AudioServerPlugInDriverInterface`. This design choice simplifies distribution and compilation while maintaining all driver logic in one location.

**Key entry points:**
- `BlackHole_Create()`: Plugin factory function called by CoreAudio
- `BlackHole_Initialize()`: Sets up plugin state and host communication
- `BlackHole_CreateDevice()`: Called when devices are instantiated
- `BlackHole_StartIO()` / `BlackHole_StopIO()`: I/O lifecycle management
- `BlackHole_DoIOOperation()`: Core audio I/O processing (read/write to ring buffer)

### Ring Buffer Architecture

Audio loopback is achieved through a shared ring buffer:
- **Size**: 65,536 + latency frames (default: 65,536 frames)
- **Format**: 32-bit float interleaved channels (`Float32* gRingBuffer`)
- **Synchronization**: Protected by `gDevice_IOMutex`
- **Processing**: Uses vDSP (Accelerate framework) for efficient audio operations
  - `vDSP_vclr()`: Buffer clearing
  - `vDSP_vsmul()`: Volume scaling

**Audio Flow:**
1. Output stream writes audio to ring buffer
2. Input stream reads from ring buffer
3. Zero additional latency (writes and reads happen in same I/O cycle)

### Dual Device System

Supports two independent device objects (`kObjectID_Device` and `kObjectID_Device2`) that share the same ring buffer:
- Allows separate input-only and output-only devices
- Controlled via preprocessor definitions: `kDevice_HasInput`, `kDevice_HasOutput`, `kDevice2_HasInput`, `kDevice2_HasOutput`
- Both devices can be hidden/visible independently (`kDevice_IsHidden`, `kDevice2_IsHidden`)
- Useful for creating UX where users see one device while routing happens behind the scenes

### Thread Safety

Two mutex locks protect shared state:
- `gPlugIn_StateMutex`: Protects plugin-level state (sample rate, device state, controls)
- `gDevice_IOMutex`: Protects I/O operations and ring buffer access

All property access and I/O operations are synchronized through these mutexes.

### Object Hierarchy

```
PlugIn (kObjectID_PlugIn)
└── Box (kObjectID_Box)
    ├── Device (kObjectID_Device)
    │   ├── Stream_Output (kObjectID_Stream_Output)
    │   ├── Stream_Input (kObjectID_Stream_Input)
    │   ├── Volume_Output_Master (kObjectID_Volume_Output_Master)
    │   ├── Volume_Input_Master (kObjectID_Volume_Input_Master)
    │   ├── Mute_Output_Master (kObjectID_Mute_Output_Master)
    │   ├── Mute_Input_Master (kObjectID_Mute_Input_Master)
    │   ├── Pitch_Adjust (kObjectID_Pitch_Adjust)
    │   └── ClockSource (kObjectID_ClockSource)
    └── Device2 (kObjectID_Device2)
        └── (same sub-objects as Device)
```

## Customization via Preprocessor Definitions

All customization is done through `GCC_PREPROCESSOR_DEFINITIONS` at build time:

### Core Customization

```bash
# Driver identity (REQUIRED when renaming)
kDriver_Name=\"CustomName\"
kPlugIn_BundleID=\"com.example.CustomDriver\"
kPlugIn_Icon=\"CustomIcon.icns\"

# Channel configuration
kNumber_Of_Channels=64  # 2, 16, 64, 128, 256 typical values

# Latency (in frames)
kLatency_Frame_Size=512  # Default: 0 (zero latency)

# Sample rates (comma-separated)
kSampleRates='44100,48000'  # Default: 8000-768000 range
```

### Device Configuration

```bash
# Primary device
kDevice_Name=\"BlackHole\"
kDevice_IsHidden=false
kDevice_HasInput=true
kDevice_HasOutput=true

# Mirror device
kDevice2_Name=\"BlackHole (Mirror)\"
kDevice2_IsHidden=true
kDevice2_HasInput=true
kDevice2_HasOutput=true
```

**Example use case:** Create input-only visible device with hidden output-only mirror:
```bash
kDevice_HasInput=true kDevice_HasOutput=false kDevice_IsHidden=false
kDevice2_HasInput=false kDevice2_HasOutput=true kDevice2_IsHidden=true
```

## Development Patterns

### Debug Logging

Enabled in Debug builds via `#if DEBUG`:
```c
DebugMsg("Message: %d", value);  // Logs to syslog
```

View logs:
```bash
log stream --predicate 'process == "coreaudiod"' --level debug
```

### Error Handling Pattern

Uses goto-based error handling with macros:
```c
FailIf(condition, Done, "Error message");
FailWithAction(condition, cleanup_action, Done, "Error message");
```

### Property Access Pattern

All AudioObject properties follow the pattern:
1. `BlackHole_HasProperty()`: Check if property exists
2. `BlackHole_GetPropertyDataSize()`: Get size needed for property data
3. `BlackHole_GetPropertyData()`: Retrieve actual property data
4. `BlackHole_SetPropertyData()`: Set property data (if settable)

### Build Configuration Precedence

1. Command-line `GCC_PREPROCESSOR_DEFINITIONS` override
2. `#ifndef` checks allow compile-time defaults
3. Default values in source code

## Common Development Tasks

### Adding a New Control

1. Add object ID to `enum` (line ~110)
2. Add to device object list (`kDevice_ObjectList[]`)
3. Implement property handlers in `BlackHole_GetPropertyData()`
4. Add to control list size calculation (`device_control_list_size()`)

### Changing Default Channel Count

Edit source or use preprocessor:
```bash
# Source code (line ~236)
#ifndef kNumber_Of_Channels
#define kNumber_Of_Channels 16  # Change default

# Or via build command
GCC_PREPROCESSOR_DEFINITIONS='$GCC_PREPROCESSOR_DEFINITIONS kNumber_Of_Channels=64'
```

### Testing Device Discovery

```bash
# List all audio devices (BlackHole should appear)
system_profiler SPAudioDataType | grep -A 10 BlackHole

# Or use Audio MIDI Setup.app UI
open -a "Audio MIDI Setup"
```

## macOS Compatibility

- Minimum: macOS 10.10 (Yosemite)
- Architectures: x86_64 and arm64 (Universal Binary)
- No kernel extensions required (user-space HAL plugin)
- No system security modifications needed

## Installer/Uninstaller Architecture

### Installer Process (create_installer.sh)
1. Builds driver with channel-specific configuration
2. Generates unique UUID for each variant
3. Code signs with Developer ID
4. Creates productbuild installer package
5. Notarizes with Apple (if enabled)
6. Staples notarization ticket

### Installation Locations
- Driver bundle: `/Library/Audio/Plug-Ins/HAL/BlackHoleXch.driver`
- Ownership: `root:wheel`
- Permissions: Standard bundle permissions

### Manual Uninstallation
```bash
# Remove driver
sudo rm -R /Library/Audio/Plug-Ins/HAL/BlackHole*ch.driver

# Restart CoreAudio
sudo killall -9 coreaudiod
```
