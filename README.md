# PCILeech Firmware Generator

[![PyPI - Version](https://img.shields.io/pypi/v/:pcileechfwgenerator)](https://pypi.org/project/pcileechfwgenerator/)
[![CI](https://github.com/ramseymcgrath/PCILeechFWGenerator/workflows/CI/badge.svg)](https://github.com/ramseymcgrath/PCILeechFWGenerator/actions)
[![codecov](https://codecov.io/gh/ramseymcgrath/PCILeechFWGenerator/branch/main/graph/badge.svg)](https://codecov.io/gh/ramseymcgrath/PCILeechFWGenerator)
![](https://dcbadge.limes.pink/api/shield/429866199833247744)

Generate spoofed PCIe DMA firmware from real donor hardware with a single command. The workflow rips the donor's configuration space, builds a personalized FPGA bit‑stream locally by default (or optionally in an isolated container), and (optionally) flashes your DMA card over USB‑JTAG.

---

## 📑 Table of Contents

- [✨ Features](#-features)
- [🚀 Quick Start](#-quick-start)
  - [Installation](#installation)
  - [Usage](#usage)
- [📋 Requirements](#-requirements)
  - [Software](#software)
  - [Hardware](#hardware)
- [🛠️ Installation & Setup](#️-installation--setup)
- [🎮 Usage](#-usage-1)
  - [Interactive TUI Mode](#interactive-tui-mode-recommended)
  - [Command Line Mode](#command-line-mode)
  - [Legacy Mode](#legacy-mode-source-installation)
- [🔌 Flashing the DMA Board](#-flashing-the-dma-board)
- [🚀 Advanced Features](#-advanced-features)
  - [Full 4 KB Config-Space Shadow](#full-4-kb-config-space-shadow)
  - [MSI-X Table Replication](#msi-x-table-replication)
  - [Capability Pruning](#capability-pruning)
  - [Deterministic Variance Seeding](#deterministic-variance-seeding)
  - [Manufacturing Variance Simulation](#manufacturing-variance-simulation)
  - [Advanced SystemVerilog Generation](#advanced-systemverilog-generation)
  - [Behavioral Profiling](#behavioral-profiling)
  - [Command-Line Options](#command-line-options)
- [🧹 Cleanup & Safety](#-cleanup--safety)
- [⚠️ Disclaimer](#️-disclaimer)
- [📦 Development & Contributing](#-development--contributing)
  - [Building from Source](#building-from-source)
  - [Contributing](#contributing)
  - [Release Process](#release-process)
- [📚 Documentation](#-documentation)
- [🔧 Troubleshooting](#-troubleshooting)
- [🏆 Acknowledgments](#-acknowledgments)
- [📄 License](#-license)
- [⚠️ Legal Notice](#️-legal-notice)

---

## ✨ Features

- **🎯 Donor Hardware Analysis**: Extract real PCIe device configurations and register maps
- **💾 Full 4 KB Config-Space Shadow**: Complete configuration space emulation with overlay RAM
- **🔄 MSI-X Table Replication**: Exact replication of MSI-X tables from donor devices
- **✂️ Capability Pruning**: Selective modification of capabilities that can't be faithfully emulated
- **🎲 Deterministic Variance Seeding**: Consistent hardware variance based on device serial number
- **📊 Behavioral Profiling**: Capture dynamic device behavior patterns for enhanced realism
- **🔧 Manufacturing Variance Simulation**: Add realistic timing jitter and parameter variations
- **⚡ Advanced SystemVerilog Generation**: Comprehensive PCIe device controller with modular architecture
- **🐳 Automated Build Pipeline**: Containerized synthesis and bit-stream generation
- **🔌 USB-JTAG Flashing**: Direct firmware deployment to DMA boards
- **🖥️ Interactive TUI**: Modern text-based interface with real-time monitoring and guided workflows
- **🔁Regular Updating**: Users can update their own features and even swap between presets.

## Read this first

### About the code

The TCL is mostly templated out using Jinja in the templates dir. Python does most of the template generation and runs in a container using podman by default. It usually autodiscovers Vivado for the final TCL compilation but you can also pass in the TCL yourself. The template generation is pretty quick but depending on your PC it might take a while to compile.

I tried to align all of the python versions and paths up to be as reliable as possible, and podman helps, but you may need to edit some paths if you see issues. Please PR it back into this codebase if its something that will help the community.
This codebase is modular enough to be imported into other code bases as well.

### Supported Donor Devices

It'll best effort clone any pcie device you give it, but generally linux-compatible network/storage/media cards that work best. I don't recommend using the unit test default values outside of local testing. 
Please avoid adding those UUIDs when making tickets.

### Goals

This tool is designed to make DMA firmware transparent. A pcileech device can access all of your PCs memory and you should know whats running on it, and what it's limits are. 
This tool also provides a replacement for huge pools of firmware with identical IDs by making it easy to use your own donor card.

## 🚀 Quick Start

### Installation

```bash
# Basic installation
pip install pcileechfwgenerator

# With TUI support (recommended)
pip install pcileechfwgenerator[tui]

# Development installation
pip install pcileechfwgenerator[dev]

# Get sudo wrapper
wget https://raw.githubusercontent.com/ramseymcgrath/PCILeechFWGenerator/refs/heads/main/install-sudo-wrapper.sh

# Install sudo wrapper scripts (recommended for TUI and build commands)
./install-sudo-wrapper.sh

sudo modprobe vfio
sudo modprobe vfio-pci

```

If you have pip issues, it's usually easiest to just run from the repo. Make sure to install the python requirements.

### Other Requirements

Install Podman with your package manager. **Please just use podman!** Normal docker isn't able to mount the pcie devices correctly and you'll run into issues

### Usage

This can all be run from a venv. Ubuntu especially likes to manage the default python and some of the packages are way too old, and a venv is way easier. 

```bash

python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

## CLI
# Production mode with all advanced features enabled by default
sudo -E python3 generate.py build

## TUI 
sudo python3 tui_generate.py


```

Your board type is generally one of: `pcileech_35t325_x4` (35T) `pcileech_75t` (75T) or `pcileech_100t` (ZDMA 100T)

If you run into issue with the donor dump process, follow the manual steps.

Careful with on-board devices like audio cards. The vfio process can sometimes lock the whole southside bus up.


### Device Suitability Indicators

In the TUI, devices are evaluated for firmware generation compatibility:

| Indicator | Meaning |
|-----------|---------|
| ✅ | **Suitable**: Device is compatible and not bound to a driver |
| ⚠️ | **Suitable with Warning**: Device is compatible but bound to a driver |
| ❌ | **Not Suitable**: Device is not compatible for firmware generation |

A device is considered suitable when its suitability score is ≥ 0.7 and it has no compatibility issues.

> [!NOTE]
> Your device might still work even though it has an ❌. The TUI should usually explain why it thinks it isn't suitable but sometimes it's wrong

## 📋 Requirements

### Software

This is primarily tested in Linux, with some fiddling you could probably get it to work on Windows too.

| Tool | Why you need it | Install |
|------|----------------|---------|
| Vivado Studio | Synthesis & bit‑stream generation | Download from Xilinx (any 2022.2+ release) |
| Podman | Rootless container runtime for the build sandbox | See installation instructions below |
| Python ≥ 3.9 | Host‑side orchestrator ([`generate.py`](generate.py)) | Distro package (python3) |
| λConcept usbloader | USB flashing utility for Screamer‑class boards | See installation instructions below |
| pciutils, usbutils | lspci / lsusb helpers | Available in most Linux distributions |

> **⚠️ Security Notice**
> Never build firmware on the same operating system you plan to run the attack from. Use a separate Linux box.

### Hardware

| Component | Notes |
|-----------|-------|
| Donor PCIe card | Any inexpensive NIC, sound, or capture card works. One donor → one firmware. Destroy or quarantine the donor after extraction. |
| DMA board | Supported Artix‑7 DMA boards (35T, 75T, 100T). Must expose the Screamer USB‑JTAG port. |

> ⚠️ **CRITICAL: Pin Assignment Configuration Required**
>
> The generated constraint files contain **example pin assignments that MUST be updated** for your specific FPGA part and package before building. Using incorrect pin assignments can cause build failures or hardware damage.
>
> **Before building firmware:**
> 1. Review [`docs/PIN_ASSIGNMENT_GUIDE.md`](docs/PIN_ASSIGNMENT_GUIDE.md) for detailed instructions
> 2. Update pin assignments in the generated `.xdc` files for your board
> 3. Validate constraints using: `python scripts/validate_constraints.py`
>
> The default constraints are placeholders and will not work on real hardware without modification.

**TUI Features:**
- 🖥️ **Visual device browser** with enhanced PCIe device information
- ⚙️ **Guided configuration** with validation and profile management
- 📊 **Real-time build monitoring** with progress tracking and resource usage
- 🔍 **Intelligent error guidance** with suggested fixes
- 📡 **System status monitoring** for Podman, Vivado, USB devices, and more

See [`docs/TUI_README.md`](docs/TUI_README.md) for detailed TUI documentation.

### Command Line Mode

For automated workflows or scripting:

```bash
# Basic generation (interactive device selection)
sudo pcileech-generate

# Direct build with specific device (local build by default)
sudo pcileech-build --bdf 0000:03:00.0 --board 75t

# Build using donor_dump kernel module (opt-in)
sudo pcileech-build --bdf 0000:03:00.0 --board 75t --use-donor-dump

# Build using a previously saved donor information file
sudo pcileech-build --bdf 0000:03:00.0 --board 75t --donor-info-file /path/to/donor_info.json

# For manual donor dump generation, see the Manual Donor Dump Guide:
# docs/MANUAL_DONOR_DUMP.md

# Production build with all advanced features (default)
sudo pcileech-build --bdf 0000:03:00.0 --board 75t

# Device-specific generation with behavior profiling
sudo pcileech-build --bdf 0000:03:00.0 --board 75t \
  --device-type network --enable-behavior-profiling

# Minimal build (disable advanced features)
sudo pcileech-build --bdf 0000:03:00.0 --board 75t \
  --disable-advanced-sv --disable-variance
```

**Note:** When using container builds, the system will automatically build the required container image (`dma-fw`) if it doesn't exist. This happens during the first run and requires an internet connection to download base images.

**Output:** `output/firmware.bin` (FPGA bit‑stream ready for flashing).

### Legacy Mode (Source Installation)

```bash
# Traditional workflow (still supported)
sudo python3 generate.py
```

## 🔌 Flashing the DMA Board

> **Note:** These steps can run on the same machine or a different PC.

1. Power down, install the DMA card, and remove the donor.

2. Connect the USB‑JTAG port.

3. Flash:

```bash
usbloader -f output/firmware.bin      # auto‑detects Screamer VID:PID 1d50:6130
```

If multiple DMA boards are attached, add `--vidpid <vid:pid>`.

## 🚀 Advanced Features

### Full 4 KB Config-Space Shadow

The configuration space shadow BRAM implementation provides a complete 4 KB PCI Express configuration space in block RAM (BRAM) on the FPGA. This is a critical component for PCIe device emulation, as it allows the PCILeech firmware to accurately respond to configuration space accesses from the host system.

**Key Capabilities:**
- **Complete Configuration Space**: Full 4 KB configuration space shadow in BRAM
- **Dual-Port Access**: Simultaneous read/write operations for improved performance
- **Overlay RAM**: Dedicated storage for writable fields (Command/Status registers)
- **Donor Initialization**: Automatic initialization from donor device configuration data
- **PCIe Compatibility**: Little-endian format compatible with PCIe specification

**Integration Benefits:**
- **Enhanced Realism**: Complete configuration space emulation for better device mimicry
- **Improved Compatibility**: Support for extended capabilities and configuration registers
- **Flexible Access**: Proper handling of read-only and read-write fields
- **Seamless Integration**: Works with MSI-X table replication and capability pruning

**Usage:**
```bash
# Enabled by default in all builds
sudo pcileech-build --bdf 0000:03:00.0 --board 75t
```

For more details, see [CONFIG_SPACE_SHADOW.md](docs/CONFIG_SPACE_SHADOW.md).

### MSI-X Table Replication

The donor's MSI-X table is cloned and automatically replicated in the generated firmware.

**Key Capabilities:**
- **Automatic Parsing**: Extract MSI-X capability structure from donor configuration space
- **BAR Integration**: Seamless integration with the BAR controller for memory-mapped access
- **Enhanced Compatibility**: Support for devices that rely on MSI-X interrupts
- **Improved Performance**: Efficient interrupt handling for high-performance devices
- **Realistic Behavior**: Accurate emulation of MSI-X interrupt delivery and masking
- **Flexible Configuration**: Support for different table sizes and configurations

**Usage:**
```bash
# Automatically enabled when MSI-X capability is detected in donor device
sudo pcileech-build --bdf 0000:03:00.0 --board 75t
```

For more details, see [MSIX_TABLE_REPLICATION.md](docs/MSIX_TABLE_REPLICATION.md).

### Capability Pruning

The PCI capability pruning feature extends the PCILeech FPGA firmware generator to analyze and selectively modify or remove PCI capabilities that cannot be faithfully emulated. This ensures that the emulated device presents a consistent and compatible configuration space to the host system.

**Key Capabilities:**
- **Automatic Analysis**: Identify and categorize all capabilities in the donor's configuration space
- **Selective Modification**: Prune or modify capabilities based on emulation feasibility
- **Chain Preservation**: Maintain capability chain integrity after modifications
- **Comprehensive Coverage**: Support for both standard and extended capabilities

**Pruning Info:**
- **ASPM / L1SS**: Clear ASPM bits and remove L1 PM Substates capability
- **OBFF / LTR**: Zero OBFF & LTR bits and remove LTR capability
- **SR-IOV**: Remove SR-IOV capability entirely
- **Advanced PM**: Keep only D0/D3hot power states and clear PME support bits
- **Improved Stability**: Prevent issues with capabilities that can't be properly emulated
- **Enhanced Compatibility**: Better compatibility with different host systems
- **Reduced Detection Risk**: Remove capabilities that might reveal the emulation
- **Focused Emulation**: Concentrate on accurately emulating supported capabilities

**Usage:**
```bash
# Enabled by default in all builds
sudo pcileech-build --bdf 0000:03:00.0 --board 75t

# Disable capability pruning if needed
sudo pcileech-build --bdf 0000:03:00.0 --board 75t --disable-capability-pruning
```

For more details, see [CAPABILITY_PRUNING.md](docs/CAPABILITY_PRUNING.md).

### Variance Seeding

The deterministic variance seeding feature ensures that two builds of the same donor device at the same commit fall in the same timing band. This *means it will have*  consistent behavior across builds using the same device. But firmware will be unique 

**Key Capabilities:**
- **Deterministic Seed Generation**: Generate consistent seeds based on device serial number (DSN) and build revision
- **Consistent Variance**: Same donor device and build revision produce identical variance parameters
- **Device-Specific Variance**: Different donor devices produce different variance parameters
- **Reproducible Builds**: Consistent behavior across builds of the same donor device
- **Enhanced Realism**: Realistic hardware variance that's unique to each donor device
- **Reduced Detection Risk**: Variance parameters that match the donor device's characteristics
- **Seamless Integration**: Works with manufacturing variance simulation and behavioral profiling

**Usage:**
```bash
# Automatically enabled by default when DSN is available in donor device
sudo pcileech-build --bdf 0000:03:00.0 --board 75t
```

For more details, see [INTEGRATED_FEATURES.md](docs/INTEGRATED_FEATURES.md).

### Manufacturing Variance Simulation

The Manufacturing Variance Simulation feature adds realistic hardware variations to make generated firmware more authentic and harder to detect. This feature models real-world manufacturing tolerances and environmental conditions.

**Key Capabilities:**
- **Device Class Support**: Consumer, Enterprise, Industrial, and Automotive grade components with appropriate variance characteristics
- **Timing Variations**: Clock jitter, register access timing, and propagation delays
- **Environmental Modeling**: Temperature drift, power supply noise, and process variations
- **Integration**: Seamlessly integrates with behavior profiling for enhanced realism

**Device Classes:**
- `CONSUMER`: Standard consumer-grade tolerances (2-5% clock jitter)
- `ENTERPRISE`: Tighter enterprise-grade specifications (1-3% variations)
- `INDUSTRIAL`: Extended temperature and reliability ranges
- `AUTOMOTIVE`: Automotive-grade specifications with enhanced stability

**Usage:**
```bash
# Manufacturing variance enabled by default in production mode
sudo pcileech-generate --bdf 0000:03:00.0 --board 75t
```

### Advanced SystemVerilog Generation

The Advanced SystemVerilog Generation feature provides a comprehensive, modular PCIe device controller with enterprise-grade capabilities and optimizations.

**Architecture Components:**
- **Modular Design**: Specialized components for power, error handling, and performance monitoring
- **Multiple Clock Domains**: Proper clock domain crossing with variance-aware timing
- **Device-Specific Logic**: Optimizations for Network, Storage, Graphics, and Audio devices
- **Comprehensive Integration**: All components work together seamlessly

**Power Management Features:**
- **PCIe Power States**: Full D0, D1, D2, D3hot, D3cold state support
- **ASPM (Active State Power Management)**: L0s, L1, L1.1, L1.2 link states
- **Dynamic Power Scaling**: Frequency and voltage scaling based on workload
- **Wake-on-LAN/Event**: Configurable wake event handling

**Error Handling & Recovery:**
- **Comprehensive Error Detection**: Correctable and uncorrectable error handling
- **Advanced Error Reporting (AER)**: Full PCIe AER implementation
- **Recovery Mechanisms**: Automatic error recovery and link retraining
- **Error Logging**: Detailed error tracking and reporting

**Performance Monitoring:**
- **Hardware Counters**: Transaction, bandwidth, and latency monitoring
- **Device-Specific Metrics**: Tailored counters for different device types
- **Real-time Monitoring**: Live performance data collection
- **Threshold-based Alerts**: Configurable performance thresholds

**Device-Specific Optimizations:**
- **Network Devices**: Multi-queue support, interrupt coalescing, checksum offload
- **Storage Devices**: Command queuing, wear leveling, power loss protection
- **Graphics Devices**: Memory bandwidth optimization, display timing
- **Audio Devices**: Low-latency processing, sample rate conversion

**Usage Examples:**
```bash
# Production build with all advanced features (default)
sudo pcileech-generate --bdf 0000:03:00.0 --board 75t

# Network device with specific optimizations (default device type)
sudo pcileech-generate --bdf 0000:03:00.0 --board 75t --device-type network

# Minimal implementation (disable advanced features)
sudo pcileech-generate --bdf 0000:03:00.0 --board 75t \
  --disable-advanced-sv --disable-variance

# Storage device with performance monitoring
sudo pcileech-generate --bdf 0000:03:00.0 --board 75t \
  --device-type storage --enable-behavior-profiling
```

### Command-Line Options

**Core Options:**
- `--bdf`: PCIe Bus:Device.Function identifier (required)
- `--board`: Target board type (35t, 75t, 100t) (required)

**Donor Device Options:**
- `--use-donor-dump`: Use the donor_dump kernel module (opt-in, not default)
- `--donor-info-file`: Path to a JSON file containing donor information from a previous run

**Production Features (enabled by default):**
- `--disable-advanced-sv`: Disable advanced SystemVerilog generation
- `--device-type`: Specify device type (default: network, options: storage, graphics, audio, generic)
- `--disable-variance`: Disable manufacturing variance simulation
- `--enable-behavior-profiling`: Enable dynamic behavior profiling
- `--profile-duration`: Profiling duration in seconds (default: 30.0)

**Feature Control:**
- `--disable-power-management`: Disable power management features
- `--disable-error-handling`: Disable error handling features
- `--disable-performance-counters`: Disable performance monitoring
- `--disable-capability-pruning`: Disable capability pruning

**Analysis & Debugging:**
- `--save-analysis`: Save detailed analysis to specified file
- `--verbose`: Enable verbose output
- `--enhanced-timing`: Enable enhanced timing models (default: enabled)

## 🧹 Cleanup & Safety

- Rebind the donor back to its original driver if you keep it around.
- Keep the generated firmware private; it contains identifiers from the donor.
- Advanced features require appropriate privileges for hardware access.
- Don't try to use a donor card and fpga at the same time. I don't think Windows will really like that tbh

## Uniqueness

When you run the generator against a donor PCIe device, almost everything that an operating system or driver probes becomes an exact clone of that donor. The build script copies the full 256-byte configuration header, any extended-capability blocks, the vendor and device IDs, subsystem IDs, BAR sizes and flags, MSI/MSI-X descriptors, power-management numbers, and every vendor-defined capability byte-for-byte into a tiny ROM baked into the FPGA. It also resynthesises the BAR aperture and decode logic so the address map aligns perfectly with what the real silicon advertises. That means lspci, Windows Device Manager, or any normal driver will see a register footprint that is indistinguishable from the original card, and the FPGA bitstream’s checksum will change for every new donor because those ROM contents ripple through synthesis.

What doesn’t change from build to build is the plumbing that actually pushes data: the AXI/Avalon bridges, DMA engines, FIFOs, debug UART, JTAG CSR map, performance-counter scaffolding, and other house-keeping IP. They stay generic and parameter-driven, so you get the same timing characteristics and resource utilisation no matter which donor you point at the generator. That gives you a predictable, maintainable hardware core while still spoofing the critical identity markers upstream.

The result is “unique enough” for most red-team and research tasks—software that limits access by PCI IDs, or that sanity-checks capability chains, will be fully satisfied.

## ⚠️ Disclaimer

This tool is intended for educational research and legitimate PCIe development purposes only. Users are responsible for ensuring compliance with all applicable laws and regulations. The authors assume no liability for misuse of this software. The firmware generation is best effort and you should always validate it before use.

## 📦 Development & Contributing

For development setup instructions, please see [`docs/DEVELOPMENT.md`](docs/DEVELOPMENT.md).

### Contributing

We welcome contributions! Please see [`CONTRIBUTING.md`](CONTRIBUTING.md) for detailed guidelines.

**Quick Start:**
1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Make your changes
4. Add tests for new functionality
5. Run the test suite (`pytest`)
6. Commit your changes (`git commit -m 'feat: add amazing feature'`)
7. Push to the branch (`git push origin feature/amazing-feature`)
8. Open a Pull Request

### Release Process

For maintainers releasing new versions:

```bash
# Automated release (recommended)
python scripts/release.py minor --release-notes "Add new TUI features and improvements"

# Manual release
python -m build
twine upload dist/*
```

## 📚 Documentation

- **[Build System Architecture](docs/BUILD_SYSTEM_ARCHITECTURE.md)**: Entry points, build flow, and troubleshooting guide
- **[TUI Documentation](docs/TUI_README.md)**: Detailed TUI interface guide
- **[Manual Donor Dump Guide](docs/MANUAL_DONOR_DUMP.md)**: Step-by-step guide for manually generating donor dumps
- **[Contributing Guide](CONTRIBUTING.md)**: Development and contribution guidelines
- **[Changelog](CHANGELOG.md)**: Version history and release notes

## 🔧 Troubleshooting

### Common Issues

**Installation Problems:**
```bash
# If pip installation fails
pip install --upgrade pip setuptools wheel
pip install pcileechfwgenerator[tui]

# For development installation issues
pip install -e .[dev]
```

**TUI Not Starting:**
```bash
# Check TUI dependencies
python -c "import textual; print('TUI dependencies OK')"

# Install TUI dependencies manually
pip install textual rich psutil watchdog
```

**Permission Issues:**
```bash
# Ensure proper permissions for PCIe operations
sudo usermod -a -G vfio $USER
sudo usermod -a -G dialout $USER  # For USB-JTAG access
```

**Container Issues:**
```bash
# Check Podman installation
podman --version

# Verify rootless setup
podman info | grep rootless

# Test container build and dependencies
./scripts/build_container.sh --test

# Manual container dependency check
podman run --rm pcileechfwgenerator:latest python3 -c "import psutil, pydantic; print('Dependencies OK')"

# Check container file structure
podman run --rm pcileechfwgenerator:latest ls -la /app/src/

# Test with specific capabilities (recommended)
podman run --rm --cap-add=SYS_RAWIO --cap-add=SYS_ADMIN pcileechfwgenerator:latest echo "Capability test passed"
```

**Donor Dump Issues:**
```bash
# If donor_dump module fails to build or load
# See the Manual Donor Dump Guide for step-by-step instructions:
# docs/MANUAL_DONOR_DUMP.md

# Manual donor dump extraction (Linux)
sudo insmod src/donor_dump/donor_dump.ko bdf=0000:03:00.0
cat /proc/donor_dump > donor_info.txt

```
### Getting Help

- **GitHub Issues**: [Report bugs or request features](https://github.com/ramseymcgrath/PCILeechFWGenerator/issues)
- **GitHub Discussions**: [Community support 
- **Documentation**: Check the docs/ directory for detailed guides

## 🏆 Acknowledgments

- **Xilinx/AMD**: For Vivado synthesis tools
- **Textual**: For the modern TUI framework
- **PCILeech Community**: For feedback and contributions

## 📄 License

This project is licensed under the Apache License - see the [LICENSE](LICENSE) file for details.

## ⚠️ Legal Notice

*AGAIN* This tool is intended for educational research and legitimate PCIe development purposes only. Users are responsible for ensuring compliance with all applicable laws and regulations. The authors assume no liability for misuse of this software.

**Security Considerations:**
- Never build firmware on systems used for production or sensitive operations
- Use isolated build environments (Seperate dedicated hardware)
- Keep generated firmware private and secure
- Follow responsible disclosure practices for any security research
- Use the SECURITY.md template to raise security concerns 

---
