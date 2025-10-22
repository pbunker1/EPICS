# Running EPICS on Yocto/PetaLinux for Xilinx ZCU106

## Overview

This guide provides instructions for running EPICS (Experimental Physics and Industrial Control System) on the Xilinx ZCU106 Zynq UltraScale+ MPSoC board using Yocto/PetaLinux.

Based on proven implementations from the EPICS community, specifically:
- John Seeberger's ZCU106 implementation (2019)
- Dirk Zimoch's Zynq Ultrascale approach (2018)
- ChimeraTK meta-layer

## Hardware Platform

**Xilinx ZCU106 Evaluation Kit**
- Zynq UltraScale+ MPSoC
- ARM Cortex-A53 64-bit processors (aarch64)
- Supports PetaLinux (Yocto-based distribution)

## Approach 1: PetaLinux with Pre-Built EPICS Binaries (Recommended)

This is the approach successfully used by John Seeberger for ZCU106 and documented in the EPICS tech-talk community.

### Why This Approach?

- **Build once, deploy many**: Compile EPICS binaries once and reuse across multiple IOC instances
- **Centralized management**: Use a central EPICS infrastructure with multiple base and module versions
- **Runtime flexibility**: Select IOC at runtime without modifying filesystem
- **Service integration**: Install IOCs as systemd services for automatic startup

### Prerequisites

1. **Download from Xilinx:**
   - PetaLinux Tools (version 2018.3 or later, 2019.1, or 2022.1 tested)
   - ZCU106 BSP (Board Support Package)
     - Example: `xilinx-zcu106-v2018.3-final.bsp`

2. **EPICS Base:**
   - EPICS Base 7.0+ (includes linux-aarch64 support)
   - EPICS Base 3.15+ also works
   - Required extensions (if needed): asyn, StreamDevice, etc.

3. **Development Environment:**
   - Linux host system (Ubuntu recommended)
   - 64-bit x86_64 host for cross-compilation
   - Sufficient disk space (~50GB recommended for PetaLinux build)

### Step 1: Setup EPICS Cross-Compilation Environment

#### 1.1 Configure EPICS Environment Variables

```bash
export EPICS_BASE=/path/to/epics-base
export EPICS_HOST_ARCH=linux-x86_64  # or linux-x86
```

#### 1.2 Configure EPICS for ARM Cross-Compilation

Edit `$EPICS_BASE/configure/CONFIG_SITE`:
```bash
CROSS_COMPILER_TARGET_ARCHS=linux-arm

# Optional: Build static libraries for embedded deployment
# SHARED_LIBRARIES = NO
# STATIC_BUILD = YES
```

**Note**: For most applications, shared libraries (default) are recommended. Use static builds only if you need a fully self-contained binary.

#### 1.3 Create/Modify ARM Target Configuration

Create or edit `$EPICS_BASE/configure/os/CONFIG_SITE.linux-x86_64.linux-arm`:

```makefile
# Cross-compilation configuration for Zynq UltraScale+ (aarch64)
# Using PetaLinux/Yocto SDK

# Determines architecture (64-bit ARM)
GNU_TARGET = aarch64-linux-gnu

# Point to PetaLinux toolchain
# For PetaLinux 2018.3:
GNU_DIR = /path/to/petalinux/petalinux-v2018.3-final/tools/linux-i386/aarch64-linux-gnu

# Alternative: Point to Yocto SDK if using standalone SDK
# GNU_DIR = /opt/xilinx/petalinux/2018.3/sysroots/x86_64-petalinux-linux/usr/bin/aarch64-xilinx-linux

# Use readline library
COMMANDLINE_LIBRARY = READLINE

# Or use readline with ncurses
# COMMANDLINE_LIBRARY = READLINE_NCURSES
```

**For 32-bit ARM targets** (if using older Zynq-7000 devices), use:
```makefile
GNU_TARGET = arm-linux-gnueabi
```

Adjust paths to match your PetaLinux installation.

#### 1.4 Build EPICS Base for ARM

```bash
# Source PetaLinux environment
source /path/to/petalinux-v2018.3/settings.sh

# Build EPICS
cd $EPICS_BASE
make clean
make
```

After successful build, you'll have binaries in:
- `$EPICS_BASE/bin/linux-arm/`
- `$EPICS_BASE/lib/linux-arm/`

### Step 2: Create PetaLinux Project

#### 2.1 Create Project from BSP

```bash
petalinux-create -t project -s /path/to/xilinx-zcu106-v2018.3-final.bsp
```

This creates a project directory: `xilinx-zcu106-v2018.3`

```bash
cd xilinx-zcu106-v2018.3
```

#### 2.2 Configure Kernel (Optional)

If you need additional drivers (e.g., USB serial for IOC communication):

```bash
petalinux-configure -c kernel
```

Enable required modules:
- USB Serial support (CONFIG_USB_SERIAL)
- Prolific PL2303 USB Serial adaptor (CONFIG_USB_SERIAL_PL2303)
- Other drivers as needed

#### 2.3 Build Base System

```bash
petalinux-build
```

### Step 3: Create EPICS Recipes for PetaLinux

PetaLinux apps are Yocto recipes located in `project-spec/meta-user/recipes-apps/`

#### 3.1 Create EPICS Binaries Recipe

```bash
petalinux-create -t apps --template install --name epics --enable
```

Navigate to the recipe directory:
```bash
cd project-spec/meta-user/recipes-apps/epics/
```

**Copy EPICS binaries to `files/` directory:**
```bash
cp $EPICS_BASE/bin/linux-arm/* files/
```

