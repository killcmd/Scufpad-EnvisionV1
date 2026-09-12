# Scufpad

A Linux application that bridges a Scuf Envision Pro V1 controller to a virtual Xbox Elite 2 controller via the uinput
subsystem.

## Overview

The Scuf Envision Pro V1 controller (VID: `0x2e95`, PID: `0x434e`) uses highly non-standard evdev mappings that cause
incorrect button/axis assignments in most Linux games. This bridge:

1. **Reads** input from the physical Scuf controller via evdev
2. **Translates** the non-standard mappings to standard Xbox Elite 2 format
3. **Outputs** to a virtual gamepad that games recognize correctly

The virtual controller appears to games as a standard Xbox Elite 2 controller (Microsoft VID: `0x045e`, PID: `0x0b12`).

## Requirements

- Linux with uinput support
- .NET 10.0 or later
- Scuf Envision Pro V1 controller (VID: `0x2e95`, PID: `0x434e`)

## Quick Start

```bash
# 1. Set up udev rules (one-time)
sudo tee /etc/udev/rules.d/99-scuf.rules << 'EOF'
SUBSYSTEM=="input", ATTRS{idVendor}=="2e95", ATTRS{idProduct}=="434e", MODE="0666"
SUBSYSTEM=="hidraw", ATTRS{idVendor}=="2e95", ATTRS{idProduct}=="434e", MODE="0666"
KERNEL=="uinput", MODE="0666"
EOF

# 2. Reload udev rules
sudo udevadm control --reload-rules && sudo udevadm trigger

# 3. Load uinput module
sudo modprobe uinput

# 4. Run the bridge
dotnet run --project src/Scufpad
```

## Game-Specific Configuration

## Disable the original controller!
Add this export line to your terminal profile. ex. .bashrc
```
export SDL_GAMECONTROLLER_IGNORE_DEVICES=0x2e95/0x434e
```

## Setup Details

### 1. Create udev rules

Create `/etc/udev/rules.d/99-scuf.rules`:

```bash
sudo tee /etc/udev/rules.d/99-scuf.rules << 'EOF'
SUBSYSTEM=="input", ATTRS{idVendor}=="2e95", ATTRS{idProduct}=="434e", MODE="0666"
SUBSYSTEM=="hidraw", ATTRS{idVendor}=="2e95", ATTRS{idProduct}=="434e", MODE="0666"
KERNEL=="uinput", MODE="0666"
EOF
```

### 2. Reload udev rules

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Then unplug and replug your controller.

### 3. Load the uinput module

```bash
sudo modprobe uinput
```

To load uinput automatically on boot:

```bash
echo 'uinput' | sudo tee /etc/modules-load.d/uinput.conf
```

### 4. Verify setup

```bash
# Check controller is detected
lsusb | grep 2e95:434e

# Check device permissions
ls -la /dev/input/by-id/*Envision*
ls -la /dev/uinput
```

## Usage

```bash
# Run from source
dotnet run --project src/Scufpad

# Or build and run
dotnet build
./src/Scufpad/bin/Debug/net10.0/Scufpad

# AOT publish (single binary)
dotnet publish -c Release -r linux-x64
```

## Architecture

### Data Flow Pipeline

```
Physical Scuf Controller
        ↓
EvdevReader (reads evdev events)
        ↓
InputPoller (multiplexes with poll(2), 4ms timeout ≈ 250Hz)
        ↓
EnvisionMapping (translates Scuf's non-standard mappings)
        ↓
InputFilter (applies deadzone and jitter filtering)
        ↓
VirtualGamepad (emits to virtual Xbox Elite 2 via uinput)
        ↓
Games see standard Xbox controller
```

### Project Structure

```
src/Scufpad/
├── Discovery/           # Device discovery via /sys/class
│   └── DeviceDiscovery.cs
├── Input/               # Input device readers
│   ├── EvdevReader.cs   # Reads evdev input events
│   ├── HidrawReader.cs  # Reads raw HID reports (optional)
│   └── InputPoller.cs   # Multiplexes input with poll(2)
├── Interop/             # P/Invoke bindings
│   ├── Hidraw.cs        # hidraw ioctl constants
│   ├── Libc.cs          # libc functions (open, read, ioctl, poll)
│   ├── LinuxInput.cs    # input_event struct, button/axis codes
│   └── Uinput.cs        # uinput ioctl constants
├── Mapping/             # Input translation and filtering
│   ├── EnvisionMapping.cs  # Scuf V1 → Xbox mapping
│   ├── InputFilter.cs      # Deadzone and jitter filtering
│   └── InputState.cs       # Controller state representation
├── Output/              # Virtual device creation
│   └── VirtualGamepad.cs   # Creates Xbox Elite 2 via uinput
├── Services/            # Main service orchestration
│   └── BridgeService.cs    # Main loop coordination
└── Program.cs           # Entry point
```

