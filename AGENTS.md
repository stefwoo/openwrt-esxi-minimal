# OpenWrt ESXi Minimal - Agent Guidelines

This repository contains configuration for building a minimal OpenWrt 24.10 firmware image optimized for ESXi deployment.

## Build Commands

Build the OpenWrt firmware image:
```bash
# Update and install feeds
./scripts/feeds update -a
./scripts/feeds install -a

# Configure with seed file
make defconfig

# Download dependencies
make download -j8 || make download -j1 V=s

# Build firmware (parallel, fallback to serial)
make -j$(nproc) || make -j1 V=s
```

Generate VMDK for ESXi:
```bash
# Uncompress the built image
cd bin/targets/x86/64/
gzip -d *-ext4-combined-efi.img.gz

# Convert to VMDK format
qemu-img convert -f raw -O vmdk *-ext4-combined-efi.img openwrt-esxi-flat.vmdk

# Create VMDK descriptor file with appropriate geometry
```

**Note**: This project does not have automated tests. Manual testing is performed by deploying the built firmware to an ESXi instance.

## Code Style Guidelines

### Shell Scripts (files/uci-defaults/*)
- Shebang: `#!/bin/sh` (POSIX shell compatibility)
- Comments: Use Chinese comments, prefix with `# ` and space
- Exit codes: Always end scripts with `exit 0` for success
- UCI commands: Use `uci set`, `uci delete`, and `uci commit` pattern
  - Delete with fallback: `uci delete network.lan.ip6assign 2>/dev/null || true`
- Configuration modifications:
  - Set values: `uci set <config>.<section>.<option>='<value>'`
  - Commit: `uci commit <config>`
- Network IP addresses: Use CIDR notation when applicable, but UCI uses separate netmask fields

### OpenWrt Config (config.seed)
- Section headers: Use `# ===` markers for logical grouping
  ```
  # ===================
  # Section Name
  # ===================
  ```
- Package format: `CONFIG_PACKAGE_<name>=y` for inclusion, `=n` for exclusion
- Organization: Group related packages (Base System, Network Tools, Kernel Modules)
- Naming: Follow OpenWrt package naming conventions
- Kernel modules: Prefix with `CONFIG_PACKAGE_kmod-`
- LuCI packages: Prefix with `CONFIG_PACKAGE_luci` or `CONFIG_LUCI_LANG_`

### File Structure
- `config.seed`: OpenWrt package selection and system configuration
- `files/99-firstboot`: First-boot initialization (language, ttyd web SSH)
- `files/99-custom-network`: Network configuration (bypass router mode, static IP)
- Files in `files/etc/uci-defaults/` are executed automatically on first boot
- All uci-defaults scripts must be executable: `chmod +x files/etc/uci-defaults/*`

### Configuration Patterns

**Network Setup** (files/99-custom-network):
- Static IP configuration with gateway and DNS
- Disable IPv6 globally
- Disable DHCP server
- Firewall in pass-through mode (ACCEPT all)

**First Boot** (files/99-firstboot):
- Set LuCI interface language
- Configure web SSH (ttyd) on LAN interface
- Enable services with init.d

**System Tuning**:
- IP forwarding: Append to `/etc/sysctl.conf`
- Kernel parameters: Use echo with append operator `>>`

## Important Notes

- This is a **minimal** OpenWrt build - disable unnecessary packages
- IPv6 is disabled globally in config.seed and network configuration
- Network configuration assumes bypass router mode (192.168.1.9, gateway 192.168.1.8)
- Chinese language support is included in LuCI interface
- VMDK creation requires `qemu-utils` package
- ESXi-specific kernel modules: e1000, e1000e, vmxnet3

## GitHub Actions Workflow

The `.github/workflows/build-openwrt.yml` builds the firmware on:
- Ubuntu 22.04 runner
- User-specified OpenWrt version (default: v24.10.5)
- Clones from multiple mirrors (GitHub, official, Gitee) for reliability
- Outputs VMDK artifacts for ESXi deployment
