# Samsung Galaxy A20s LK2ND Porting Guide

## Device Information
- **Model**: Samsung Galaxy A20s (SM-A207F, SM-A207G, etc.)
- **SoC**: Qualcomm SDM450 (ARMv8, 8-core: 4x Cortex-A53 + 4x Cortex-A53)
- **Base Bootloader**: Qualcomm & Samsung-specific boot chain
- **Linux Support**: Mainline Linux with Halium modifications

## Existing Support Status
✅ **Good News**: SDM450 is already fully supported in LK2ND via the msm8953 platform
✅ **Already Working Devices**: Galaxy A6+, Galaxy J8, Tab A2 XL (all SDM450-based)
⚠️  **A20s Status**: Device tree entry added, ready for testing

## Building LK2ND for A20s

### Requirements
- ARM cross-compiler: `arm-none-eabi-gcc` or `aarch64-none-eabi-gcc`
- Build tools: `make`, `dtc` (device tree compiler)
- Python 3.x
- Linux build environment

### Build Instructions

```bash
# Navigate to the lk2nd repository
cd /path/to/lk2nd-main

# Build LK2ND for MSM8953 (which supports SDM450)
make TOOLCHAIN_PREFIX=aarch64-none-eabi- lk2nd-msm8953

# Output will be: build-lk2nd-msm8953/lk2nd.img
```

### Create Boot Image

```bash
# If you want to create a flashable boot image:
make TOOLCHAIN_PREFIX=aarch64-none-eabi- lk2nd-msm8953 2>&1 | tail -20
# Look for the lk2nd.img output path

# Flash to device (requires fastboot or heimdall):
fastboot flash boot build-lk2nd-msm8953/lk2nd.img
# OR for Samsung devices:
heimdall flash --BOOT build-lk2nd-msm8953/lk2nd.img
```

## Device Tree Configuration

The A20s has been added to the SDM450 Samsung device tree with the following configuration:

**File**: `lk2nd/device/dts/msm8953/sdm450-samsung-r04.dts`

```devicetree
a20s-generic {
    model = "Samsung Galaxy A20s";
    compatible = "samsung,a20s";
    lk2nd,match-bootloader = "A207*";
    lk2nd,dtb-files = "sdm450-samsung-a20s";
};
```

### Bootloader ID Matching

The device is identified by the bootloader name pattern `A207*`. This matches:
- SM-A207F (Europe)
- SM-A207G (Europe)
- SM-A207FV (Regional variants)
- SM-A20700 (Other regions)

If your variant uses a different bootloader ID, you may need to modify the pattern.

## Troubleshooting

### Device Not Recognized

If the device shows "unknown bootloader" during fastboot detection:

1. **Check bootloader ID**:
   ```bash
   # Enter fastboot mode on device
   adb reboot bootloader
   
   # Query bootloader info
   fastboot getvar bootloader
   ```

2. **Add your variant** if it doesn't match "A207*":
   - Edit `lk2nd/device/dts/msm8953/sdm450-samsung-r04.dts`
   - Add a new entry with your bootloader ID pattern
   - Rebuild

### Device Won't Boot After Flash

1. **Verify flash was successful**:
   ```bash
   fastboot boot build-lk2nd-msm8953/lk2nd.img
   # Test boot mode first
   ```

2. **Check logs** (if device gets into LK2ND):
   ```bash
   fastboot oem log
   fastboot get_staged /tmp/lk2nd_log.txt
   cat /tmp/lk2nd_log.txt
   ```

3. **Common issues**:
   - Incompatible kernel or missing kernel DTB
   - USB connection issues
   - Device variant not properly matched

### Debugging

To enable maximum debugging:

1. Edit `lk2nd/project/lk2nd.mk`:
   - Ensure `DEBUG := 1` is set (already done)
   
2. Build with debug output:
   ```bash
   make TOOLCHAIN_PREFIX=aarch64-none-eabi- DEBUG=1 lk2nd-msm8953 V=1
   ```

3. Collect detailed logs after flash

## Device-Specific Workarounds

Based on similar Samsung SDM450 devices (Galaxy A6+, J8), the A20s may need:

### USB Considerations
- Samsung devices sometimes require USB workarounds
- Check `lk2nd/device/dts/msm8953/sdm450-samsung-r04.dts` for any MUIC settings

### Memory Configuration
- Typical SDM450 devices: 2GB or 3GB RAM
- LK2ND auto-detects, but verify if boot fails

### Display/UI
- A20s has 6.4" display (IPS LCD)
- LK2ND includes fastboot menu support
- Volume keys for navigation

## Next Steps After Successful Boot

1. **Test Fastboot Interface**:
   ```bash
   fastboot devices
   fastboot getvar all
   ```

2. **Flash Custom Kernel**:
   - Use the A20s kernel from the android_kernel_samsung_a20s-main/ directory
   - Create a boot image: `mkbootimg --kernel Image.gz-dtb --ramdisk ramdisk.img -o boot.img`
   - Flash: `fastboot flash boot boot.img`

3. **Install Mainline Linux** (if desired):
   - Prepare arm64 kernel with proper device tree
   - Boot via LK2ND's file system or partition offset features

## References

### LK2ND Documentation
- Main README: `README.md`
- Device documentation: `Documentation/devices.md`
- Building guide: `Documentation/building.md`

### Related Devices  (Working examples)
- **Galaxy A6+** (SM-A605F) - SDM450
- **Galaxy J8** (SM-J810F) - SDM450
- **Tab A2 XL** (SM-T595) - SDM450

### Hardware Info
- **SoC**: Qualcomm SDM450
- **RAM**: 3-4GB
- **Storage**: 32-64GB eMMC
- **Display**: 6.4" 1560×720 IPS LCD

## Contributing Back

If you successfully port to your A20s variant:

1. **Document any bootloader IDs** that are different
2. **Note any device-specific issues** found and fixed
3. **Consider submitting patches** to the mainline LK2ND project
4. **Share kernel DTB files** for other variants

## Support & Community

- **LK2ND Matrix**: `#lk2nd:postmarketos.org`
- **Issue Tracker**: https://github.com/msm8916-mainline/lk2nd/issues
- **Related Projects**: PostmarketOS, Halium, Mainline Linux on Mobile

---

**Last Updated**: 2026-04-19
**Status**: Device tree entry added, ready for testing