Typical binaries include:
- `softIoc` - Soft IOC
- `caget`, `caput`, `camonitor`, `cainfo` - Channel Access tools
- `caRepeater` - CA Repeater daemon
- `pvget`, `pvput`, `pvlist` - PVAccess tools (if using EPICS 7+)
- Other utilities (`acctst`, `caConnTest`, `caEventRate`, etc.)

**Optional: Add systemd service for caRepeater**

Create `files/caRepeater.service`:
```ini
[Unit]
Description=EPICS Channel Access Repeater
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/caRepeater
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Edit `epics.bb` recipe:**

```bitbake
#
# EPICS binaries recipe - tested with EPICS Base 3.15.5 and 7.0.x
# Based on real-world PetaLinux implementation for ZCU106
#

SUMMARY = "EPICS base binaries for ARM"
DESCRIPTION = "EPICS (Experimental Physics and Industrial Control System) \
command-line tools and soft IOC for embedded ARM targets"
SECTION = "PETALINUX/apps"
LICENSE = "EPICS"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "file://acctst \
           file://caConnTest \
           file://caEventRate \
           file://caget \
           file://cainfo \
           file://camonitor \
           file://caput \
           file://caRepeater \
           file://caRepeater.service \
           file://casw \
           file://catime \
           file://softIoc \
          "

S = "${WORKDIR}"

# Runtime dependency on EPICS libraries
RDEPENDS_${PN} += "epicslib"

# Inherit systemd if using service
inherit systemd
SYSTEMD_SERVICE_${PN} = "caRepeater.service"
SYSTEMD_AUTO_ENABLE = "enable"

do_install() {
    # Install binaries
    install -d ${D}${bindir}
    install -m 0755 ${S}/acctst ${D}${bindir}
    install -m 0755 ${S}/caConnTest ${D}${bindir}
    install -m 0755 ${S}/caEventRate ${D}${bindir}
    install -m 0755 ${S}/caget ${D}${bindir}
    install -m 0755 ${S}/cainfo ${D}${bindir}
    install -m 0755 ${S}/camonitor ${D}${bindir}
    install -m 0755 ${S}/caput ${D}${bindir}
    install -m 0755 ${S}/caRepeater ${D}${bindir}
    install -m 0755 ${S}/casw ${D}${bindir}
    install -m 0755 ${S}/catime ${D}${bindir}
    install -m 0755 ${S}/softIoc ${D}${bindir}

    # Install systemd service
    if ${@bb.utils.contains('DISTRO_FEATURES','systemd','true','false',d)}; then
        install -d ${D}${systemd_system_unitdir}
        install -m 0644 ${S}/caRepeater.service ${D}${systemd_system_unitdir}
    fi
}

