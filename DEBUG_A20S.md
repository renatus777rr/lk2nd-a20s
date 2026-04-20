# Samsung Galaxy A20s LK2ND - Detailed Debugging & Testing Guide

## Pre-Flight Checks

### 1. Device Preparation

**Back up your data**
```bash
# Your device data will be erased/modified during flashing
adb backup -apk -shared -all -f backup.ab
```

**Check current bootloader**
```bash
# Connect device to computer via USB
adb devices  # Verify device is recognized

# Enter fastboot mode (USB debugging must be enabled)
adb reboot bootloader
# OR manually: Power off > Hold Vol Down + Power

# Check bootloader variant
fastboot getvar bootloader
# Should output something like: A207FXXU1ATJ1 or A20700FXXU1ATJ1

fastboot getvar variant
fastboot getvar product
```

### 2. Build Verification

Before flashing, verify LK2ND builds correctly:

```bash
cd /path/to/lk2nd-main

# Full clean rebuild
make clean BOOTLOADER_OUT=./

# Build for msm8953 (SDM450)
make TOOLCHAIN_PREFIX=aarch64-none-eabi- \
     BOOTLOADER_OUT=./ \
     lk2nd-msm8953 -v 2>&1 | tee build.log

# Check output
ls -la build-lk2nd-msm8953/lk2nd.img
file build-lk2nd-msm8953/lk2nd.img

# Should be a valid boot image
```

## Identifying the Correct Bootloader ID

### Issue: Device Shows "Unknown Device" in LK2ND

The bootloader ID pattern matching is critical. Samsung uses a specific format.

**Find your exact bootloader version:**
```bash
# In fastboot mode:
fastboot getvar bootloader

# Expected patterns for A20s:
# A207FXXU* (European)
# A207GXXU* (European)
# A20700XX* (Some regions)
```

**If your ID doesn't match "A207*":**

1. Find the actual prefix:
   ```bash
   fastboot getvar bootloader | grep -o '^[A-Z0-9]*'
   ```

2. Edit device tree to add your variant:
   ```bash
   # Edit: lk2nd/device/dts/msm8953/sdm450-samsung-r04.dts
   ```

3. Add a new entry:
   ```devicetree
   a20s-<your_region> {
       model = "Samsung Galaxy A20s (<your_region>)";
       compatible = "samsung,a20s";
       lk2nd,match-bootloader = "A20F*";  # or A20G*, A207*, etc.
       lk2nd,dtb-files = "sdm450-pmi632";
   };
   ```

4. Rebuild: `make lk2nd-msm8953`

## Step-by-Step Testing

### Stage 1: Test Boot (Non-Destructive)

```bash
# Enter fastboot mode on device
adb reboot bootloader

# Boot LK2ND without flashing (temporary test)
cd /path/to/lk2nd-main/build-lk2nd-msm8953
fastboot boot lk2nd.img

# What to observe:
# - Device should show LK2ND fastboot menu
# - Volume keys should navigate menu
# - Should see "Samsung Galaxy A20s" or "Unknown Device"
```

### Stage 2: Verify Device Recognition

If device boots into LK2ND fastboot:

```bash
# With device showing LK2ND fastboot menu:
fastboot getvar all

# Should output bootloader info, including:
# bootloader: <version>
# product: <product_id>
# device: <device_name>
```

### Stage 3: Collect Logs

If device doesn't boot or you see issues:

```bash
# Boot into LK2ND (test mode)
fastboot boot lk2nd.img

# Wait for it to boot/fail

# Get logs from LK2ND (if it reaches fastboot)
fastboot oem log

# Extract to file
fastboot get_staged /tmp/lk2nd_log.txt
cat /tmp/lk2nd_log.txt
```

### Stage 4: Flash to Boot Partition

**Only proceed if Stage 1-3 are successful**

```bash
# BACKUP YOUR DEVICE FIRST!
adb backup -apk -shared -all -f backup_preflash.ab

# Enter fastboot
adb reboot bootloader

# Flash LK2ND to boot partition
fastboot flash boot build-lk2nd-msm8953/lk2nd.img

# Verify flash
fastboot getvar all | grep -i boot

# Boot device
fastboot reboot
```

### Stage 5: Test from Device Side

```bash
# Device should boot into LK2ND fastboot menu
# Wait for menu to appear (~5-10 seconds)

# Try these keys:
# - Volume Down: Next menu item / Enter fastboot mode
# - Volume Up: Previous menu item / Boot recovery

# Connect via USB while in LK2ND fastboot
adb devices  # Should NOT show device (fastboot is active)

# Use fastboot instead:
fastboot devices  # Should show device
fastboot getvar all
```

## Common Issues & Solutions

### Issue 1: Device Not Recognized by Fastboot

**Symptoms**: `fastboot devices` shows nothing

**Solutions**:
```bash
# 1. Check USB connection
lsusb  # Linux/Mac
# Should show "Qualcomm" device

# 2. Check fastboot drivers (Windows)
# Download Android SDK Platform Tools
# https://developer.android.com/tools/releases/platform-tools

# 3. Manually enter bootloader
adb reboot bootloader

# 4. Reload drivers
fastboot getvar bootloader  # Forces driver load
```

### Issue 2: Device Shows "Unknown Device" in LK2ND

**Symptoms**: LK2ND boots but says "Unknown Device" (might still work)

**Cause**: Bootloader ID doesn't match pattern

