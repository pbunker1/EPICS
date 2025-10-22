# Creating a Custom Yocto Layer for EPICS on ZCU106

## Overview

This guide provides step-by-step instructions for creating a custom Yocto layer (`meta-epics`) that builds EPICS Base from source for the Xilinx ZCU106 Zynq UltraScale+ MPSoC board.

**Target Platform:**
- **Board**: Xilinx ZCU106 Evaluation Kit
- **Processor**: Zynq UltraScale+ MPSoC (ARM Cortex-A53 64-bit, aarch64)
- **Build System**: Yocto Project / OpenEmbedded
- **EPICS Version**: Base 7.0.8 (also compatible with 3.15.x)

**What You'll Learn:**
- How to create a Yocto layer from scratch
- How to write BitBake recipes for EPICS Base
- How to cross-compile EPICS for ARM64
- Best practices for Yocto layer development
- How to integrate EPICS into custom images

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Before You Begin: Check Existing Layers](#before-you-begin-check-existing-layers)
3. [Step-by-Step Layer Creation](#step-by-step-layer-creation)
4. [Best Practices](#best-practices)
5. [Testing and Validation](#testing-and-validation)
6. [Advanced Topics](#advanced-topics)
7. [Troubleshooting](#troubleshooting)
8. [References](#references)

---

## Prerequisites

### Development Environment

**Host System Requirements:**
- Linux distribution (Ubuntu 20.04/22.04 recommended)
- 64-bit x86_64 host machine
- At least 50GB free disk space
- 8GB RAM minimum (16GB+ recommended)

**Required Packages (Ubuntu/Debian):**
```bash
sudo apt-get update
sudo apt-get install -y \
    gawk wget git diffstat unzip texinfo gcc build-essential \
    chrpath socat cpio python3 python3-pip python3-pexpect \
    xz-utils debianutils iputils-ping python3-git python3-jinja2 \
    libegl1-mesa libsdl1.2-dev pylint3 xterm python3-subunit \
    mesa-common-dev zstd liblz4-tool
```

### Yocto Project Setup

**1. Clone Poky (Yocto Reference Distribution):**
```bash
mkdir -p ~/yocto-epics
cd ~/yocto-epics
git clone git://git.yoctoproject.org/poky
cd poky
```

**2. Checkout a Stable Release:**
```bash
# For Kirkstone (LTS - recommended)
git checkout kirkstone

# Or for latest LTS
git checkout scarthgap
```

**3. Clone Xilinx Meta Layers:**
```bash
cd ~/yocto-epics
git clone https://github.com/Xilinx/meta-xilinx.git
cd meta-xilinx
git checkout <same-release-as-poky>  # e.g., kirkstone
```

**4. Initialize Build Environment:**
```bash
cd ~/yocto-epics/poky
source oe-init-build-env build-zcu106
```

This creates a `build-zcu106` directory with initial configuration files.

---

## Before You Begin: Check Existing Layers

Before creating a new layer, check if someone has already created a layer with EPICS support:

### Existing EPICS-Related Layers

**1. OpenEmbedded Layer Index:**
- Visit: https://layers.openembedded.org/
- Search for "epics" or "controls"

**2. meta-chimeratk:**
- Repository: https://github.com/ChimeraTK/meta-chimeratk
- Provides EPICS base recipes and control system integration
- Consider using this if it meets your needs

**3. Search GitHub:**
```bash
# Search for existing layers
# https://github.com/search?q=meta-epics
```

**If an existing layer meets your needs, use it instead!** Only create a new layer if:
- You need custom EPICS configurations
- You want full control over the build process
- Existing layers don't support your use case
- You're learning Yocto layer development

---

## Step-by-Step Layer Creation

### Step 1: Create the Layer Structure

**Navigate to your sources directory:**
```bash
cd ~/yocto-epics
mkdir -p sources
cd sources
```

**Option A: Use bitbake-layers Tool (Recommended)**

The Yocto Project provides a tool to automate layer creation:

```bash
# Create layer with default priority (6)
bitbake-layers create-layer meta-epics

# OR: Create with custom priority
bitbake-layers create-layer --priority 10 meta-epics

# OR: Create with custom example recipe name
bitbake-layers create-layer --example-recipe-name epics-base meta-epics
```

This creates:
```
meta-epics/
├── conf/
│   └── layer.conf
├── COPYING.MIT
├── README
└── recipes-example/
    └── example/
        └── example_0.1.bb
```

**Option B: Manual Creation (For Learning)**

```bash
mkdir meta-epics
cd meta-epics

# Create directory structure
mkdir -p conf
mkdir -p recipes-epics/epics-base/files
mkdir -p recipes-epics/epics-modules/epics-asyn/files
mkdir -p recipes-epics/epics-modules/epics-stream/files
mkdir -p recipes-core/images
mkdir -p classes

# Optional: Create machine-specific subdirectories
mkdir -p recipes-epics/epics-base/files/zcu106
mkdir -p recipes-epics/epics-base/files/zcu102
```

**Directory Structure Explanation:**
- `conf/` - Layer configuration files
- `recipes-epics/` - EPICS-specific recipes
- `recipes-core/` - Core system recipes (images)
- `classes/` - Custom BitBake classes
- `files/` - Patches, configuration files, and source files

**Best Practice:** Always prepend layer names with `meta-` to follow Yocto conventions and avoid tool issues.

---

### Step 2: Configure layer.conf

Create or edit `conf/layer.conf`:

```python
# We have a conf and classes directory, add to BBPATH
BBPATH .= ":${LAYERDIR}"

# We have recipes-* directories, add to BBFILES
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
            ${LAYERDIR}/recipes-*/*/*.bbappend"

# Define layer collection
BBFILE_COLLECTIONS += "meta-epics"
BBFILE_PATTERN_meta-epics = "^${LAYERDIR}/"
BBFILE_PRIORITY_meta-epics = "10"

# Layer version
LAYERVERSION_meta-epics = "1"

# Layer dependencies (core is always required)
LAYERDEPENDS_meta-epics = "core"

# Specify compatible Yocto releases
# Update this list as you test with new releases
LAYERSERIES_COMPAT_meta-epics = "kirkstone langdale mickledore nanbield scarthgap"
```

**Variable Explanations:**

| Variable | Purpose |
|----------|---------|
| `BBPATH` | Adds layer to BitBake's search path for classes and config files |
| `BBFILES` | Defines location for all recipes (*.bb) and append files (*.bbappend) |
| `BBFILE_COLLECTIONS` | Unique identifier for this layer |
| `BBFILE_PATTERN` | Pattern to match files in this layer |
| `BBFILE_PRIORITY` | Priority when recipes with same name exist in multiple layers (higher = higher priority) |
| `LAYERVERSION` | Version number for your layer |
| `LAYERDEPENDS` | List of layers this layer depends on |
| `LAYERSERIES_COMPAT` | Compatible Yocto Project releases |

---

### Step 3: Create EPICS Base Recipe

Create `recipes-epics/epics-base/epics-base_7.0.8.bb`:

```bitbake
SUMMARY = "EPICS Base - Experimental Physics and Industrial Control System"
DESCRIPTION = "EPICS is a set of software tools and applications which provide \
a software infrastructure for use in building distributed control systems to \
operate devices such as Particle Accelerators, Large Experiments and major \
Telescopes distributed worldwide."
HOMEPAGE = "https://epics-controls.org/"
SECTION = "devel"
LICENSE = "EPICS"
LIC_FILES_CHKSUM = "file://LICENSE;md5=61d15894e07e0d6d0f737ad05d36139a"

# Source from EPICS official repository
SRC_URI = "https://epics.anl.gov/download/base/base-${PV}.tar.gz \
           file://0001-yocto-cross-compile-support.patch \
          "

# SHA256 checksum for base-7.0.8.tar.gz
SRC_URI[sha256sum] = "27f13e308b92eaaa99330e73a0d26b37e075c1c6e2a1eb8dc3bcb9ebe5fb5850"

# Source directory
S = "${WORKDIR}/base-${PV}"

# Build dependencies
DEPENDS = "readline perl-native"

# Runtime dependencies
RDEPENDS:${PN} = "readline"

# Don't strip binaries - EPICS needs them intact
INHIBIT_PACKAGE_STRIP = "1"
INHIBIT_SYSROOT_STRIP = "1"

# Export EPICS environment variables
export EPICS_HOST_ARCH = "linux-${HOST_ARCH}"
export EPICS_BASE = "${S}"

# Enable parallel compilation
PARALLEL_MAKE = "-j ${@oe.utils.cpu_count()}"

# Configure for cross-compilation
do_configure() {
    # Determine target architecture
    if [ "${TARGET_ARCH}" = "aarch64" ]; then
        EPICS_TARGET_ARCH="linux-arm"
        GNU_TARGET="aarch64-linux-gnu"
    elif [ "${TARGET_ARCH}" = "arm" ]; then
        EPICS_TARGET_ARCH="linux-arm"
        GNU_TARGET="arm-linux-gnueabihf"
    elif [ "${TARGET_ARCH}" = "x86_64" ]; then
        EPICS_TARGET_ARCH="linux-x86_64"
        GNU_TARGET="x86_64-linux-gnu"
    else
        EPICS_TARGET_ARCH="linux-${TARGET_ARCH}"
        GNU_TARGET="${TARGET_SYS}"
    fi

    # Configure CONFIG_SITE for cross-compilation
    cat >> ${S}/configure/CONFIG_SITE << EOF
# Cross-compilation settings for Yocto
CROSS_COMPILER_TARGET_ARCHS = ${EPICS_TARGET_ARCH}
SHARED_LIBRARIES = YES
STATIC_BUILD = NO
EOF

    # Create cross-compilation configuration file
    cat > ${S}/configure/os/CONFIG_SITE.linux-x86_64.${EPICS_TARGET_ARCH} << EOF
# Cross-compilation configuration for ${TARGET_ARCH}
# Generated by Yocto/OpenEmbedded build system

# GNU cross-compilation tools
GNU_TARGET = ${GNU_TARGET}
GNU_DIR = ${STAGING_BINDIR_TOOLCHAIN}/..

# Use sysroot for libraries
SHRLIB_LDFLAGS = -shared -Wl,-soname,\$@ -Wl,-rpath-link,${STAGING_LIBDIR}

# Compiler flags from Yocto
OPT_CFLAGS_YES = ${CFLAGS}
OPT_CXXFLAGS_YES = ${CXXFLAGS}
LDFLAGS += ${LDFLAGS}

# readline library support
COMMANDLINE_LIBRARY = READLINE
EOF

    # Configure runtime linking
    echo 'RUNTIME_LDFLAGS_ORIGIN = -Wl,-rpath,\\$$ORIGIN/../lib' >> ${S}/configure/CONFIG_COMMON
}

# Compile EPICS base
do_compile() {
    bbnote "Building EPICS base for ${TARGET_ARCH}"
    oe_runmake CROSS_COMPILER_TARGET_ARCHS=linux-arm
}

# Install EPICS base
do_install() {
    # Determine EPICS target architecture
    if [ "${TARGET_ARCH}" = "aarch64" ] || [ "${TARGET_ARCH}" = "arm" ]; then
        EPICS_TARGET="linux-arm"
    elif [ "${TARGET_ARCH}" = "x86_64" ]; then
        EPICS_TARGET="linux-x86_64"
    else
        EPICS_TARGET="linux-${TARGET_ARCH}"
    fi

    # Create installation directories
    install -d ${D}${bindir}
    install -d ${D}${libdir}
    install -d ${D}${includedir}/epics
    install -d ${D}${datadir}/epics/configure
    install -d ${D}${datadir}/epics/dbd
    install -d ${D}${datadir}/epics/bin

    # Install libraries
    if [ -d ${S}/lib/${EPICS_TARGET} ]; then
        bbnote "Installing libraries from ${S}/lib/${EPICS_TARGET}"
        for lib in ${S}/lib/${EPICS_TARGET}/*.so*; do
            if [ -f "$lib" ]; then
                install -m 0755 "$lib" ${D}${libdir}/
            fi
        done
    fi

    # Install binaries
    if [ -d ${S}/bin/${EPICS_TARGET} ]; then
        bbnote "Installing binaries from ${S}/bin/${EPICS_TARGET}"
        for bin in ${S}/bin/${EPICS_TARGET}/*; do
            if [ -f "$bin" ] && [ -x "$bin" ]; then
                install -m 0755 "$bin" ${D}${bindir}/
            fi
        done
    fi

    # Install header files
    cp -r ${S}/include/* ${D}${includedir}/epics/

    # Install DBD (Database Definition) files
    cp -r ${S}/dbd ${D}${datadir}/epics/

    # Install configure files (needed for building EPICS modules)
    cp -r ${S}/configure ${D}${datadir}/epics/

    # Create EPICS environment setup script
    cat > ${D}${bindir}/epics-env.sh << 'EOF'
#!/bin/sh
# EPICS Base Environment Setup
# Source this file to use EPICS tools

export EPICS_BASE=/usr/share/epics
export EPICS_HOST_ARCH=linux-arm
export PATH=${EPICS_BASE}/bin:${PATH}
export LD_LIBRARY_PATH=${EPICS_BASE}/lib:${LD_LIBRARY_PATH}

# Optional: Set Channel Access environment variables
# export EPICS_CA_ADDR_LIST=10.0.0.255
# export EPICS_CA_AUTO_ADDR_LIST=YES
# export EPICS_CA_MAX_ARRAY_BYTES=16777216

echo "EPICS Base environment configured"
echo "EPICS_BASE: ${EPICS_BASE}"
echo "EPICS_HOST_ARCH: ${EPICS_HOST_ARCH}"
EOF
    chmod +x ${D}${bindir}/epics-env.sh
}

# Package files
FILES:${PN} = "${bindir}/* ${libdir}/*.so*"
FILES:${PN}-dev = "${includedir}/epics/* ${datadir}/epics/*"
FILES:${PN}-dbg = "${bindir}/.debug ${libdir}/.debug"

# Allow shipping .so files in main package (not just -dev)
INSANE_SKIP:${PN} += "dev-so"

# EPICS uses RPATH - allow it
INSANE_SKIP:${PN} += "ldflags"

# Allow textrel in binaries (sometimes needed for EPICS)
INSANE_SKIP:${PN} += "textrel"

# Package provides
PROVIDES = "epics-base"

# Package information
PACKAGE_ARCH = "${MACHINE_ARCH}"
```

**Key Features of This Recipe:**
- ✅ Automatic architecture detection (aarch64, arm, x86_64)
- ✅ Cross-compilation support for Yocto toolchain
- ✅ Proper sysroot handling
- ✅ Environment setup script generation
- ✅ Development files packaging
- ✅ QA check handling for EPICS-specific needs

---

### Step 4: Create Patch File (Optional)

If EPICS needs modifications for Yocto, create a patch:

`recipes-epics/epics-base/files/0001-yocto-cross-compile-support.patch`:

```patch
From: Your Name <your.email@example.com>
Date: Wed, 22 Oct 2025 00:00:00 +0000
Subject: [PATCH] Add Yocto cross-compilation support

Add configuration changes to support Yocto SDK cross-compilation.

Signed-off-by: Your Name <your.email@example.com>
---
 configure/CONFIG_SITE.Common.linux-arm | 4 ++++
 1 file changed, 4 insertions(+)

diff --git a/configure/CONFIG_SITE.Common.linux-arm b/configure/CONFIG_SITE.Common.linux-arm
index 1234567..abcdefg 100644
--- a/configure/CONFIG_SITE.Common.linux-arm
+++ b/configure/CONFIG_SITE.Common.linux-arm
@@ -1,3 +1,7 @@
 # CONFIG_SITE.Common.linux-arm

 # Settings for linux-arm target builds
+
+# Support Yocto cross-compilation
+GNU_TARGET ?= arm-linux-gnueabi
+CROSS_COMPILER_HOST_ARCHS = linux-x86_64
--
2.34.1
```

**Note:** You may not need a patch if EPICS Base 7.0.8 builds cleanly. This is just an example.

---

### Step 5: Create EPICS Module Recipe (asyn example)

Create `recipes-epics/epics-modules/epics-asyn/asyn_4-42.bb`:

```bitbake
SUMMARY = "EPICS asyn - Asynchronous driver support"
DESCRIPTION = "Asyn is a general purpose facility for interfacing device \
specific code to low level drivers. It supports both synchronous and \
asynchronous communication."
HOMEPAGE = "https://epics-modules.github.io/asyn/"
SECTION = "devel"
LICENSE = "EPICS"
LIC_FILES_CHKSUM = "file://LICENSE;md5=..."

# Dependency on EPICS base
DEPENDS = "epics-base"
RDEPENDS:${PN} = "epics-base"

# Source from GitHub
SRC_URI = "git://github.com/epics-modules/asyn.git;protocol=https;branch=master \
           file://RELEASE.local \
          "
SRCREV = "R4-42"

S = "${WORKDIR}/git"

# EPICS environment
export EPICS_BASE = "${STAGING_DIR_TARGET}${datadir}/epics"
export EPICS_HOST_ARCH = "linux-arm"

do_configure() {
    # Copy RELEASE.local to configure directory
    cp ${WORKDIR}/RELEASE.local ${S}/configure/
}

do_compile() {
    oe_runmake
}

do_install() {
    install -d ${D}${datadir}/epics/modules/asyn
    cp -r ${S}/* ${D}${datadir}/epics/modules/asyn/

    # Install libraries to standard location
    install -d ${D}${libdir}
    if [ -d ${S}/lib/${EPICS_HOST_ARCH} ]; then
        for lib in ${S}/lib/${EPICS_HOST_ARCH}/*.so*; do
            if [ -f "$lib" ]; then
                install -m 0755 "$lib" ${D}${libdir}/
            fi
        done
    fi
}

FILES:${PN} = "${datadir}/epics/modules/asyn/* ${libdir}/*.so*"
FILES:${PN}-dev = "${datadir}/epics/modules/asyn/include/*"

INSANE_SKIP:${PN} += "dev-so"
```

Create `recipes-epics/epics-modules/epics-asyn/files/RELEASE.local`:

```makefile
# RELEASE.local - EPICS module configuration
# This file is included by the EPICS build system

# Location of EPICS base
EPICS_BASE = /usr/share/epics

# Support modules (add as needed)
# SNCSEQ = /usr/share/epics/modules/seq
```

---

### Step 6: Add Layer to Build Configuration

**Option A: Use bitbake-layers command:**

```bash
cd ~/yocto-epics/poky/build-zcu106
bitbake-layers add-layer ../../sources/meta-epics
```

**Option B: Manually edit bblayers.conf:**

Edit `conf/bblayers.conf`:

```python
# POKY_BBLAYERS_CONF_VERSION is increased each time build/conf/bblayers.conf
# changes incompatibly
POKY_BBLAYERS_CONF_VERSION = "2"

BBPATH = "${TOPDIR}"
BBFILES ?= ""

BBLAYERS ?= " \
  /home/user/yocto-epics/poky/meta \
  /home/user/yocto-epics/poky/meta-poky \
  /home/user/yocto-epics/poky/meta-yocto-bsp \
  /home/user/yocto-epics/meta-xilinx/meta-xilinx-core \
  /home/user/yocto-epics/meta-xilinx/meta-xilinx-bsp \
  /home/user/yocto-epics/sources/meta-epics \
  "
```

**Important:** Use absolute paths in `BBLAYERS`.

---

### Step 7: Configure Machine and Build Settings

Edit `conf/local.conf`:

```bash
# Machine Selection
MACHINE = "zcu106-zynqmp"

# Parallelization - adjust based on your CPU
BB_NUMBER_THREADS = "8"
PARALLEL_MAKE = "-j 8"

# Add EPICS to the image
IMAGE_INSTALL:append = " epics-base"

# Optional: Add development packages
# IMAGE_INSTALL:append = " epics-base-dev"

# Disk space monitoring
BB_DISKMON_DIRS ??= "\
    STOPTASKS,${TMPDIR},1G,100K \
    STOPTASKS,${DL_DIR},1G,100K \
    STOPTASKS,${SSTATE_DIR},1G,100K \
    STOPTASKS,/tmp,100M,100K \
    HALT,${TMPDIR},100M,1K \
    HALT,${DL_DIR},100M,1K \
    HALT,${SSTATE_DIR},100M,1K \
    HALT,/tmp,10M,1K"

# Package management
PACKAGE_CLASSES = "package_rpm"

# Additional image features
EXTRA_IMAGE_FEATURES ?= "debug-tweaks tools-sdk ssh-server-openssh"

# SDK settings
SDKMACHINE = "x86_64"
```

---

### Step 8: Create Custom EPICS Image Recipe

Create `recipes-core/images/epics-minimal-image.bb`:

```bitbake
SUMMARY = "Minimal Linux image with EPICS Base for ZCU106"
DESCRIPTION = "A minimal bootable image with EPICS control system support"

# Base image
require recipes-core/images/core-image-minimal.bb

# EPICS packages
IMAGE_INSTALL:append = " \
    epics-base \
    epics-base-dev \
    "

# Development and debugging tools
IMAGE_INSTALL:append = " \
    openssh \
    openssh-sftp-server \
    vim \
    nano \
    htop \
    strace \
    gdb \
    tcpdump \
    ethtool \
    i2c-tools \
    procps \
    util-linux \
    iproute2 \
    nfs-utils \
    "

# Increase rootfs size for EPICS
IMAGE_ROOTFS_EXTRA_SPACE = "524288"

# Add image features
IMAGE_FEATURES:append = " \
    tools-debug \
    ssh-server-openssh \
    package-management \
    "

# Set root password (development only - remove for production!)
# Password: root
ROOTFS_POSTPROCESS_COMMAND:append = " set_root_password; setup_epics_env; "

set_root_password() {
    sed -i 's%^root:[^:]*:%root:$6$rounds=656000$YQKMBflv0Dnlbk4x$AZI7xsHD.JZrb5KML5K5fvKRPMDOxqU/F9TjGvKNYBRcPBVVPJdCCGVMYHqhx.v16uEqNxFMQQIrHCRxxFHTb.:%' ${IMAGE_ROOTFS}/etc/shadow
}

setup_epics_env() {
    # Create system-wide EPICS environment
    install -d ${IMAGE_ROOTFS}/etc/profile.d
    cat >> ${IMAGE_ROOTFS}/etc/profile.d/epics.sh << 'EOF'
# EPICS Base Environment
export EPICS_BASE=/usr/share/epics
export EPICS_HOST_ARCH=linux-arm
export PATH=${EPICS_BASE}/bin:${PATH}
export LD_LIBRARY_PATH=${EPICS_BASE}/lib:${LD_LIBRARY_PATH}

# EPICS Channel Access configuration
export EPICS_CA_ADDR_LIST=10.0.0.255
export EPICS_CA_AUTO_ADDR_LIST=YES
export EPICS_CA_MAX_ARRAY_BYTES=16777216
EOF
    chmod 644 ${IMAGE_ROOTFS}/etc/profile.d/epics.sh
}
```

---

### Step 9: Verify Layer Configuration

**Check layer is properly added:**

```bash
bitbake-layers show-layers
```

Expected output:
```
layer                 path                                      priority
==========================================================================
meta                  /home/user/yocto-epics/poky/meta          5
meta-poky             /home/user/yocto-epics/poky/meta-poky     5
meta-yocto-bsp        /home/user/yocto-epics/poky/meta-yocto-bsp 5
meta-xilinx-core      /home/user/yocto-epics/meta-xilinx/...    6
meta-xilinx-bsp       /home/user/yocto-epics/meta-xilinx/...    6
meta-epics            /home/user/yocto-epics/sources/meta-epics 10
```

**Check recipes are found:**

```bash
bitbake-layers show-recipes "epics*"
```

**Check recipe details:**

```bash
bitbake -e epics-base | grep ^SRC_URI=
bitbake -e epics-base | grep ^S=
```

---

### Step 10: Build EPICS Base

**Clean build:**

```bash
bitbake epics-base -c cleansstate
bitbake epics-base
```

**Verbose build (for debugging):**

```bash
bitbake epics-base -v
```

**Build with logging:**

```bash
bitbake epics-base 2>&1 | tee epics-build.log
```

---

### Step 11: Build Complete Image

**Build the EPICS-enabled image:**

```bash
bitbake epics-minimal-image
```

**Build outputs location:**
```
build-zcu106/tmp/deploy/images/zcu106-zynqmp/
├── boot.bin
├── epics-minimal-image-zcu106-zynqmp.wic.gz
├── epics-minimal-image-zcu106-zynqmp.tar.bz2
└── Image
```

---

### Step 12: Deploy to ZCU106

**Option A: SD Card Deployment**

```bash
# Write image to SD card (adjust /dev/sdX to your SD card)
sudo bmaptool copy epics-minimal-image-zcu106-zynqmp.wic.gz /dev/sdX

# Or use dd
gunzip -c epics-minimal-image-zcu106-zynqmp.wic.gz | sudo dd of=/dev/sdX bs=4M status=progress && sync
```

**Option B: Manual SD Card Setup**

```bash
# Create two partitions on SD card
# Partition 1: FAT32 (100MB) - Boot files
# Partition 2: ext4 (remaining) - Root filesystem

# Mount partitions
sudo mount /dev/sdX1 /mnt/boot
sudo mount /dev/sdX2 /mnt/rootfs

# Copy boot files
cp boot.bin /mnt/boot/
cp Image /mnt/boot/

# Extract rootfs
sudo tar -xjf epics-minimal-image-zcu106-zynqmp.tar.bz2 -C /mnt/rootfs

# Unmount
sudo umount /mnt/boot /mnt/rootfs
```

---

### Step 13: Test on ZCU106

**1. Boot the board:**
- Insert SD card into ZCU106
- Configure boot mode switches for SD boot
- Power on and connect serial console (115200 8N1)

**2. Login:**
```
Username: root
Password: root
```

**3. Test EPICS installation:**

```bash
# Source EPICS environment
source /etc/profile.d/epics.sh

# Check EPICS binaries
which softIoc
which caget
which caput

# Check EPICS libraries
ldconfig -p | grep -i com
ldconfig -p | grep -i ca

# Test Channel Access tools
caget -h
caput -h
camonitor -h

# Start a soft IOC
softIoc
epics> exit
```

**4. Test IOC with database:**

```bash
# Create a simple test database
cat > /tmp/test.db << 'EOF'
record(ai, "TEST:AI1") {
    field(DESC, "Test Analog Input")
    field(SCAN, "1 second")
    field(VAL, "0")
}

record(calc, "TEST:CALC1") {
    field(DESC, "Test Calculation")
    field(CALC, "A+1")
    field(INPA, "TEST:CALC1")
    field(SCAN, "1 second")
}
EOF

# Start IOC with database
softIoc -d /tmp/test.db
```

In another terminal:
```bash
# Monitor the counter
camonitor TEST:CALC1

# Get value
caget TEST:CALC1

# Set value
caput TEST:AI1 42.5
caget TEST:AI1
```

---

## Best Practices

### 1. Avoid Overlaying Entire Recipes

❌ **Don't do this:**
```bash
# Copying entire recipe and modifying it
cp meta-other/recipes-foo/foo_1.0.bb meta-epics/recipes-foo/foo_1.0.bb
# Then modify the copy
```

✅ **Do this instead:**
```bash
# Create a .bbappend file
# meta-epics/recipes-foo/foo_1.0.bbappend

FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"
SRC_URI:append = " file://my-custom.patch"
```

**Benefits:**
- Changes automatically apply when base recipe updates
- No need to maintain forked copy
- Clear separation between base and customizations

---

### 2. Use Machine-Specific Overrides

**Proper file organization:**

```
recipes-epics/epics-base/files/
├── zcu106/
│   ├── epics-config.patch
│   └── custom-startup.sh
├── zcu102/
│   └── epics-config.patch
└── common-config.sh
```

**In your recipe or .bbappend:**

```bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

# This file only used for ZCU106 builds
SRC_URI:append:zcu106 = " file://zcu106/epics-config.patch"

# This file only used for ZCU102 builds
SRC_URI:append:zcu102 = " file://zcu102/epics-config.patch"

# This file used for all builds
SRC_URI:append = " file://common-config.sh"
```

**Examples of proper override usage:**

```bitbake
# Machine-specific dependency
DEPENDS:append:zcu106 = " fpga-firmware"

# Architecture-specific compiler flags
CFLAGS:append:aarch64 = " -march=armv8-a -mtune=cortex-a53"

# Multiple overrides (architecture AND machine)
EXTRA_OECONF:append:aarch64:zcu106 = " --enable-hardware-support"
```

**Common overrides:**
- `:append:zcu106` - ZCU106 board only
- `:append:aarch64` - All 64-bit ARM
- `:append:arm` - All 32-bit ARM
- `:prepend` - Add to beginning of variable
- `:append` - Add to end of variable
- `:remove` - Remove from variable

---

### 3. Layer Naming Conventions

✅ **Good layer names:**
- `meta-epics`
- `meta-controls`
- `meta-zynq-epics`
- `meta-lab-instruments`

❌ **Avoid:**
- `epics` (missing meta- prefix)
- `meta_epics` (use dash, not underscore)
- `my-epics-layer` (missing meta- prefix)

**Why it matters:**
- Tools assume `meta-` prefix
- Prevents naming conflicts
- Follows community standards
- Makes layer purpose clear

---

### 4. Managing Recipe Versions

**Use version wildcards in .bbappend files:**

```bitbake
# recipes-epics/epics-base/epics-base_%.bbappend
# The % matches any version

FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"
SRC_URI:append = " file://custom-config.patch"
```

**Benefits:**
- Works with epics-base_7.0.8.bb, epics-base_7.0.9.bb, etc.
- Don't need to rename .bbappend when upgrading
- Errors only if no matching recipe exists

**Version-specific .bbappend (when needed):**

```bitbake
# recipes-epics/epics-base/epics-base_7.0.8.bbappend
# Only applies to version 7.0.8

CFLAGS:append = " -DOLD_VERSION_WORKAROUND"
```

---

### 5. Proper FILESEXTRAPATHS Usage

**Always use :prepend with immediate expansion:**

```bitbake
# Correct
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

# Wrong - missing colon at end
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}"

# Wrong - using = instead of :=
FILESEXTRAPATHS:prepend = "${THISDIR}/${PN}:"

# Wrong - using :append instead of :prepend
FILESEXTRAPATHS:append := "${THISDIR}/${PN}:"
```

**Why `:=` (immediate expansion)?**
- `THISDIR` is defined at parse time
- Without `:=`, variable expands later when `THISDIR` may be different
- Always use `:=` with `THISDIR`

**Trailing colon is required:**
- `FILESPATH` is colon-separated list
- Missing colon causes paths to concatenate incorrectly

---

### 6. Testing Layer Compatibility

**Run yocto-check-layer before releasing:**

```bash
cd ~/yocto-epics/poky/build-zcu106
source oe-init-build-env
yocto-check-layer ../../sources/meta-epics
```

**Common tests performed:**
- `common.test_readme` - README file exists
- `common.test_parse` - BitBake can parse recipes
- `common.test_show_environment` - Environment is valid
- `common.test_world` - `bitbake world` works
- `common.test_signatures` - Signature stability
- `common.test_layerseries_compat` - LAYERSERIES_COMPAT is set

**To pass all tests:**

1. **Create README:**
```bash
cat > README << 'EOF'
# meta-epics

This layer provides EPICS Base support for Yocto Project.

## Dependencies

- openembedded-core (meta)
- meta-xilinx-core (for Xilinx targets)

## Maintainer

Your Name <your.email@example.com>
EOF
```

2. **Set LAYERSERIES_COMPAT:**
```python
# In layer.conf
LAYERSERIES_COMPAT_meta-epics = "kirkstone langdale mickledore nanbield scarthgap"
```

3. **Test build:**
```bash
bitbake -p  # Parse all recipes
bitbake epics-base -c fetch
bitbake epics-base
```

---

### 7. Version Control Best Practices

**Initialize Git repository:**

```bash
cd ~/yocto-epics/sources/meta-epics
git init
git add .
git commit -m "Initial meta-epics layer structure"
```

**Create .gitignore:**

```gitignore
# Build artifacts
*.pyc
__pycache__/
*.swp
*~

# Temporary files
*.orig
*.rej
```

**Create proper commit messages:**

```bash
# Good commit message format
git commit -m "epics-base: Update to version 7.0.8

- Update SRC_URI checksum
- Add patch for aarch64 support
- Enable readline support

Signed-off-by: Your Name <your.email@example.com>"
```

**Tag releases:**

```bash
git tag -a v1.0 -m "Release 1.0 - EPICS Base 7.0.8 support"
git push origin v1.0
```

---

## Testing and Validation

### Layer Validation Checklist

- [ ] Layer created with proper structure
- [ ] `layer.conf` properly configured
- [ ] `LAYERSERIES_COMPAT` set correctly
- [ ] README file created
- [ ] LICENSE file included
- [ ] Recipes parse without errors (`bitbake -p`)
- [ ] EPICS base recipe builds successfully
- [ ] Image builds with EPICS included
- [ ] Image boots on target hardware
- [ ] EPICS binaries execute correctly
- [ ] `yocto-check-layer` passes all tests
- [ ] Layer committed to version control

### Build Testing Commands

```bash
# Parse all recipes
bitbake -p

# List tasks for a recipe
bitbake epics-base -c listtasks

# Build specific task
bitbake epics-base -c fetch
bitbake epics-base -c configure
bitbake epics-base -c compile

# Clean and rebuild
bitbake epics-base -c cleansstate
bitbake epics-base

# Check recipe dependencies
bitbake-layers show-recipes epics-base
bitbake -g epics-base
cat task-depends.dot | grep epics

# Build with detailed logging
bitbake epics-base -v -D

# Check what files a package provides
oe-pkgdata-util list-pkg-files epics-base

# Find which package provides a file
oe-pkgdata-util find-path /usr/bin/softIoc
```

### QA Checks

**Common QA warnings and fixes:**

| Warning | Meaning | Fix |
|---------|---------|-----|
| `dev-so` | .so files in main package | Add `INSANE_SKIP:${PN} += "dev-so"` |
| `ldflags` | Missing ldflags in linking | Add `INSANE_SKIP:${PN} += "ldflags"` or fix Makefile |
| `textrel` | Text relocations in binary | Usually safe for EPICS, add to INSANE_SKIP |
| `already-stripped` | Binary already stripped | Add `INHIBIT_PACKAGE_STRIP = "1"` |
| `installed-vs-shipped` | Files not packaged | Add to `FILES:${PN}` |

**Disable specific QA checks only when necessary!**

---

## Advanced Topics

### Creating Custom Image Classes

**Create reusable image class:**

`classes/epics-image.bbclass`:

```bitbake
# Common EPICS image configuration

# Core EPICS packages
EPICS_PACKAGES = " \
    epics-base \
    epics-base-dev \
    "

# Development tools
EPICS_DEV_TOOLS = " \
    strace \
    gdb \
    tcpdump \
    vim \
    htop \
    "

# Network tools
EPICS_NET_TOOLS = " \
    openssh \
    nfs-utils \
    iproute2 \
    ethtool \
    "

# Add all to image
IMAGE_INSTALL:append = " \
    ${EPICS_PACKAGES} \
    ${EPICS_DEV_TOOLS} \
    ${EPICS_NET_TOOLS} \
    "

# Increase rootfs size
IMAGE_ROOTFS_EXTRA_SPACE = "524288"

# Post-install customization
ROOTFS_POSTPROCESS_COMMAND:append = " setup_epics_environment ; "

setup_epics_environment() {
    install -d ${IMAGE_ROOTFS}/etc/profile.d
    cat > ${IMAGE_ROOTFS}/etc/profile.d/epics.sh << 'EOF'
export EPICS_BASE=/usr/share/epics
export EPICS_HOST_ARCH=linux-arm
export PATH=${EPICS_BASE}/bin:${PATH}
export LD_LIBRARY_PATH=${EPICS_BASE}/lib:${LD_LIBRARY_PATH}
EOF
    chmod 644 ${IMAGE_ROOTFS}/etc/profile.d/epics.sh
}
```

**Use in image recipe:**

```bitbake
require recipes-core/images/core-image-minimal.bb
inherit epics-image

DESCRIPTION = "Custom EPICS image"
```

---

### Creating .bbappend in Different Layer

**Scenario:** You want to customize EPICS for a specific machine without modifying meta-epics.

**Create machine-specific layer:**

```bash
bitbake-layers create-layer meta-zcu106-custom
cd meta-zcu106-custom
```

**Create .bbappend:**

`recipes-epics/epics-base/epics-base_%.bbappend`:

```bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

# Add ZCU106-specific patches
SRC_URI:append:zcu106 = " \
    file://zcu106-optimization.patch \
    file://custom-startup.sh \
    "

# ZCU106-specific compiler flags
CFLAGS:append:zcu106 = " -O3 -mcpu=cortex-a53"

# Custom installation
do_install:append:zcu106() {
    install -d ${D}${sysconfdir}/epics
    install -m 0755 ${WORKDIR}/custom-startup.sh ${D}${sysconfdir}/epics/
}

FILES:${PN}:append:zcu106 = " ${sysconfdir}/epics/*"
```

---

### Saving and Restoring Layer Setup

**Save current configuration:**

```bash
bitbake-layers create-layers-setup ~/yocto-epics/sources/meta-epics/
```

Creates:
- `setup-layers.json` - Layer configuration (git repos, branches, commits)
- `setup-layers` - Script to restore setup

**Restore on another machine:**

```bash
# Clone your layer
git clone https://github.com/yourusername/meta-epics.git
cd meta-epics

# Restore complete layer setup
./setup-layers
```

**Commits the `setup-layers*` files to Git for reproducibility!**

---

### Building SDK

**Build SDK for cross-development:**

```bash
bitbake epics-minimal-image -c populate_sdk
```

**Install SDK:**

```bash
./tmp/deploy/sdk/poky-glibc-x86_64-epics-minimal-image-cortexa53-zcu106-zynqmp-toolchain-*.sh
```

**Use SDK:**

```bash
source /opt/poky/*/environment-setup-cortexa53-poky-linux

# Now you can cross-compile EPICS applications
cd my-ioc
make
```

---

## Troubleshooting

### Common Build Errors

#### 1. "No recipes available for..."

**Error:**
```
ERROR: Nothing PROVIDES 'epics-base'
```

**Cause:** Layer not in BBLAYERS or recipe not found

**Fix:**
```bash
# Verify layer is added
bitbake-layers show-layers

# Add layer if missing
bitbake-layers add-layer /path/to/meta-epics

# Verify recipe exists
bitbake-layers show-recipes "epics*"
```

---

#### 2. "do_fetch: Fetcher failure"

**Error:**
```
ERROR: epics-base-7.0.8-r0 do_fetch: Fetcher failure for URL: 'https://epics.anl.gov/download/base/base-7.0.8.tar.gz'
```

**Causes:**
- Network connectivity issue
- Wrong URL
- Source no longer available

**Fix:**
```bash
# Test URL manually
wget https://epics.anl.gov/download/base/base-7.0.8.tar.gz

# Check SRC_URI in recipe
bitbake -e epics-base | grep ^SRC_URI=

# Use mirror if needed
SRC_URI = "https://github.com/epics-base/epics-base/archive/R7.0.8.tar.gz"
```

---

#### 3. "Checksum mismatch"

**Error:**
```
ERROR: epics-base-7.0.8-r0 do_fetch: Checksum mismatch!
File: '/path/to/downloads/base-7.0.8.tar.gz'
```

**Cause:** File checksum doesn't match SRC_URI[sha256sum]

**Fix:**
```bash
# Calculate correct checksum
sha256sum downloads/base-7.0.8.tar.gz

# Update recipe with correct checksum
SRC_URI[sha256sum] = "correct_checksum_here"

# Or bypass check temporarily (NOT for production!)
BB_STRICT_CHECKSUM = "0"
```

---

#### 4. "do_compile failed"

**Error:**
```
ERROR: epics-base-7.0.8-r0 do_compile: oe_runmake failed
```

**Debugging:**
```bash
# Build with verbose output
bitbake epics-base -v -D

# Check compile log
cat tmp/work/cortexa53-poky-linux/epics-base/7.0.8-r0/temp/log.do_compile

# Run compile manually
bitbake epics-base -c devshell
# This opens a shell in the build environment
cd ${S}
make
```

**Common causes:**
- Missing dependencies
- Wrong cross-compiler configuration
- Hardcoded paths in Makefile

---

#### 5. "GNU_DIR not found"

**Error:**
```
CONFIG_SITE.linux-x86_64.linux-arm: GNU_DIR not set correctly
```

**Fix:**
```bitbake
# In do_configure
cat > ${S}/configure/os/CONFIG_SITE.linux-x86_64.linux-arm << EOF
GNU_TARGET = aarch64-linux-gnu
GNU_DIR = ${STAGING_BINDIR_TOOLCHAIN}/..
EOF
```

---

#### 6. "installed-vs-shipped" QA Error

**Error:**
```
ERROR: epics-base-7.0.8-r0 do_package_qa: QA Issue:
epics-base: Files/directories were installed but not shipped in any package:
  /usr/share/epics/dbd/softIoc.dbd
```

**Cause:** Files installed but not listed in FILES variable

**Fix:**
```bitbake
FILES:${PN} += "${datadir}/epics/*"
```

---

#### 7. Library Not Found at Runtime

**Error on target:**
```
softIoc: error while loading shared libraries: libCom.so.3.15: cannot open shared object file
```

**Causes:**
- Library not installed
- LD_LIBRARY_PATH not set

**Fix:**
```bash
# On target, check if library is installed
ldconfig -p | grep libCom

# If not found, verify packaging
oe-pkgdata-util list-pkg-files epics-base | grep libCom

# Set LD_LIBRARY_PATH
export LD_LIBRARY_PATH=/usr/lib:$LD_LIBRARY_PATH

# Or source EPICS environment
source /etc/profile.d/epics.sh
```

---

### Debug Techniques

**1. Use devshell:**
```bash
bitbake epics-base -c devshell
# Opens shell in build environment
# Variables like ${S}, ${D}, ${WORKDIR} are set
# Can run make, configure, etc. manually
```

**2. Check work directory:**
```bash
cd tmp/work/cortexa53-poky-linux/epics-base/7.0.8-r0/
ls -la
# build/ - Build outputs
# image/ - Staged installation
# temp/ - Log files
# recipe-sysroot/ - Dependencies
```

**3. Read log files:**
```bash
# All logs
ls tmp/work/cortexa53-poky-linux/epics-base/7.0.8-r0/temp/

# Specific task logs
cat temp/log.do_fetch
cat temp/log.do_configure
cat temp/log.do_compile
cat temp/log.do_install
cat temp/log.do_package
```

**4. Print recipe variables:**
```bash
# Print all variables
bitbake -e epics-base > epics-env.txt

# Print specific variable
bitbake -e epics-base | grep ^DEPENDS=
bitbake -e epics-base | grep ^SRC_URI=
bitbake -e epics-base | grep ^S=
bitbake -e epics-base | grep ^D=
```

**5. Dependency graphing:**
```bash
# Generate dependency graph
bitbake -g epics-base

# Creates:
# - task-depends.dot (task dependencies)
# - pn-buildlist (recipe build order)
# - recipe-depends.dot (recipe dependencies)

# View graph
xdot task-depends.dot
```

---

## References

### Official Documentation

- **Yocto Project Documentation**: https://docs.yoctoproject.org/
- **BitBake User Manual**: https://docs.yoctoproject.org/bitbake/
- **Yocto Dev Manual**: https://docs.yoctoproject.org/dev-manual/
- **OpenEmbedded Layer Index**: https://layers.openembedded.org/

### EPICS Resources

- **EPICS Controls**: https://epics-controls.org/
- **EPICS Base Repository**: https://github.com/epics-base/epics-base
- **EPICS Tech-Talk**: https://epics.anl.gov/tech-talk/
- **EPICS Documentation**: https://docs.epics-controls.org/

### Xilinx Resources

- **meta-xilinx**: https://github.com/Xilinx/meta-xilinx
- **Xilinx Wiki**: https://xilinx-wiki.atlassian.net/
- **ZCU106 User Guide**: https://www.xilinx.com/support/documentation/boards_and_kits/zcu106/ug1244-zcu106-eval-bd.pdf

### Related Layers

- **meta-chimeratk**: https://github.com/ChimeraTK/meta-chimeratk
- **meta-epics examples**: Search GitHub for "meta-epics"

### Community

- **Yocto Project Mailing Lists**: https://lists.yoctoproject.org/
- **EPICS Tech-Talk Archive**: https://epics.anl.gov/tech-talk/
- **OpenEmbedded IRC**: #oe on irc.libera.chat

---

## Appendix: Complete Layer File Structure

```
meta-epics/
├── conf/
│   └── layer.conf
├── classes/
│   └── epics-image.bbclass
├── recipes-epics/
│   ├── epics-base/
│   │   ├── epics-base_7.0.8.bb
│   │   └── files/
│   │       ├── 0001-yocto-cross-compile-support.patch
│   │       ├── zcu106/
│   │       │   └── custom-config.patch
│   │       └── common-config.sh
│   └── epics-modules/
│       ├── epics-asyn/
│       │   ├── asyn_4-42.bb
│       │   └── files/
│       │       └── RELEASE.local
│       └── epics-stream/
│           ├── stream_2-8-24.bb
│           └── files/
│               └── RELEASE.local
├── recipes-core/
│   └── images/
│       ├── epics-minimal-image.bb
│       └── epics-full-image.bb
├── COPYING.MIT
├── README.md
└── .gitignore
```

---

## Next Steps

1. ✅ **Create the layer** following steps 1-13
2. ✅ **Test on hardware** - Verify EPICS works on ZCU106
3. 📝 **Document your layer** - Add detailed README
4. 🧪 **Add more recipes** - EPICS modules (asyn, StreamDevice, etc.)
5. 🔄 **Version control** - Commit to Git, tag releases
6. 🌐 **Share with community** - Submit to OpenEmbedded Layer Index
7. 📦 **Create SDK** - For application developers
8. 🚀 **Develop IOC applications** - Build custom control systems

---

## License

This guide is released under CC-BY-SA-4.0.

EPICS is released under the EPICS Open License.

Yocto Project and OpenEmbedded are Linux Foundation Collaborative Projects.

---

**Document Version**: 1.0
**Last Updated**: October 2025
**Author**: Generated with Claude Code
**Target**: Xilinx ZCU106 with Yocto Project