## Hardware Mapping Reference

### Scuf Envision Pro V1 Axis Mappings

The V1 uses completely non-standard axis positions:

| Scuf Axis         | evdev Code     | Value Range     | Maps To       |
|-------------------|----------------|-----------------|---------------|
| Left Stick X      | ABS_X (0)      | -32768 to 32767 | Left Stick X  |
| Left Stick Y      | ABS_Y (1)      | -32768 to 32767 | Left Stick Y  |
| **Right Stick X** | **ABS_Z (2)**  | -32768 to 32767 | Right Stick X |
| **Left Trigger**  | **ABS_RX (3)** | 0 to 1023       | Left Trigger  |
| **Right Trigger** | **ABS_RY (4)** | 0 to 1023       | Right Trigger |
| **Right Stick Y** | **ABS_RZ (5)** | -32768 to 32767 | Right Stick Y |
| D-pad X           | ABS_HAT0X (16) | -1 to 1         | D-pad X       |
| D-pad Y           | ABS_HAT0Y (17) | -1 to 1         | D-pad Y       |

**Bold** = Non-standard mapping (differs from typical Xbox controllers)

### Scuf Envision Pro V1 Button Mappings

The V1 uses highly non-standard button codes:

| Scuf Button | evdev Code           | Standard Code | Maps To           |
|-------------|----------------------|---------------|-------------------|
| A           | BTN_SOUTH (0x130)    | BTN_SOUTH     | A Button          |
| B           | BTN_EAST (0x131)     | BTN_EAST      | B Button          |
| **X**       | **BTN_C (0x132)**    | BTN_WEST      | X Button          |
| Y           | BTN_NORTH (0x133)    | BTN_NORTH     | Y Button          |
| **LB**      | **BTN_WEST (0x134)** | BTN_TL        | Left Bumper       |
| **RB**      | **BTN_Z (0x135)**    | BTN_TR        | Right Bumper      |
| **L3**      | **BTN_TL2 (0x138)**  | BTN_THUMBL    | Left Stick Click  |
| **R3**      | **BTN_TR2 (0x139)**  | BTN_THUMBR    | Right Stick Click |
| Select      | BTN_TL (0x136)       | BTN_SELECT    | Back/Select       |
| Start       | BTN_TR (0x137)       | BTN_START     | Start             |
| Guide       | BTN_MODE (0x13c)     | BTN_MODE      | G2                |
| Paddle 1    | BTN_TRIGGER_HAPPY1   | -             | Paddle 1          |
| Paddle 2    | BTN_TRIGGER_HAPPY2   | -             | Paddle 2          |
| Paddle 3    | BTN_TRIGGER_HAPPY3   | -             | Paddle 3          |

**Bold** = Non-standard mapping

**Note:** The V1 only has 3 paddle buttons, not 4.

## Performance Tuning

The bridge is optimized for low-latency input:

| Setting                  | Value         | Purpose                                       |
|--------------------------|---------------|-----------------------------------------------|
| Poll timeout             | 4ms           | ~250Hz polling for sub-frame latency at 60fps |
| Stick deadzone           | 3500 (~10.7%) | Eliminates stick drift                        |
| Trigger deadzone         | 10 (~1%)      | Minimal deadzone for responsiveness           |
| Stick jitter threshold   | 300           | Filters analog noise                          |
| Trigger jitter threshold | 20 (~2%)      | Low threshold for responsive triggers         |

These defaults can be adjusted in `InputFilter.cs` if needed.

## Troubleshooting

### "Permission denied" errors

```bash
# Verify udev rules
cat /etc/udev/rules.d/99-scuf.rules

# Reload rules
sudo udevadm control --reload-rules && sudo udevadm trigger

# Unplug and replug controller
# Check permissions
ls -la /dev/hidraw* /dev/uinput /dev/input/event*
```

### "uinput device not found"

```bash
# Load the module
sudo modprobe uinput

# Verify it's loaded
lsmod | grep uinput
```

### Controller not detected

```bash
# Check USB connection
lsusb | grep 2e95

# View kernel messages
dmesg | tail -20

# Ensure you have the V1 controller (PID 434e)
```

## Building

```bash
# Debug build
dotnet build

# Release build
dotnet build -c Release

# Run tests
dotnet test

# AOT publish (single binary, no .NET runtime required)
dotnet publish -c Release -r linux-x64
```

## Testing

```bash
# Run all tests
dotnet test

# Run specific test project
dotnet test tests/scufpad.Tests.Unit
dotnet test tests/scufpad.Tests.Integration

# Test with evtest
evtest  # Select the virtual Xbox controller to verify output
```

## License

MIT