**Solutions**:
```bash
# 1. Check your actual bootloader ID
fastboot getvar bootloader

# 2. Update device tree if needed
# Edit: lk2nd/device/dts/msm8953/sdm450-samsung-r04.dts
# Change lk2nd,match-bootloader pattern

# 3. Rebuild and test:
make lk2nd-msm8953
fastboot boot build-lk2nd-msm8953/lk2nd.img
```

### Issue 3: Device Won't Boot into LK2ND

**Symptoms**: Device hangs/loops/blank screen after flash

**Debugging Steps**:

```bash
# 1. Go back to original bootloader
# Hold Vol Up + Power to enter recovery
# Use stock Samsung bootloader

# 2. Reflash original boot partition
adb reboot bootloader
fastboot flash boot original_boot.img  # From backup
fastboot reboot

# 3. Test with existing working device
# If it works with Galaxy A6+ DTB, issue is device-specific

# 4. Enable maximum logging
# Rebuild with DEBUG=1:
make DEBUG=1 lk2nd-msm8953
fastboot boot build-lk2nd-msm8953/lk2nd.img

# 5. Capture logs
fastboot oem log
fastboot get_staged boot_log.txt
```

### Issue 4: USB Not Working

**Symptoms**: Device recognized in LK2ND but USB shows errors

**Note**: Samsung devices sometimes need USB workarounds

**Workarounds**:
1. Try different USB port (preferably USB 2.0)
2. Use different USB cable
3. Check device tree for Samsung-specific workarounds

```devicetree
# Check if this exists in device tree:
muic-reset {
   compatible = "samsung,muic-reset";
}
```

### Issue 5: Kernel Won't Boot from LK2ND

**Symptoms**: LK2ND works but kernel fails to boot

**Causes**:
- Missing/incompatible kernel DTB
- Kernel command line issues
- Hardware initialization in kernel

**Testing**:
```bash
# 1. Create a minimal test image
# Use existing A20s kernel Image.gz
ls android_kernel_samsung_a20s-main/arch/arm64/boot/

# 2. Create boot image
cd android_kernel_samsung_a20s-main/arch/arm64/boot/

# 3. Get ramdisk from original boot
bootimg_tools --extract /path/to/original_boot.img

# 4. Repack with lfm2nd kernel
mkbootimg \
  --kernel Image.gz-dtb \
  --ramdisk ramdisk.img \
  --kernel_offset 0x80008000 \
  --ramdisk_offset 0x02000000 \
  --base 0x80000000 \
  --pagesize 4096 \
  -o test_boot.img

# 5. Flash and test
fastboot flash boot test_boot.img
```

## Advanced Debugging

### Enable Serial Console Output

If your device supports UART debugging (advanced):

```bash
# Compile LK2ND with UART output enabled
make ENABLE_UART_DM_LOGGING=1 lk2nd-msm8953

# Connect USB serial adapter to UART pins (if accessible)
# View bootloader output in real-time
screen /dev/ttyUSB0 115200
```

### Analyze Boot Logs

After collecting logs:

```bash
# Extract from fastboot
fastboot oem log
fastboot get_staged /tmp/lk2nd_log.txt

# Common log patterns to look for:
grep -i "error\|fail\|panic" /tmp/lk2nd_log.txt

# Check device detection
grep -i "samsung\|a20s\|dtb" /tmp/lk2nd_log.txt

# Check USB initialization
grep -i "usb" /tmp/lk2nd_log.txt
```

### Compare with Working Device

If you have another SDM450 Samsung device that works:

```bash
# Boot the working device with LK2ND
fastboot boot build-lk2nd-msm8953/lk2nd.img

# Get its logs
fastboot oem log
fastboot get_staged working_log.txt

# Compare logs
diff working_log.txt /tmp/lk2nd_log.txt

# Look for differences:
# - Memory detection
# - Clock initialization
# - Storage detection
```

## Recovery Procedures

### Reset to Stock

If device becomes unbootable:

```bash
# Use EDL (Emergency Download Mode) via EDL tool
# Or use Samsung Odin/Heimdall

# Via Heimdall (Linux/Mac):
heimdall print-pic  # Verify device detection
heimdall flash --BOOT original_boot.bin --resetFlashCounter

# Via Fastboot (if accessible):
fastboot flash boot original_boot.img
fastboot flash recovery original_recovery.img
fastboot reboot
```

### Boot from Recovery

If boot partition is corrupted:

```bash
# Should still be able to enter recovery
# Hold Vol Up + Power in bootloader

# From recovery, can reflash boot via:
# adb sideload boot.zip
```

## Success Indicators

✅ **Device Booting Successfully Into LK2ND:**
- Fastboot menu appears
- `fastboot devices` shows device
- `fastboot getvar all` returns info
- Device recognized as "Samsung Galaxy A20s"

✅ **Ready for Kernel Flashing:**
- LK2ND boots consistently
- Volume keys work in menu
- USB connectivity stable
- No error messages in log

## Next: Kernel Flashing

Once LK2ND is confirmed working:

```bash
# Flash the Android kernel (or other OS)
cd android_kernel_samsung_a20s-main/arch/arm64/boot

# Create boot image with kernel
mkbootimg \
  --kernel Image.gz-dtb \
  --ramdisk /path/to/ramdisk.img \
  -o a20s_boot.img

# Flash via LK2ND fastboot
fastboot flash boot a20s_boot.img

# Boot
fastboot reboot
```

## Support Resources

- **LK2ND Issues**: https://github.com/msm8916-mainline/lk2nd/issues
- **PostmarketOS**: https://postmarketos.org/
- **Halium**: https://halium.org/
- **Matrix Chat**: `#lk2nd:postmarketos.org`

---

**Last Updated**: 2026-04-19
**Status**: Testing guide for SDM450 A20s support