FILES_${PN} += "${systemd_system_unitdir}/caRepeater.service"
```

**Important Notes:**
- The `RDEPENDS_${PN} += "epicslib"` ensures EPICS libraries are installed before binaries
- The systemd service ensures caRepeater starts automatically on boot
- If not using systemd, remove the inherit and service-related lines

#### 3.2 Create EPICS Libraries Recipe

```bash
cd ../..
petalinux-create -t apps --template install --name epicslib --enable
cd epicslib/
```

**Copy EPICS libraries to `files/` directory:**
```bash
cp $EPICS_BASE/lib/linux-arm/*.so files/
# If using extensions (asyn, stream, etc.)
cp /path/to/asyn/lib/linux-arm/*.so files/
cp /path/to/stream/lib/linux-arm/*.so files/
```

**Important Note on Library Versioning:**

Yocto/BitBake expects versioned shared libraries (e.g., `libCom.so.3.15.5`) with proper symlinks. However, this can cause packaging issues. Two approaches:

**Approach A: Use Non-Versioned Libraries (Simpler)**
```bash
cd files/
# Copy as non-versioned
cp $EPICS_BASE/lib/linux-arm/libCom.so.3.15.5 libCom.so
cp $EPICS_BASE/lib/linux-arm/libca.so.3.15.5 libca.so
# ... repeat for all libraries
```

**Approach B: Use Versioned Libraries with Symlinks (Recommended)**
```bash
cd files/
# Copy versioned libraries
cp $EPICS_BASE/lib/linux-arm/*.so* .

# Create symlinks
ln -sf libCom.so.3.15.5 libCom.so.3
ln -sf libCom.so.3.15.5 libCom.so
ln -sf libca.so.3.15.5 libca.so.3
ln -sf libca.so.3.15.5 libca.so
# ... repeat for all libraries
```

**Edit `epicslib.bb` recipe:**

```bitbake
#
# EPICS libraries recipe - tested with EPICS Base 3.15.5 and 7.0.x
# Based on real-world PetaLinux implementation for ZCU106
#

SUMMARY = "EPICS base libraries for ARM"
DESCRIPTION = "Shared libraries for EPICS (Experimental Physics and \
Industrial Control System) including Channel Access and IOC database support"
SECTION = "PETALINUX/apps"
LICENSE = "EPICS"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

# Using non-versioned libraries to avoid Yocto packaging issues
# If you encounter errors with oe_soinstall, use this approach
SRC_URI = "file://libca.so \
           file://libcas.so \
           file://libCom.so \
           file://libdbCore.so \
           file://libdbRecStd.so \
           file://libgdd.so \
           file://libasyn.so \
           file://libstream.so \
          "

# Alternative: If using versioned libraries (commented out)
# SRC_URI = "file://libca.so.3.15.5 \
#            file://libcas.so.3.15.5 \
#            file://libCom.so.3.15.5 \
#            file://libdbCore.so.3.15.5 \
#            file://libdbRecStd.so.3.15.5 \
#            file://libgdd.so.3.15.5 \
#            file://libasyn.so \
#            file://libstream.so \
#           "

S = "${WORKDIR}"

do_install() {
    install -d ${D}${libdir}

    # Install non-versioned libraries
    install ${S}/libca.so ${D}${libdir}
    install ${S}/libcas.so ${D}${libdir}
    install ${S}/libCom.so ${D}${libdir}
    install ${S}/libdbCore.so ${D}${libdir}
    install ${S}/libdbRecStd.so ${D}${libdir}
    install ${S}/libgdd.so ${D}${libdir}

    # Install EPICS module libraries (if using asyn, stream, etc.)
    install ${S}/libasyn.so ${D}${libdir}
    install ${S}/libstream.so ${D}${libdir}

    # Alternative: If using oe_soinstall with versioned libraries
    # oe_soinstall works but may complain about non-standard versioning
    # oe_soinstall ${S}/libca.so.3.15.5 ${D}${libdir}
    # oe_soinstall ${S}/libcas.so.3.15.5 ${D}${libdir}
    # oe_soinstall ${S}/libCom.so.3.15.5 ${D}${libdir}
    # oe_soinstall ${S}/libdbCore.so.3.15.5 ${D}${libdir}
    # oe_soinstall ${S}/libdbRecStd.so.3.15.5 ${D}${libdir}
    # oe_soinstall ${S}/libgdd.so.3.15.5 ${D}${libdir}
}

# Package all .so files in main package (not just -dev)
FILES_${PN} += "${libdir}/*.so*"

# Skip QA checks that would fail for development libraries in main package
INSANE_SKIP_${PN} += "dev-so"

# Also skip ldflags check if EPICS libraries don't use standard linker flags
# INSANE_SKIP_${PN} += "ldflags"
```

**Troubleshooting Library Issues:**

If you get errors like "oe_soinstall: file libCom.so.3.15.5 in package epicslib doesn't have GNU_HASH", use the non-versioned approach with `INSANE_SKIP_${PN} += "dev-so"`.

#### 3.3 Create IOC Application Recipe (Optional)

If you have a custom IOC application:

```bash
cd ../..
petalinux-create -t apps --template install --name myioc --enable
cd myioc/
```

**Copy your IOC directory to `files/`:**
```bash
cp -r /path/to/your/ioc/iocBoot files/
```

**Edit `myioc.bb` recipe:**

```bitbake
# Custom IOC application recipe
SUMMARY = "Custom EPICS IOC application"
SECTION = "PETALINUX/apps"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

# Prevent stripping binaries
INHIBIT_PACKAGE_STRIP = "1"

SRC_URI = "file://iocBoot \
          "

homedir = "/home/root"

RDEPENDS_${PN} += "epicslib epics"

S = "${WORKDIR}"

do_install() {
    install -d ${D}/home/root
    cp -r ${WORKDIR}/iocBoot ${D}/home/root/
}

FILES_${PN} += "/home/root/*"
```

### Step 4: Build Complete System with EPICS

```bash
cd /path/to/xilinx-zcu106-v2018.3
petalinux-build
```

### Step 5: Create Bootable Image

```bash
petalinux-package --boot --format BIN \
  --fsbl images/linux/zynqmp_fsbl.elf \
  --u-boot images/linux/u-boot.elf \
  --pmufw images/linux/pmufw.elf \
  --fpga images/linux/*.bit \
  --force
```

This creates:
- `images/linux/BOOT.BIN` - Boot image
- `images/linux/image.ub` - Linux kernel and rootfs

### Step 6: Deploy to SD Card

1. Format SD card with FAT32 partition
2. Copy files to SD card:
   ```bash
   cp images/linux/BOOT.BIN /media/sdcard/
   cp images/linux/image.ub /media/sdcard/
   ```

### Step 7: Boot and Test on ZCU106

1. Insert SD card into ZCU106
2. Configure boot mode switches for SD boot
3. Power on the board
4. Connect via serial console (115200 8N1)

**Test EPICS Installation:**
```bash
# Check binaries
which softIoc
which caget

# Check libraries
ldconfig -p | grep -i com
ldconfig -p | grep -i ca

# Test CA tools
caget -h

# Test softIOC
softIoc
# At IOC prompt, type: exit
```

### Optional: Centralized EPICS Infrastructure

For managing multiple IOCs across multiple devices, consider implementing a centralized architecture:

#### Directory Structure

```
/epics/
├── base-3.15.5/          # EPICS base version 1
├── base-7.0.8/           # EPICS base version 2
├── modules/
│   └── ion/              # IOC binaries (built once)
│       ├── bin/
│       ├── lib/
│       └── dbd/
└── iocs/
    ├── xf28id1-ion1/     # IOC instance 1 (hostname-based)
    │   ├── startup.cmd
    │   ├── st.cmd
    │   ├── autosave/
    │   └── config/
    └── xf28id2-ion1/     # IOC instance 2
        ├── startup.cmd
        ├── st.cmd
        ├── autosave/
        └── config/
```

#### Benefits

1. **Build once, deploy many**: Compile IOC binaries once, create multiple instances
2. **No compilation on target**: IOC instances are just configuration files
3. **Runtime selection**: Device automatically starts its IOC based on IP/hostname
4. **Version management**: Multiple EPICS base versions coexist
5. **Service integration**: IOCs managed by systemd as services

#### Implementation

**1. Create IOC instance directory structure:**
```bash
mkdir -p /epics/iocs/${HOSTNAME}
cd /epics/iocs/${HOSTNAME}
```

**2. Create startup script with runtime selection:**
```bash
#!/bin/sh
# Determine device name from IP address
my_ip=$(ip -4 addr show eth0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
name=$(jq -c '.[]' /etc/zynq-config.json | awk "/$my_ip/ {print \$0}" | jq -r '.name')

# Start IOC only if this is the correct device
if [ "$name" != "${HOSTNAME}" ]; then
    exit 0
fi

# Set EPICS environment
export EPICS_BASE=/epics/base-7.0.8
export EPICS_HOST_ARCH=linux-arm
export PATH=${EPICS_BASE}/bin/${EPICS_HOST_ARCH}:/epics/modules/ion/bin:$PATH
export LD_LIBRARY_PATH=${EPICS_BASE}/lib/${EPICS_HOST_ARCH}:/epics/modules/ion/lib:$LD_LIBRARY_PATH

# Start the IOC
cd /epics/iocs/${HOSTNAME}
./st.cmd
```

**3. Create device configuration JSON:**
```json
{
  "devices": [
    {
      "name": "xf28id1-ion1",
      "ip": "192.168.1.100",
      "ioc": "ion",
      "base_version": "7.0.8"
    },
    {
      "name": "xf28id2-ion1",
      "ip": "192.168.1.101",
      "ioc": "ion",
      "base_version": "7.0.8"
    }
  ]
}
```

**4. Add systemd service:**
```ini
[Unit]
Description=EPICS IOC - %i
After=network.target caRepeater.service
Requires=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/epics/iocs/%i
ExecStart=/epics/iocs/%i/startup.sh
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Enable with:
```bash
systemctl enable epics-ioc@${HOSTNAME}.service
systemctl start epics-ioc@${HOSTNAME}.service
```

## Approach 2: Using meta-chimeratk Layer

ChimeraTK provides a Yocto layer with EPICS support.

### Repository
```bash
git clone https://github.com/ChimeraTK/meta-chimeratk.git
```

### Integration

Add to your `bblayers.conf`:
```
BBLAYERS += "/path/to/meta-chimeratk"
```

The layer provides:
- EPICS base recipes
- Control system adapter support
- Generic device server with EPICS integration

## Approach 3: Custom meta-epics Layer from Scratch

For full control, create a dedicated Yocto layer that builds EPICS from source.

### Step-by-Step Guide to Creating meta-epics Layer

#### Important: Check Existing Layers First

Before creating a new layer, check if someone has already created a layer containing the EPICS metadata you need:

1. **OpenEmbedded Metadata Index**: https://layers.openembedded.org/
2. **meta-chimeratk**: https://github.com/ChimeraTK/meta-chimeratk (provides EPICS support)
3. Search GitHub for "meta-epics" or similar

**Best Practices for Layer Creation:**
- Always prepend layer directory names with "meta-"
- Create layers outside the Yocto Project Source Directory (not inside poky)
- Follow the layer naming convention: `meta-root_name`
- Store custom layers in Git repositories using the `meta-layer_name` format

#### Step 1: Create the Layer Structure

Navigate to your Yocto build directory's sources folder (outside of poky):

```bash
cd /path/to/yocto-build
mkdir -p sources
cd sources
```

**Option A: Use bitbake-layers tool (Recommended)**

```bash
# Create the layer with default priority 6
bitbake-layers create-layer meta-epics

# Or specify a different priority
bitbake-layers create-layer --priority 10 meta-epics

# Or specify custom example recipe name
bitbake-layers create-layer --example-recipe-name epics-base meta-epics
```

This automatically creates:
- `conf/layer.conf` with proper configuration
- `recipes-example/example/` directory with sample recipe
- `COPYING.MIT` license file
- `README` file

**Option B: Manual creation (for learning)**

```bash
mkdir meta-epics
cd meta-epics
```

Create the directory structure:

```bash
mkdir -p conf
mkdir -p recipes-epics/epics-base
mkdir -p recipes-epics/epics-modules/epics-asyn
mkdir -p recipes-epics/epics-modules/epics-stream
mkdir -p recipes-core/images

# Optional: Create machine-specific subdirectories
mkdir -p recipes-epics/epics-base/files/zcu106
```

**Why recipes-* subdirectories?**
- Yocto expects recipes in `recipes-*` directories for organization
- Standard categories: `recipes-bsp`, `recipes-core`, `recipes-kernel`, `recipes-support`
- Custom categories like `recipes-epics` are perfectly valid

#### Step 2: Configure layer.conf

Edit `conf/layer.conf`:

```python
# We have a conf and classes directory, add to BBPATH
BBPATH .= ":${LAYERDIR}"

# We have recipes-* directories, add to BBFILES
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
            ${LAYERDIR}/recipes-*/*/*.bbappend"

BBFILE_COLLECTIONS += "meta-epics"
BBFILE_PATTERN_meta-epics = "^${LAYERDIR}/"
BBFILE_PRIORITY_meta-epics = "10"

# Specify compatible Yocto releases
LAYERDEPENDS_meta-epics = "core"
LAYERSERIES_COMPAT_meta-epics = "kirkstone langdale mickledore nanbield scarthgap"

# Layer version
LAYERVERSION_meta-epics = "1"
```

**Key Configuration Variables:**
- `BBPATH` - Adds layer to BitBake's search path
- `BBFILES` - Defines location for all recipes (*.bb files)
- `BBFILE_COLLECTIONS` - Establishes unique identifier for the layer
- `BBFILE_PRIORITY` - Establishes priority for recipes (higher = higher priority)
- `LAYERSERIES_COMPAT` - Lists compatible Yocto Project releases

#### Step 3: Create EPICS Base Recipe

Create `recipes-epics/epics-base/epics-base_7.0.8.bb`:

```bitbake
SUMMARY = "EPICS Base - Experimental Physics and Industrial Control System"
DESCRIPTION = "EPICS is a set of software tools and applications which provide \
a software infrastructure for use in building distributed control systems to \
operate devices such as particle accelerators, telescopes and other large \
scientific facilities."
HOMEPAGE = "https://epics-controls.org/"
SECTION = "devel"

LICENSE = "EPICS"
LIC_FILES_CHKSUM = "file://LICENSE;md5=61d15894e07e0d6d0f737ad05d36139a"

# Source from EPICS ANL
SRC_URI = "https://epics.anl.gov/download/base/base-${PV}.tar.gz \
           file://0001-configure-site-fixes.patch \
          "

SRC_URI[sha256sum] = "27f13e308b92eaaa99330e73a0d26b37e075c1c6e2a1eb8dc3bcb9ebe5fb5850"

S = "${WORKDIR}/base-${PV}"

# Build dependencies
DEPENDS = "readline perl-native"
RDEPENDS_${PN} = "readline"

# Don't strip binaries - EPICS uses them
INHIBIT_PACKAGE_STRIP = "1"
INHIBIT_SYSROOT_STRIP = "1"

# Export environment variables for EPICS build system
export EPICS_HOST_ARCH = "linux-${HOST_ARCH}"
export EPICS_BASE = "${S}"

# Parallel make support
PARALLEL_MAKE = "-j ${@oe.utils.cpu_count()}"

# Configure for cross-compilation
do_configure() {
    # Set cross-compilation in CONFIG_SITE
    echo "CROSS_COMPILER_TARGET_ARCHS = linux-arm" >> ${S}/configure/CONFIG_SITE
    echo "SHARED_LIBRARIES = YES" >> ${S}/configure/CONFIG_SITE
    echo "STATIC_BUILD = NO" >> ${S}/configure/CONFIG_SITE

    # Determine target architecture
    if [ "${TARGET_ARCH}" = "aarch64" ]; then
        EPICS_TARGET_ARCH="linux-arm"
        GNU_TARGET="aarch64-linux-gnu"
    elif [ "${TARGET_ARCH}" = "arm" ]; then
        EPICS_TARGET_ARCH="linux-arm"
        GNU_TARGET="arm-linux-gnueabi"
    else
        EPICS_TARGET_ARCH="linux-${TARGET_ARCH}"
        GNU_TARGET="${TARGET_SYS}"
    fi

    # Create cross-compilation config file
    cat > ${S}/configure/os/CONFIG_SITE.linux-x86_64.${EPICS_TARGET_ARCH} << EOF
# Cross-compilation configuration for ${TARGET_ARCH}
# Generated by Yocto/OpenEmbedded

# GNU cross-compilation tools
GNU_TARGET = ${GNU_TARGET}
GNU_DIR = ${STAGING_BINDIR_TOOLCHAIN}/..

# Use sysroot
SHRLIB_LDFLAGS = -shared -Wl,-soname,\$@ -Wl,-rpath-link,${STAGING_LIBDIR}

# Compiler and linker flags from Yocto
OPT_CFLAGS_YES = ${CFLAGS}
OPT_CXXFLAGS_YES = ${CXXFLAGS}
LDFLAGS += ${LDFLAGS}

# readline library
COMMANDLINE_LIBRARY = READLINE
EOF

    # Handle GNU/Linux dynamic linking
    echo 'RUNTIME_LDFLAGS_ORIGIN = -Wl,-rpath,\\$$ORIGIN/../lib' >> ${S}/configure/CONFIG_COMMON
}

do_compile() {
    # Build EPICS base
    oe_runmake CROSS_COMPILER_TARGET_ARCHS=linux-arm
}

do_install() {
    # Determine EPICS target architecture
    if [ "${TARGET_ARCH}" = "aarch64" ] || [ "${TARGET_ARCH}" = "arm" ]; then
        EPICS_TARGET="linux-arm"
    else
        EPICS_TARGET="linux-${TARGET_ARCH}"
    fi

    # Create directories
    install -d ${D}${libdir}
    install -d ${D}${bindir}
    install -d ${D}${includedir}/epics
    install -d ${D}${datadir}/epics/configure
    install -d ${D}${datadir}/epics/dbd

    # Install libraries
    if [ -d ${S}/lib/${EPICS_TARGET} ]; then
        for lib in ${S}/lib/${EPICS_TARGET}/*.so*; do
            if [ -f "$lib" ]; then
                install -m 0755 "$lib" ${D}${libdir}/
            fi
        done
    fi

    # Install binaries
    if [ -d ${S}/bin/${EPICS_TARGET} ]; then
        for bin in ${S}/bin/${EPICS_TARGET}/*; do
            if [ -f "$bin" ] && [ -x "$bin" ]; then
                install -m 0755 "$bin" ${D}${bindir}/
            fi
        done
    fi

    # Install headers
    cp -r ${S}/include/* ${D}${includedir}/epics/

    # Install DBD files (database definitions)
    cp -r ${S}/dbd ${D}${datadir}/epics/

    # Install configure files for building modules
    cp -r ${S}/configure ${D}${datadir}/epics/

    # Create environment setup script
    cat > ${D}${bindir}/epics-env.sh << 'EOF'
#!/bin/sh
# EPICS environment setup script
export EPICS_BASE=/usr/share/epics
export EPICS_HOST_ARCH=linux-arm
export PATH=${EPICS_BASE}/bin/${EPICS_HOST_ARCH}:$PATH
export LD_LIBRARY_PATH=${EPICS_BASE}/lib/${EPICS_HOST_ARCH}:$LD_LIBRARY_PATH
EOF
    chmod +x ${D}${bindir}/epics-env.sh
}

# Package files
FILES_${PN} = "${bindir}/* ${libdir}/*.so*"
FILES_${PN}-dev = "${includedir}/epics/* ${datadir}/epics/*"
FILES_${PN}-dbg = "${bindir}/.debug ${libdir}/.debug"

# Allow shipping .so files in main package
INSANE_SKIP_${PN} += "dev-so"

# EPICS uses RPATH - allow it
INSANE_SKIP_${PN} += "ldflags"

# Provides
PROVIDES = "epics-base"
```

#### Step 4: Add Patch File (if needed)

Create `recipes-epics/epics-base/files/0001-configure-site-fixes.patch` for any build system adjustments:

```patch
--- a/configure/CONFIG_SITE.Common.linux-arm
+++ b/configure/CONFIG_SITE.Common.linux-arm
@@ -1,3 +1,6 @@
 # CONFIG_SITE.Common.linux-arm

-# Settings here apply to all linux-arm target builds
+# Settings for linux-arm target builds
+
+# Use GNU tools
+GNU_TARGET = arm-linux-gnueabi
```

#### Step 5: Create Recipe for EPICS Modules (Optional)

Create `recipes-epics/epics-asyn/asyn_4.42.bb`:

```bitbake
SUMMARY = "EPICS asyn module - Asynchronous driver support"
DESCRIPTION = "Asyn is a general purpose facility for interfacing device specific \
code to low level drivers. It supports synchronous and asynchronous communication, \
and it is not restricted to use with EPICS."
HOMEPAGE = "https://epics-modules.github.io/asyn/"
LICENSE = "EPICS"
LIC_FILES_CHKSUM = "file://LICENSE;md5=..."

DEPENDS = "epics-base"
RDEPENDS_${PN} = "epics-base"

SRC_URI = "https://github.com/epics-modules/asyn/archive/R${PV}.tar.gz \
           file://RELEASE.local \
          "

S = "${WORKDIR}/asyn-R${PV}"

export EPICS_BASE = "${STAGING_DIR_TARGET}/usr/share/epics"
export EPICS_HOST_ARCH = "linux-arm"

do_configure() {
    # Configure build to find EPICS base
    cp ${WORKDIR}/RELEASE.local ${S}/configure/
}

do_compile() {
    oe_runmake
}

do_install() {
    install -d ${D}${datadir}/epics/modules/asyn
    cp -r ${S}/* ${D}${datadir}/epics/modules/asyn/

    # Install libraries
    install -d ${D}${libdir}
    install -m 0755 ${S}/lib/${EPICS_HOST_ARCH}/*.so* ${D}${libdir}/
}

FILES_${PN} = "${datadir}/epics/modules/asyn/* ${libdir}/*.so*"
INSANE_SKIP_${PN} += "dev-so"
```

Create `recipes-epics/epics-asyn/files/RELEASE.local`:

```makefile
# EPICS R7 base
EPICS_BASE = /usr/share/epics
```

#### Step 6: Add Layer to Your Build

Add the layer to your build configuration:

```bash
cd /path/to/yocto-build
bitbake-layers add-layer ../sources/meta-epics
```

Or manually edit `conf/bblayers.conf`:

```python
BBLAYERS ?= " \
  /path/to/poky/meta \
  /path/to/poky/meta-poky \
  /path/to/poky/meta-yocto-bsp \
  /path/to/meta-xilinx/meta-xilinx-core \
  /path/to/meta-xilinx/meta-xilinx-bsp \
  /path/to/sources/meta-epics \
  "
```

#### Step 7: Verify Layer Installation

Check that the layer is recognized:

```bash
bitbake-layers show-layers
```

You should see `meta-epics` in the output:

```
layer                 path                                      priority
==========================================================================
meta                  /path/to/poky/meta                        5
meta-poky             /path/to/poky/meta-poky                   5
meta-yocto-bsp        /path/to/poky/meta-yocto-bsp              5
meta-xilinx-core      /path/to/meta-xilinx/meta-xilinx-core     6
meta-xilinx-bsp       /path/to/meta-xilinx/meta-xilinx-bsp      6
meta-epics            /path/to/sources/meta-epics               10
```

**Additional Layer Management Commands:**

```bash
# Show recipes provided by all layers
bitbake-layers show-recipes

# Show only EPICS recipes
bitbake-layers show-recipes "epics*"

# Show which layers provide a specific recipe
bitbake-layers show-recipes epics-base

# Show .bbappend files and their target recipes
bitbake-layers show-appends

# Show overlayed recipes (same recipe in multiple layers)
bitbake-layers show-overlayed

# Show cross-layer dependencies
bitbake-layers show-cross-depends
```

#### Step 8: Add EPICS to Your Image

Edit your image recipe or `conf/local.conf`:

```bash
IMAGE_INSTALL_append = " epics-base"
```

Or create a custom image recipe `recipes-core/images/custom-epics-image.bb`:

```bitbake
require recipes-core/images/core-image-minimal.bb

DESCRIPTION = "Custom Linux image with EPICS support for ZCU106"

IMAGE_INSTALL += " \
    epics-base \
    epics-asyn \
    packagegroup-core-boot \
    openssh \
    "

# Increase rootfs size for EPICS
IMAGE_ROOTFS_EXTRA_SPACE = "512000"
```

#### Step 9: Build the Image

For ZCU106:

```bash
export MACHINE=zcu106-zynqmp
bitbake custom-epics-image
```

Or if using PetaLinux:

```bash
cd /path/to/petalinux-project
petalinux-config -c rootfs
# Enable user packages -> epics-base
petalinux-build
```

#### Step 10: Test the Recipe

Test building just the EPICS base recipe:

```bash
bitbake epics-base -c cleansstate
bitbake epics-base
```

Debug if needed:

```bash
# See all tasks
bitbake epics-base -c listtasks

# Check variables
bitbake -e epics-base | grep ^SRC_URI=

# Build with verbose output
bitbake epics-base -v
```

### Complete Layer Structure

Your final `meta-epics` directory structure:

```
meta-epics/
├── conf/
│   ├── layer.conf
│   └── COPYING.MIT
├── recipes-epics/
│   ├── epics-base/
│   │   ├── epics-base_7.0.8.bb
│   │   └── files/
│   │       └── 0001-configure-site-fixes.patch
│   ├── epics-asyn/
│   │   ├── asyn_4.42.bb
│   │   └── files/
│   │       └── RELEASE.local
│   └── epics-stream/
│       ├── stream_2.8.24.bb
│       └── files/
│           └── RELEASE.local
├── recipes-core/
│   └── images/
│       └── custom-epics-image.bb
└── README.md
```

### Advanced: Using recipetool

You can also use `recipetool` to auto-generate recipe templates:

```bash
# Create recipe from source tarball
recipetool create -o epics-base_7.0.8.bb \
    https://epics.anl.gov/download/base/base-7.0.8.tar.gz

# Create recipe from git repository
recipetool create -o asyn_4.42.bb \
    https://github.com/epics-modules/asyn.git
```

Then manually adjust the generated recipes for EPICS-specific build requirements.

### Best Practices for Yocto Layers

#### 1. Avoid Overlaying Entire Recipes

**Don't**: Copy an entire recipe and modify it
**Do**: Use `.bbappend` files to override only necessary parts

Example - To modify EPICS base configuration, create:
`meta-epics/recipes-epics/epics-base/epics-base_7.0.8.bbappend`

```bitbake
# Append to existing EPICS base recipe
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

# Add machine-specific configuration
SRC_URI:append:zcu106 = " file://zcu106-config.patch"

# Modify compiler flags for specific machine
CFLAGS:append:zcu106 = " -march=armv8-a"
```

#### 2. Use Machine-Specific Overrides

Place machine-specific files in subdirectories:

```
recipes-epics/epics-base/files/
├── zcu106/
│   ├── epics-config.patch
│   └── custom-dbd.template
├── zcu102/
│   └── epics-config.patch
└── common-config.sh
```

Then in your `.bbappend`:

```bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

# This file will only be used for ZCU106 builds
SRC_URI:append:zcu106 = " file://zcu106/epics-config.patch"
```

#### 3. Use :append and :prepend with Overrides

```bitbake
# Good: Machine-specific dependency
DEPENDS:append:zcu106 = " fpga-firmware"

# Good: Architecture-specific flags
KERNEL_CC:append:aarch64 = " ${TOOLCHAIN_OPTIONS}"

# Bad: Unconditional change affecting all machines
DEPENDS += " fpga-firmware"  # Don't do this in a layer!
```

#### 4. Testing Layer Compatibility

Yocto provides `yocto-check-layer` script to test layer compatibility:

```bash
# From your build directory
source oe-init-build-env
yocto-check-layer ../sources/meta-epics
```

The script runs tests including:
- **common.test_readme**: Checks for README file
- **common.test_parse**: BitBake can parse files without error
- **common.test_show_environment**: Environment is correctly configured
- **common.test_world**: `bitbake world` works
- **common.test_signatures**: Recipes don't unexpectedly change signatures
- **common.test_layerseries_compat**: Layer compatibility is set properly

To pass compatibility testing, ensure your `layer.conf` has:
```python
LAYERSERIES_COMPAT_meta-epics = "kirkstone langdale mickledore nanbield scarthgap"
```

### Using .bbappend Files with meta-epics

If you want to customize EPICS for a specific board without modifying the base recipe:

**Create a BSP-specific layer:**
```bash
bitbake-layers create-layer meta-zcu106-custom
```

**Add a .bbappend file:**
`meta-zcu106-custom/recipes-epics/epics-base/epics-base_%.bbappend`

```bitbake
# The % wildcard matches any version of epics-base

FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

# Add ZCU106-specific patches
SRC_URI:append = " \
    file://zcu106-optimizations.patch \
    file://custom-startup.sh \
"

# Modify installation for ZCU106
do_install:append() {
    install -d ${D}${sysconfdir}/epics
    install -m 0644 ${WORKDIR}/custom-startup.sh ${D}${sysconfdir}/epics/
}

FILES:${PN} += "${sysconfdir}/epics/*"
```

**Benefits of .bbappend:**
- Changes automatically apply when base recipe updates
- No need to maintain a forked copy of the entire recipe
- Clear separation between base functionality and customizations

### Saving and Restoring Layer Configuration

For reproducible builds across different machines:

```bash
# Save current layer configuration
bitbake-layers create-layers-setup /path/to/meta-epics/

# This creates:
# - setup-layers.json (layer configuration)
# - setup-layers (script to restore configuration)
```

On another machine:

```bash
# Clone the bootstrap layer
git clone <your-meta-epics-repo>

# Restore layer configuration
cd meta-epics
./setup-layers
```

### Testing on ZCU106

After building and deploying to ZCU106:

```bash
# Source EPICS environment
source /usr/bin/epics-env.sh

# Test installation
which softIoc
# Output: /usr/bin/softIoc

caget -h
# Should show caget help

# Start a test IOC
softIoc -d /usr/share/epics/dbd/softIoc.dbd
```

### Advanced: Creating EPICS-Specific Image Class

For repeated EPICS image builds, create a custom image class:

Create `meta-epics/classes/epics-image.bbclass`:

```bitbake
# Common configuration for all EPICS images

# Default EPICS packages
EPICS_CORE_PACKAGES = " \
    epics-base \
    epics-base-dev \
"

EPICS_TOOLS_PACKAGES = " \
    strace \
    procps \
    openssh \
    nfs-utils \
"

# Add to image
IMAGE_INSTALL:append = " \
    ${EPICS_CORE_PACKAGES} \
    ${EPICS_TOOLS_PACKAGES} \
"

# Increase rootfs size for EPICS
IMAGE_ROOTFS_EXTRA_SPACE = "524288"

# Set EPICS environment variables
IMAGE_FEATURES:append = " tools-debug ssh-server-openssh"

# Post-install commands
ROOTFS_POSTPROCESS_COMMAND:append = " setup_epics_environment ; "

setup_epics_environment() {
    # Create system-wide EPICS environment
    cat >> ${IMAGE_ROOTFS}/etc/profile.d/epics.sh << 'EOF'
export EPICS_BASE=/usr/share/epics
export EPICS_HOST_ARCH=linux-arm
export PATH=${EPICS_BASE}/bin/${EPICS_HOST_ARCH}:$PATH
export LD_LIBRARY_PATH=${EPICS_BASE}/lib/${EPICS_HOST_ARCH}:$LD_LIBRARY_PATH
EOF
    chmod 644 ${IMAGE_ROOTFS}/etc/profile.d/epics.sh
}
```

Then create a simple image recipe using it:

`meta-epics/recipes-core/images/epics-minimal-image.bb`:

```bitbake
require recipes-core/images/core-image-minimal.bb

DESCRIPTION = "Minimal EPICS image for ZCU106"

# Use our EPICS image class
inherit epics-image

# Add any additional packages
IMAGE_INSTALL:append = " \
    epics-asyn \
    i2c-tools \
"
```

## Configuration Files Reference

### CONFIG_SITE.linux-x86_64.linux-arm (Dirk Zimoch's approach)

```makefile
# Cross-compiler configuration for Zynq Ultrascale (aarch64)
# Using Yocto/PetaLinux SDK

GNU_HOST_ARCH = x86_64
GNU_HOST_OS = linux

# Target configuration
GNU_TARGET = aarch64-xilinx-linux

# SDK installation directory
SDK_DIR = /opt/xilinx/petalinux/2018.3

# Sysroot and toolchain paths
GNU_DIR = $(SDK_DIR)/sysroots/x86_64-petalinux-linux/usr/bin/aarch64-xilinx-linux
CROSS_COMPILER_SYSROOT = $(SDK_DIR)/sysroots/aarch64-xilinx-linux

# Use sysroot for includes and libraries
SHRLIB_LDFLAGS = -shared -Wl,-soname,$@ -Wl,-rpath-link,$(CROSS_COMPILER_SYSROOT)/lib
```

## Troubleshooting

### Library Versioning Issues

**Problem:** Yocto complains about library versioning
**Solution:** Use non-versioned `.so` files or add to recipe:
```bitbake
INSANE_SKIP_${PN} += "dev-so"
```

### Missing Dependencies

**Problem:** EPICS tools fail with "library not found"
**Solution:**
1. Check libraries are installed: `ldconfig -p | grep libCom`
2. Add library path: `export LD_LIBRARY_PATH=/usr/lib:$LD_LIBRARY_PATH`
3. Verify RDEPENDS in recipe

### Cross-Compilation Errors

**Problem:** Compilation fails with architecture mismatch
**Solution:**
1. Verify `GNU_TARGET` matches your architecture (aarch64-linux-gnu)
2. Source PetaLinux settings before building
3. Check toolchain path in `GNU_DIR`

### PetaLinux Build Failures

**Problem:** Recipe not found during build
**Solution:**
1. Ensure recipe is enabled: `petalinux-config -c rootfs`
2. Check recipe syntax: `bitbake -e epics | less`
3. Verify file paths in `SRC_URI`

## Performance Considerations

### Real-Time Performance

For real-time IOC requirements:
1. Enable PREEMPT_RT kernel patch in PetaLinux
2. Configure CPU isolation
3. Set IOC thread priorities

```bash
petalinux-config -c kernel
# Enable: Preemptible Kernel (Low-Latency Desktop)
```

### Network Optimization

For high-throughput CA/PVA traffic:
1. Increase network buffer sizes
2. Enable jumbo frames (if supported)
3. Configure EPICS environment variables:

```bash
export EPICS_CA_MAX_ARRAY_BYTES=16777216
export EPICS_CA_ADDR_LIST="your.broadcast.address"
export EPICS_CA_AUTO_ADDR_LIST=NO
```

## References

1. **EPICS Tech-Talk Thread:**
   - "using epics-base with yocto" (April-October 2019)
   - Contributors: John Seeberger, Dirk Zimoch, Mariano Ruiz

2. **Research Papers:**
   - "Design of EPICS IOC based on RAIN1000Z1 ZYNQ module"
   - "Xilinx Zynq UltraScale+ Used as Embedded IOC for BPMs"
   - "Development of Zynq SoC-Based EPICS IOC for KOMAC"

3. **Xilinx Documentation:**
   - PetaLinux Tools Documentation (UG1144)
   - Zynq UltraScale+ MPSoC Technical Reference Manual

4. **EPICS Documentation:**
   - EPICS Application Developer's Guide
   - EPICS Build System Documentation

5. **GitHub Repositories:**
   - https://github.com/ChimeraTK/meta-chimeratk
   - https://github.com/epics-base/epics-base

## Next Steps

1. Download PetaLinux tools and ZCU106 BSP from Xilinx
2. Build EPICS base for linux-arm (aarch64)
3. Create PetaLinux project and recipes
4. Test on ZCU106 hardware
5. Develop custom IOC applications as needed

## Support

- EPICS Tech-Talk mailing list: tech-talk@aps.anl.gov
- EPICS Website: https://epics-controls.org/
- Xilinx Forums: https://support.xilinx.com/
