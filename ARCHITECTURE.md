# spksrc Architecture Documentation

## Table of Contents
1. [Overview](#overview)
2. [Repository Structure](#repository-structure)
3. [Build System Architecture](#build-system-architecture)
4. [How Packages Are Built](#how-packages-are-built)
5. [Package Configuration and Settings](#package-configuration-and-settings)
6. [Toolchain System](#toolchain-system)
7. [Makefile Framework](#makefile-framework)
8. [Service Integration](#service-integration)
9. [Creating a New Package](#creating-a-new-package)
10. [Advanced Topics](#advanced-topics)

---

## Overview

spksrc is a cross-compilation framework designed to compile and package software for Synology NAS devices. It provides a comprehensive build system that handles:

- Cross-compilation for multiple Synology architectures
- Dependency management
- Package creation in SPK format
- Service integration with DSM (DiskStation Manager)
- Installation wizards and UI configuration

### Key Features

- **Multi-Architecture Support**: Builds packages for various Synology CPU architectures (ARM, x86, PowerPC)
- **Toolchain Management**: Automated toolchain setup for different DSM versions
- **Dependency Resolution**: Automatic handling of package dependencies
- **Standardized Build Process**: Consistent Makefile-based build system
- **Service Integration**: Built-in support for DSM services, users, and permissions

---

## Repository Structure

The repository is organized into several key directories:

### Directory Layout

```
spksrc/
├── cross/          # Cross-compilation package definitions
├── diyspk/         # DIY SPK packages
├── kernel/         # Kernel modules for specific architectures
├── mk/             # Makefile framework (core build logic)
├── native/         # Native tools built for build environment
├── python/         # Python-specific modules
├── spk/            # SPK package definitions (final packages)
├── toolchain/      # Cross-compilation toolchains
├── toolkit/        # DSM toolkit files
├── distrib/        # Downloaded source archives (created during build)
└── packages/       # Built SPK files (created during build)
```

### Directory Descriptions

#### `cross/`
Contains Makefiles for cross-compiling individual software components. Each subdirectory represents a library or application that can be built for target architectures.

**Example**: `cross/aria2/` contains the build definition for the aria2 download utility.

**Key files**:
- `Makefile` - Build instructions
- `digests` - SHA checksums for source archives
- `PLIST` - List of files installed (optional)

#### `spk/`
Contains complete package definitions for SPK files. Each package combines cross-compiled binaries with metadata, scripts, and configuration.

**Example**: `spk/aria2/` contains the complete package for aria2.

**Key files**:
- `Makefile` - SPK package configuration
- `src/` - Package-specific files (icons, scripts, configs, wizards)
- `src/service-setup.sh` - Service initialization script
- `src/wizard/` - Installation wizard UI definitions

#### `mk/`
The Makefile framework containing reusable build rules and logic. These files are included by packages to provide standardized build functionality.

**Key files**:
- `spksrc.common.mk` - Common variables and settings
- `spksrc.cross-cc.mk` - Cross-compilation rules
- `spksrc.spk.mk` - SPK package creation rules
- `spksrc.service.mk` - Service integration rules
- `spksrc.download.mk` - Source download rules
- `spksrc.install.mk` - Installation rules

#### `toolchain/`
Contains toolchains for cross-compiling to different Synology architectures and DSM versions.

**Naming convention**: `syno-{ARCH}-{DSM_VERSION}/`
- Example: `syno-avoton-7.1/` (x86_64 architecture, DSM 7.1)

#### `native/`
Tools that need to be built natively on the build system (not cross-compiled). These are typically build dependencies.

#### `toolkit/`
DSM toolkit files and libraries specific to different DSM versions.

#### `kernel/`
Kernel headers and modules for specific architectures, used when building kernel modules.

---

## Build System Architecture

The spksrc build system uses GNU Make with a hierarchical structure:

### Build Flow

```
User runs make
    ↓
Root Makefile (spksrc/Makefile)
    ↓
SPK Makefile (spk/{package}/Makefile)
    ↓
Framework Makefiles (mk/spksrc.*.mk)
    ↓
Cross-compilation Makefiles (cross/{dependency}/Makefile)
    ↓
Toolchain Setup
    ↓
Download → Extract → Patch → Configure → Compile → Install → Package
```

### Build Phases

1. **Pre-check**: Validate build environment and parameters
2. **Download**: Fetch source archives from upstream
3. **Checksum**: Verify downloaded files
4. **Extract**: Unpack source archives
5. **Patch**: Apply any necessary patches
6. **Configure**: Run configure scripts with cross-compilation settings
7. **Compile**: Build the software
8. **Install**: Install to staging directory
9. **Strip**: Remove debugging symbols to reduce size
10. **Package**: Create the final SPK file

### Work Directories

During the build process, several directories are created:

- `work-{ARCH}-{TCVERSION}/` - Build workspace for a specific architecture
- `work-{ARCH}-{TCVERSION}/install/` - Installed files
- `work-{ARCH}-{TCVERSION}/staging/` - Staging area for packaging

---

## How Packages Are Built

### Step-by-Step Package Build Process

#### 1. Source Package (cross/)

A source package in `cross/` defines how to build a specific piece of software.

**Example: cross/aria2/Makefile**
```makefile
PKG_NAME = aria2
PKG_VERS = 1.37.0
PKG_EXT = tar.xz
PKG_DIST_NAME = $(PKG_NAME)-$(PKG_VERS).$(PKG_EXT)
PKG_DIST_SITE = https://github.com/aria2/aria2/releases/download/release-$(PKG_VERS)
PKG_DIR = $(PKG_NAME)-$(PKG_VERS)

DEPENDS = cross/openssl3 cross/libssh2 cross/libexpat cross/c-ares cross/sqlite

HOMEPAGE = https://aria2.github.io/
COMMENT  = aria2 download utility
LICENSE  = GPLv2

GNU_CONFIGURE = 1
CONFIGURE_ARGS = --with-ca-bundle=/etc/ssl/certs/ca-certificates.crt

include ../../mk/spksrc.cross-cc.mk
```

**Key variables**:
- `PKG_NAME`, `PKG_VERS` - Package name and version
- `PKG_DIST_SITE` - Download URL
- `DEPENDS` - Build dependencies
- `GNU_CONFIGURE = 1` - Uses GNU autotools
- `CONFIGURE_ARGS` - Custom configure options

#### 2. Building Dependencies

Dependencies are built recursively before the main package:

```
aria2 depends on:
  → openssl3
  → libssh2 (which depends on openssl3)
  → libexpat
  → c-ares
  → sqlite
```

Each dependency is built for the target architecture and installed in the work directory.

#### 3. Cross-Compilation Environment

The framework sets up cross-compilation variables:
- `CC` - Cross-compiler (e.g., `arm-linux-gcc`)
- `CXX` - Cross C++ compiler
- `CFLAGS`, `LDFLAGS` - Compiler and linker flags
- `SYSROOT` - Root directory for target libraries

#### 4. SPK Package Creation

The SPK package combines the built binaries with metadata and scripts.

**Example: spk/aria2/Makefile**
```makefile
SPK_NAME = aria2
SPK_VERS = 1.37.0
SPK_REV = 1
SPK_ICON = src/aria2.png

DEPENDS = cross/aria2

MAINTAINER = cnrat
DESCRIPTION = aria2 is a lightweight multi-protocol download utility
DISPLAY_NAME = Aria2

WIZARDS_DIR = src/wizard
STARTABLE = yes
SERVICE_SETUP = src/service-setup.sh
SERVICE_USER = auto
SERVICE_PORT = 6800

include ../../mk/spksrc.spk.mk
```

**Key variables**:
- `SPK_NAME`, `SPK_VERS`, `SPK_REV` - Package identification
- `DEPENDS` - Cross-compiled dependencies
- `MAINTAINER`, `DESCRIPTION` - Package metadata
- `STARTABLE` - Whether the package is a service
- `SERVICE_*` - Service configuration

#### 5. Package Structure

The final SPK file contains:

```
package.spk
├── INFO                    # Package metadata (JSON)
├── PACKAGE_ICON.PNG       # Package icon
├── PACKAGE_ICON_256.PNG   # High-res icon
├── scripts/
│   ├── installer          # Installation script
│   ├── start-stop-status  # Service control script
│   └── service-setup      # Service initialization
├── conf/
│   ├── privilege          # User/permission settings
│   └── resource           # Resource configuration (DSM 7)
├── app/
│   └── config             # DSM UI configuration
└── package.tgz            # Actual package files
    └── (installed to /var/packages/{package}/target/)
```

#### 6. Installation on Synology

When installed on a Synology NAS:
1. SPK is extracted to temporary location
2. `scripts/installer` runs (preinst, postinst)
3. `package.tgz` is extracted to `/var/packages/{package}/target/`
4. Service user is created (if `SERVICE_USER = auto`)
5. Service is configured and can be started

---

## Package Configuration and Settings

### Installation Wizard

Packages can include installation wizards to collect configuration from users.

**Wizard file format** (`src/wizard/install_uifile`):
```json
[
    {
        "step_title": "Configuration",
        "items": [
            {
                "type": "textfield",
                "subitems": [
                    {
                        "key": "wizard_download_share",
                        "desc": "Shared folder for downloads",
                        "defaultValue": "downloads",
                        "validator": {
                            "allowBlank": false
                        }
                    }
                ]
            }
        ]
    }
]
```

**Wizard field types**:
- `textfield` - Text input
- `password` - Password input
- `combobox` - Dropdown selection
- `multiselect` - Multiple selection

**Using wizard values** (in `service-setup.sh`):
```bash
# Wizard values are available as environment variables
sed -e "s|%dir%|${wizard_download_share}|g" -i ${CFG_FILE}
```

### Service Setup Script

The `service-setup.sh` script configures the service at installation and runtime.

**Example structure**:
```bash
# Define configuration files
CFG_FILE="${SYNOPKG_PKGVAR}/config.conf"

# Define service command
SERVICE_COMMAND="${SYNOPKG_PKGDEST}/bin/myapp --config=${CFG_FILE}"
SVC_BACKGROUND=y
SVC_WRITE_PID=y

# Post-installation hook
service_postinst ()
{
    if [ "${SYNOPKG_PKG_STATUS}" == "INSTALL" ]; then
        # First-time installation
        # Apply wizard settings to configuration
        sed -e "s|%value%|${wizard_value}|g" -i ${CFG_FILE}
    fi
}

# Pre-upgrade hook
service_preupgrade ()
{
    # Backup configuration before upgrade
}

# Post-upgrade hook
service_postupgrade ()
{
    # Restore and migrate configuration
}
```

**Available variables**:
- `SYNOPKG_PKGNAME` - Package name
- `SYNOPKG_PKGVER` - Package version
- `SYNOPKG_PKGDEST` - Installation directory (`/var/packages/{package}/target`)
- `SYNOPKG_PKGVAR` - Variable data directory (`/var/packages/{package}/var`)
- `SYNOPKG_PKG_STATUS` - INSTALL, UPGRADE, or REPAIR
- `wizard_*` - Values from installation wizard

### Service Configuration Variables

In the SPK Makefile:

```makefile
# Service settings
STARTABLE = yes                          # Package creates a service
SERVICE_USER = auto                      # Create service user sc-{package}
SERVICE_SETUP = src/service-setup.sh     # Service setup script
SERVICE_COMMAND = ...                    # Direct service command (alternative)

# Port configuration
SERVICE_PORT = 8080                      # Service port
SERVICE_PORT_PROTOCOL = http             # http or https
FWPORTS = src/{package}.sc              # Custom firewall config

# UI integration
NO_SERVICE_SHORTCUT = yes               # Don't create app icon
DSM_UI_CONF = src/app/config            # Custom UI config

# Wizard
WIZARDS_DIR = src/wizard                # Wizard UI files
SERVICE_WIZARD_SHARENAME = wizard_share  # Wizard variable for share selection

# Advanced
USE_ALTERNATE_TMPDIR = 1                # Use package-specific temp directory
SPK_COMMANDS = bin/mytool               # Create symlinks in /usr/local/bin/
```

### User and Permissions

For DSM 7, packages use the privilege system:

**Generated `conf/privilege`** (when `SERVICE_USER = auto`):
```json
{
    "defaults": {
        "run-as": "package"
    },
    "username": "sc-{package}"
}
```

This creates a service user `sc-{package}` with limited permissions.

### Resource Configuration (DSM 7)

**Generated `conf/resource`**:
```json
{
    "data-share": {
        "shares": [{
            "name": "{share_name}",
            "permission": {
                "{service_user}": "RW"
            }
        }]
    }
}
```

---

## Toolchain System

### Understanding Toolchains

A toolchain is a collection of tools for cross-compiling:
- Compiler (gcc, g++)
- Assembler (as)
- Linker (ld)
- Binary utilities (ar, ranlib, strip, etc.)
- Standard libraries (libc, libstdc++, etc.)

### Toolchain Organization

Toolchains are organized by architecture and DSM version:

```
toolchain/
├── syno-88f6281-6.1/       # Marvell Kirkwood (ARMv5), DSM 6.1
├── syno-avoton-6.1/        # Intel Avoton (x86_64), DSM 6.1
├── syno-avoton-7.1/        # Intel Avoton (x86_64), DSM 7.1
├── syno-armada37xx-7.1/    # Marvell Armada (ARMv8), DSM 7.1
└── ...
```

### Architecture Types

**Specific architectures**:
- `88f6281` - Marvell Kirkwood (ARMv5)
- `avoton` - Intel Avoton (x86_64)
- `armada37xx` - Marvell Armada 37xx (ARMv8)
- `rtd1296` - Realtek RTD1296 (ARMv8)
- Many more...

**Generic architectures** (groups of compatible specific architectures):
- `x64` - 64-bit x86 architectures
- `armv7` - ARMv7 architectures
- `armv8` - ARMv8 architectures

### Selecting Architecture

When building a package:

```bash
# Build for specific architecture
cd spk/aria2
make ARCH=avoton-7.1

# Build for generic architecture (builds for all compatible archs)
make ARCH=x64

# Build for all supported architectures
make all-supported
```

### Toolchain Version (TCVERSION)

The toolchain version corresponds to DSM versions:
- `6.1` - DSM 6.1
- `6.2` - DSM 6.2
- `7.0` - DSM 7.0
- `7.1` - DSM 7.1

### Architecture Support

Packages can specify unsupported architectures:

```makefile
# Exclude old architectures
UNSUPPORTED_ARCHS = $(ARMv5_ARCHS) $(OLD_PPC_ARCHS)

# Conditional exclusion based on compiler version
include ../../mk/spksrc.common.mk
ifeq ($(call version_lt,${TCVERSION},6.0),1)
UNSUPPORTED_ARCHS += $(ARCH)
endif
```

---

## Makefile Framework

The `mk/` directory contains the core build logic organized into modular Makefiles.

### Core Framework Files

#### `spksrc.common.mk`
Common variables, functions, and settings used by all makefiles.

**Key contents**:
- `BASEDIR` - Repository root
- `ENV_VARS_TO_CLEAN` - Environment cleanup for reproducible builds
- `RUN` - Command wrapper for build environment
- Helper functions (version comparison, etc.)

#### `spksrc.directories.mk`
Directory structure definitions.

**Key variables**:
- `WORK_DIR` - Build workspace
- `INSTALL_DIR` - Installation directory
- `STAGING_DIR` - Staging directory for packaging
- `INSTALL_PREFIX` - Target installation path
- `DISTRIB_DIR` - Downloaded sources
- `PACKAGES_DIR` - Built SPK files

#### `spksrc.download.mk`
Handles source code downloading.

**Supports**:
- HTTP/HTTPS URLs
- Git repositories
- SVN repositories
- Local files

**Checksum verification** using `digests` file.

#### `spksrc.cross-cc.mk`
Main cross-compilation rules.

**Includes**:
1. Pre-check
2. Cross-compilation environment setup
3. Download
4. Dependency resolution
5. Checksum verification
6. Extract
7. Patch
8. Configure
9. Compile
10. Install

#### `spksrc.spk.mk`
SPK package creation rules.

**Responsibilities**:
- Combine cross-compiled binaries
- Generate package metadata (INFO file)
- Create scripts (installer, start-stop-status)
- Package icons
- Create final SPK file

#### `spksrc.service.mk`
Service integration.

**Generates**:
- Service scripts
- Privilege configuration
- Firewall configuration
- DSM UI integration
- Resource configuration (DSM 7)

### Build Configuration Options

#### Configure Methods

```makefile
# GNU Autotools (./configure)
GNU_CONFIGURE = 1
CONFIGURE_ARGS = --enable-feature --with-library

# CMake
CMAKE_USE_NINJABUILD = 1
CMAKE_ARGS = -DENABLE_FEATURE=ON

# Meson
MESON_USE_NINJABUILD = 1
CONFIGURE_ARGS = -Dfeature=enabled

# Custom configure
CONFIGURE_TARGET = mypackage_configure
```

#### Compile Options

```makefile
# Parallel compilation
COMPILE_MAKE_OPTIONS = -j$(shell nproc)

# Custom compile target
COMPILE_TARGET = mypackage_compile
```

#### Install Options

```makefile
# Custom install command
INSTALL_MAKE_OPTIONS = install-strip DESTDIR=$(INSTALL_DIR)

# Post-install customization
POST_INSTALL_TARGET = mypackage_post_install
```

### Hooks and Customization

Most build phases support pre/post hooks:

```makefile
# Pre-patch hook
PRE_PATCH_TARGET = mypackage_pre_patch

.PHONY: mypackage_pre_patch
mypackage_pre_patch:
    # Custom commands before patching
    @echo "Preparing for patch..."

# Post-install hook
POST_INSTALL_TARGET = mypackage_post_install

.PHONY: mypackage_post_install
mypackage_post_install:
    # Custom commands after installation
    install -m 644 src/config.conf $(STAGING_DIR)/var/
```

---

## Service Integration

### Service Types

#### Startable Services

Services that run continuously (daemons):

```makefile
STARTABLE = yes
SERVICE_SETUP = src/service-setup.sh
SERVICE_USER = auto
SERVICE_COMMAND = "${SYNOPKG_PKGDEST}/bin/daemon --config=${CFG_FILE}"
SVC_BACKGROUND = y
SVC_WRITE_PID = y
```

#### Non-startable Packages

Command-line tools without a service:

```makefile
STARTABLE = no
```

### Service Scripts

#### Generic Service (Recommended)

Use `SERVICE_SETUP` with a service-setup script:

```bash
#!/bin/bash

# Configuration
CFG_FILE="${SYNOPKG_PKGVAR}/config.conf"
SERVICE_COMMAND="${SYNOPKG_PKGDEST}/bin/myapp --config=${CFG_FILE}"

# Service options
SVC_BACKGROUND=y        # Run in background
SVC_WRITE_PID=y         # Write PID file

# Hooks
service_postinst() {
    # After installation
}

service_preuninst() {
    # Before uninstallation
}

service_postuninst() {
    # After uninstallation
}

service_preupgrade() {
    # Before upgrade
}

service_postupgrade() {
    # After upgrade
}
```

#### Custom Service Script

For complex service management:

```makefile
SSS_SCRIPT = src/start-stop-status.sh
```

### Service User Management

#### Automatic Service User (DSM 7)

```makefile
SERVICE_USER = auto
```

Creates user `sc-{package}` with restricted permissions.

#### Group Configuration

```makefile
SPK_GROUP = mygroup              # Primary group
SYSTEM_GROUP = users             # Additional group membership
```

### Port and Firewall Configuration

#### Service Port

```makefile
SERVICE_PORT = 8080
SERVICE_PORT_PROTOCOL = http     # or https
```

#### Custom Firewall Configuration

**File: `src/{package}.sc`**
```json
{
    "{package}": {
        "title": "Package Name",
        "desc": "Package Description",
        "port_forward": "yes",
        "dst": [{
            "type": "tcp",
            "port": "8080"
        }]
    }
}
```

### DSM UI Integration

#### Application Icon

Automatically created when `SERVICE_PORT` is defined:

```makefile
SERVICE_PORT = 8080
# Creates app icon that opens http://{nas}:8080
```

Disable with:
```makefile
NO_SERVICE_SHORTCUT = yes
```

#### Custom UI Configuration

**File: `src/app/config`**
```json
{
    ".url": {
        "com.synocommunity.{package}": {
            "title": "Package Name",
            "desc": "Package Description",
            "icon": "app/{package}.png",
            "type": "url",
            "protocol": "http",
            "port": "8080",
            "url": "/",
            "allUsers": true
        }
    }
}
```

---

## Creating a New Package

### Step 1: Create Cross-Compilation Package

Create `cross/{package}/Makefile`:

```makefile
PKG_NAME = myapp
PKG_VERS = 1.0.0
PKG_EXT = tar.gz
PKG_DIST_NAME = $(PKG_NAME)-$(PKG_VERS).$(PKG_EXT)
PKG_DIST_SITE = https://example.com/releases
PKG_DIR = $(PKG_NAME)-$(PKG_VERS)

# Dependencies
DEPENDS = cross/openssl cross/zlib

# Metadata
HOMEPAGE = https://example.com
COMMENT  = My application
LICENSE  = MIT

# Build configuration
GNU_CONFIGURE = 1
CONFIGURE_ARGS = --enable-ssl

# Include cross-compilation framework
include ../../mk/spksrc.cross-cc.mk
```

### Step 2: Generate Checksums

```bash
cd cross/myapp
make digests
```

This creates `digests` file with SHA256/SHA512 checksums.

### Step 3: Test Cross-Compilation

```bash
cd cross/myapp
make ARCH=avoton-7.1
```

### Step 4: Create SPK Package

Create `spk/{package}/Makefile`:

```makefile
SPK_NAME = myapp
SPK_VERS = 1.0.0
SPK_REV = 1
SPK_ICON = src/myapp.png

# Dependencies
DEPENDS = cross/myapp

# Metadata
MAINTAINER = yourusername
DESCRIPTION = My application for Synology
DISPLAY_NAME = My App
LICENSE = MIT
HOMEPAGE = https://example.com

# Service configuration
STARTABLE = yes
SERVICE_SETUP = src/service-setup.sh
SERVICE_USER = auto
SERVICE_PORT = 8080

# Wizard
WIZARDS_DIR = src/wizard

# Include SPK framework
include ../../mk/spksrc.spk.mk
```

### Step 5: Create Package Resources

#### Icon (`spk/myapp/src/myapp.png`)
72x72 PNG image

#### Service Setup (`spk/myapp/src/service-setup.sh`)
```bash
#!/bin/bash

CFG_FILE="${SYNOPKG_PKGVAR}/config.conf"
SERVICE_COMMAND="${SYNOPKG_PKGDEST}/bin/myapp --config=${CFG_FILE}"
SVC_BACKGROUND=y
SVC_WRITE_PID=y

service_postinst ()
{
    if [ "${SYNOPKG_PKG_STATUS}" == "INSTALL" ]; then
        # First installation
        cp ${SYNOPKG_PKGDEST}/var/config.conf.template ${CFG_FILE}
        
        # Apply wizard settings
        sed -e "s|%port%|${wizard_port}|g" -i ${CFG_FILE}
    fi
}
```

#### Installation Wizard (`spk/myapp/src/wizard/install_uifile`)
```json
[
    {
        "step_title": "Configuration",
        "items": [
            {
                "type": "textfield",
                "subitems": [
                    {
                        "key": "wizard_port",
                        "desc": "Service port",
                        "defaultValue": "8080",
                        "validator": {
                            "allowBlank": false,
                            "regex": {
                                "expr": "/^[0-9]+$/",
                                "errorText": "Port must be a number"
                            }
                        }
                    }
                ]
            }
        ]
    }
]
```

### Step 6: Build SPK Package

```bash
cd spk/myapp
make ARCH=avoton-7.1
```

The SPK file will be created in `packages/myapp_avoton-7.1_{version}.spk`

### Step 7: Test Installation

1. Copy SPK to your Synology NAS
2. Install via Package Center → Manual Install
3. Verify service starts correctly
4. Check logs in `/var/log/` or package-specific log location

---

## Advanced Topics

### Python Packages

For Python applications:

```makefile
# SPK Makefile
WHEELS = src/requirements.txt
PYTHON_PACKAGE = python311

# Use Python cross-compilation
include ../../mk/spksrc.python.mk
```

**Requirements file** (`src/requirements.txt`):
```
Flask==2.0.1
requests==2.28.0
```

### Kernel Modules

For packages requiring kernel modules:

```makefile
DEPENDS = cross/myapp
KERNEL_DEPENDS = kernel/mymodule

include ../../mk/spksrc.kernel-required.mk
```

### Multi-Architecture Generic Packages

Build for all compatible architectures:

```makefile
# In SPK Makefile
ARCH = x64  # Generic x86_64 architecture

# Remove specific archs if needed
UNSUPPORTED_ARCHS = legacy-arch
```

### Custom Patches

Create `cross/{package}/patches/`:

```bash
cross/myapp/patches/
├── 001-fix-compiler-warning.patch
└── 002-enable-feature.patch
```

Patches are applied automatically in order.

### Environment Variables

Access in service scripts:

```bash
# Package information
SYNOPKG_PKGNAME="myapp"
SYNOPKG_PKGVER="1.0.0"
SYNOPKG_PKGDEST="/var/packages/myapp/target"
SYNOPKG_PKGVAR="/var/packages/myapp/var"

# Installation status
SYNOPKG_PKG_STATUS="INSTALL"  # or UPGRADE, REPAIR

# Shared folders (when using SERVICE_WIZARD_SHARENAME)
SHARE_PATH="/volume1/myshare"

# User/Group
SERVICE_USER="sc-myapp"
SERVICE_GROUP="sc-myapp"
```

### Resource Management (DSM 7)

#### Share Management

```makefile
SERVICE_WIZARD_SHARENAME = wizard_share_name
```

Automatically configures permissions in `conf/resource`.

#### Certificate Management

```makefile
SERVICE_CERT = myapp
SERVICE_CERT_RELOAD = scripts/reload-cert.sh
```

DSM will manage SSL certificates for the service.

### Debugging Build Issues

#### Enable verbose output

```bash
make PSTAT=  # Disable progress statistics
make V=1     # Verbose mode (if supported)
```

#### Check build logs

```bash
# Build log location
cat cross/myapp/work-{arch}-{tc}/build.log
```

#### Clean build

```bash
# Clean specific package
cd cross/myapp
make clean

# Clean everything
cd spksrc
make dist-clean
```

### Testing in Docker

```bash
# Pull build container
docker pull ghcr.io/synocommunity/spksrc

# Run container
docker run -it --platform=linux/amd64 \
  -v $(pwd):/spksrc \
  -w /spksrc \
  ghcr.io/synocommunity/spksrc /bin/bash

# Inside container, build package
cd spk/myapp
make ARCH=avoton-7.1
```

---

## Build System Flow Diagram

```
┌─────────────────────────────────────────┐
│  Developer runs: make ARCH=avoton-7.1   │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│  spk/{package}/Makefile                 │
│  - Defines SPK metadata                 │
│  - Lists dependencies (DEPENDS)         │
│  - Includes mk/spksrc.spk.mk            │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│  mk/spksrc.spk.mk                       │
│  - SPK package creation rules           │
│  - Calls dependency builds              │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│  cross/{dependency}/Makefile            │
│  - Download source                      │
│  - Extract and patch                    │
│  - Configure with toolchain             │
│  - Compile                              │
│  - Install to work-{arch}/install/      │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│  mk/spksrc.service.mk                   │
│  - Generate service scripts             │
│  - Create privilege config              │
│  - Setup UI integration                 │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│  Final SPK Package                      │
│  packages/{package}_{arch}_{ver}.spk    │
│  - INFO (metadata)                      │
│  - scripts/ (installer, service)        │
│  - conf/ (privileges, resources)        │
│  - package.tgz (binaries and files)     │
└─────────────────────────────────────────┘
```

---

## Summary

The spksrc framework provides a comprehensive, standardized way to build packages for Synology NAS devices. Key points:

1. **Organized structure**: Separate directories for cross-compiled libraries (`cross/`), final packages (`spk/`), and build framework (`mk/`)

2. **Dependency management**: Automatic resolution and building of dependencies

3. **Multi-architecture support**: Build for specific or generic architectures across DSM versions

4. **Service integration**: Built-in support for DSM services, users, permissions, and UI

5. **Wizard system**: User-friendly configuration during installation

6. **Modular Makefiles**: Reusable build logic in `mk/` directory

7. **Standardized workflow**: Download → Extract → Patch → Configure → Compile → Install → Package

For more information, see:
- [README.md](README.md) - Setup and quick start
- [CONTRIBUTING.md](CONTRIBUTING.md) - How to contribute
- [Developers HOW TO](https://github.com/SynoCommunity/spksrc/wiki/Developers-HOW-TO) - Detailed development guide
- [Package Documentation Index](https://github.com/SynoCommunity/spksrc/wiki/Package-Documentation-Index) - Package-specific docs
