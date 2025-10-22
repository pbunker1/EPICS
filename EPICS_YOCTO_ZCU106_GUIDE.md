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

This is the approach successfully used by John Seeberger for ZCU106.

### Prerequisites

1. **Download from Xilinx:**
   - PetaLinux Tools (version 2018.3 or later)
   - ZCU106 BSP (Board Support Package)
     - Example: `xilinx-zcu106-v2018.3-final.bsp`

2. **EPICS Base:**
   - EPICS Base 7.0+ (includes linux-aarch64 support)
   - Required extensions (if needed): asyn, StreamDevice, etc.

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
```

#### 1.3 Create/Modify ARM Target Configuration

Edit `$EPICS_BASE/configure/os/CONFIG_SITE.linux-x86_64.linux-arm`:
```makefile
# Determines architecture (64-bit ARM)
GNU_TARGET = aarch64-linux-gnu

# Point to PetaLinux toolchain
GNU_DIR = /path/to/petalinux/petalinux-v2018.3-final/tools/linux-i386/aarch64-linux-gnu
```

Adjust path to match your PetaLinux installation.

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
- Other utilities

**Edit `epics.bb` recipe:**

```bitbake
# EPICS binaries recipe
SUMMARY = "EPICS base binaries for ARM"
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
           file://casw \
           file://catime \
           file://softIoc \
          "

S = "${WORKDIR}"
RDEPENDS_${PN} += "epicslib"

do_install() {
    install -d ${D}/${bindir}
    install -m 0755 ${S}/acctst ${D}/${bindir}
    install -m 0755 ${S}/caConnTest ${D}/${bindir}
    install -m 0755 ${S}/caEventRate ${D}/${bindir}
    install -m 0755 ${S}/caget ${D}/${bindir}
    install -m 0755 ${S}/cainfo ${D}/${bindir}
    install -m 0755 ${S}/camonitor ${D}/${bindir}
    install -m 0755 ${S}/caput ${D}/${bindir}
    install -m 0755 ${S}/caRepeater ${D}/${bindir}
    install -m 0755 ${S}/casw ${D}/${bindir}
    install -m 0755 ${S}/catime ${D}/${bindir}
    install -m 0755 ${S}/softIoc ${D}/${bindir}
}
```

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

**Important Note:** Use non-versioned libraries (e.g., `libCom.so` instead of `libCom.so.3.15.5`) to avoid Yocto packaging issues.

If you have versioned libraries, create symlinks:
```bash
cd files/
ln -s libCom.so.3.15.5 libCom.so
ln -s libca.so.3.15.5 libca.so
# ... repeat for all libraries
```

**Edit `epicslib.bb` recipe:**

```bitbake
# EPICS libraries recipe
SUMMARY = "EPICS base libraries for ARM"
SECTION = "PETALINUX/apps"
LICENSE = "EPICS"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "file://libca.so \
           file://libcas.so \
           file://libCom.so \
           file://libdbCore.so \
           file://libdbRecStd.so \
           file://libgdd.so \
           file://libasyn.so \
           file://libstream.so \
          "

S = "${WORKDIR}"

do_install() {
    install -d ${D}/${libdir}
    install ${S}/libca.so ${D}/${libdir}
    install ${S}/libcas.so ${D}/${libdir}
    install ${S}/libCom.so ${D}/${libdir}
    install ${S}/libdbCore.so ${D}/${libdir}
    install ${S}/libdbRecStd.so ${D}/${libdir}
    install ${S}/libgdd.so ${D}/${libdir}
    install ${S}/libasyn.so ${D}/${libdir}
    install ${S}/libstream.so ${D}/${libdir}
}

FILES_${PN} += "${libdir}/*.so"
INSANE_SKIP_${PN} += "dev-so"
```

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
ldconfig -p | grep epics

# Test CA tools
softIoc -D $EPICS_BASE/dbd/softIoc.dbd
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

#### Step 1: Create the Layer Structure

Navigate to your Yocto build directory's sources folder:

```bash
cd /path/to/yocto-build/sources
```

Use the Yocto tool to create the layer:

```bash
bitbake-layers create-layer meta-epics
cd meta-epics
```

This creates the basic structure. Now customize it for EPICS:

```bash
mkdir -p recipes-epics/epics-base
mkdir -p recipes-epics/epics-asyn
mkdir -p recipes-epics/epics-stream
```

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
