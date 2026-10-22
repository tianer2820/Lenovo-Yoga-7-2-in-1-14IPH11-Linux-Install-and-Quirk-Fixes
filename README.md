# Lenovo Yoga 7 2-in-1 (14IPH11) Linux Guide

Comprehensive Linux installation guide, hardware compatibility report, and quirk fixes for the **Lenovo Yoga 7 2-in-1 14IPH11** (Machine Type: `83TC`, Intel Core Ultra Series 3 / Panther Lake `PTL`).

---

## Table of Contents
- [Hardware Compatibility](#hardware-compatibility)
- [Installation Guide](#installation-guide)
  - [Tested Environment](#tested-environment)
  - [BIOS Configuration (Secure Boot & 3rd Party Certificates)](#bios-configuration-secure-boot--3rd-party-certificates)
  - [BitLocker Dual-Boot Note](#bitlocker-dual-boot-note)
- [Quirk Fixes](#quirk-fixes)
  - [1. Audio Setup & Quad-Speaker (Woofers/Bass) Fix](#1-audio-setup--quad-speaker-woofersbass-fix)
  - [2. Auto-Rotation & 2-in-1 Sensor Hub (Intel ISH Fix)](#2-auto-rotation--2-in-1-sensor-hub-intel-ish-fix)
- [OLED Screen Care & Burn-in Prevention](#oled-screen-care--burn-in-prevention)
- [References](#references)

---

## Hardware Compatibility

Most laptop components work out-of-the-box on recent Linux kernels (Fedora 44 KDE):

| Hardware Component | Status | Notes |
| :--- | :---: | :--- |
| **OLED Display** | **Working** | HDR, brightness control, and scaling work properly. |
| **Touchscreen & Stylus** | **Working** | Multi-touch and active stylus function out of the box. |
| **Keyboard & Touchpad** | **Working** | Function keys, backlight, gestures, and clickpad work. |
| **Sleep / Suspend (S2idle)** | **Working** | Suspends and resumes cleanly without high battery drain. |
| **Wi-Fi & Bluetooth** | **Working** | Intel Wi-Fi and Bluetooth connect reliably. |
| **Webcam & IR Camera** | **Working** | UVC camera works out of the box. |
| **Fingerprint Reader** | **Working** |  Can enroll finger print and unlock|
| **Audio (Internal Speakers)** | **Fix Required** | No sound initially; quad-speaker woofers stay muted without config fix. |
| **Auto-Rotation & Tent Mode** | **Fix Required** | Accelerometer/gyroscope fails due to OEM firmware signature mismatch. |

---

## Installation Guide

### Tested Environment
* **Distribution**: Fedora 44 KDE
* **Kernel**: `7.2.11`
* Future distributions and newer kernels should work. If you are installing an older distribution or kernel, refer to [johnmeade/linux-yoga-9i-2-in-1-aura](https://github.com/johnmeade/linux-yoga-9i-2-in-1-aura) for additional troubleshooting steps.

---

### BIOS Configuration (Secure Boot & 3rd Party Certificates)

To boot and install Linux from a live USB drive, you must adjust the UEFI/BIOS security settings:

1. **Enter BIOS**:
   Power on the laptop and immediately press `F2` repeatedly (you may need to do `Fn + F2` depending on your function key configuration).
2. **Navigate to the Security Tab**:
   * Set **`Secure Boot`** &rarr; **`Disabled`**  
     *(Required to allow the live USB installation media to boot).*
   * Set **`Allow 3rd party UEFI certificates`** &rarr; **`Enabled`**  
     *(Required for Linux bootloaders and drivers signed by Microsoft 3rd-party UEFI CA).*
3. **Save and Exit**:
   Press `F10` (or select *Save Changes and Exit*).
4. **Post-Installation Note**:
   After Linux is successfully installed, you can re-enable **`Secure Boot`** in the BIOS for extra security. Leave **`Allow 3rd party UEFI certificates`** set to **`Enabled`**.

### BitLocker Dual-Boot Note

> [!WARNING]
> **Windows BitLocker Recovery Notice**:
> If you are dual-booting with an existing Windows installation that uses BitLocker drive encryption, Windows may prompt for your BitLocker recovery key or refuse to boot while Secure Boot is disabled.
>
> **Do not panic**: This is standard BitLocker security behavior. Once you finish installing Linux and re-enable **Secure Boot** in the BIOS (with 3rd-party certificates enabled), Windows will boot normally again without demanding recovery keys. Keep your BitLocker recovery key handy just in case.

---

## Quirk Fixes

### 1. Audio Setup & Quad-Speaker (Woofers/Bass) Fix

#### Background
This laptop uses an **Intel SoundWire (SOF)** audio topology with two distinct codecs:
* **Realtek RT721 (`ALC721`)**: Handles the 3.5mm combo jack and internal amplifier driving the **2 regular/tweeter speakers**.
* **Realtek RT1320**: A dedicated smart amplifier driving the **2 woofer/bass speakers**.

Out of the box, audio is completely silent because the ALSA Use Case Manager (UCM) lacks the required configuration files. After applying basic UCM files, sound is often "tinny" because the woofers remain physically muted.

Related issue: https://github.com/alsa-project/alsa-ucm-conf/issues/851

---

#### Step 1: Update ALSA UCM Configuration (Enable Base Sound)

Fetch the latest upstream configuration from the `alsa-ucm-conf` repository to provide ALSA and PipeWire with the split `rt721.conf` and `rt1320.conf` profiles:

```bash
# Download latest upstream ALSA UCM configuration
curl -L -o alsa-ucm-conf.tar.gz https://github.com/alsa-project/alsa-ucm-conf/archive/refs/heads/master.tar.gz

# Extract configuration directly into /usr/share/alsa/
sudo tar xvzf alsa-ucm-conf.tar.gz -C /usr/share/alsa --strip-components=1 --wildcards "*/ucm" "*/ucm2"

# Clean up archive
rm alsa-ucm-conf.tar.gz
```

*Reboot your system.* At this stage, sound should work through the tweeters, but the woofers will still be inactive.

---

#### Step 2: Unmute the Woofers (Hardware Switches)

The RT1320 woofer amplifier has hardware output gates exposed to ALSA that are muted (`[off]`) by default. Unmute them and set them to full scale:

```bash
# Unmute the left and right RT1320 woofer channels
amixer -c0 sset "rt1320-1 OT23 L" on
amixer -c0 sset "rt1320-1 OT23 R" on

# Set output volumes to 100% (0.00 dB)
amixer -c0 sset "rt721 FU06" 100%      # Tweeters / Primary volume
amixer -c0 sset "rt1320-1 FU21" 100%   # Woofers / Bass volume
```

This should immediately open the audio gate on the RT1320 smart amplifier without requiring a restart.

---

#### Step 3: Permanent Fix in ALSA UCM (`rt721.conf`)

Due to an upstream scoping bug in `/usr/share/alsa/ucm2/sof-soundwire/rt721.conf`, the `If.amp` macro for the RT1320 is placed **outside** the `SectionDevice."Speaker"` definition. Because of this, PipeWire never executes the RT1320 unmute macro when switching to the internal speakers.

Edit `/usr/share/alsa/ucm2/sof-soundwire/rt721.conf`:

```bash
# Use Nano or any other text editor
sudo nano /usr/share/alsa/ucm2/sof-soundwire/rt721.conf
```

Move the `If.amp` block **inside** `SectionDevice."Speaker"`:

```diff
--- a/usr/share/alsa/ucm2/sof-soundwire/rt721.conf
+++ b/usr/share/alsa/ucm2/sof-soundwire/rt721.conf
@@ -67,17 +67,17 @@ If.codecspk {
 		String "${var:SpeakerCodec1}"
 	}
 	True {
-		If.amp {
-			Condition {
-				Type String
-				String1 "${var:SpeakerAmp1}"
-				String2 "rt1320"
-			}
-			True.Macro.num1.rt1320AmpNum { }
-		}
-
 		SectionDevice."Speaker" {
 			Comment "Speaker"
 
+			If.amp {
+				Condition {
+					Type String
+					String1 "${var:SpeakerAmp1}"
+					String2 "rt1320"
+				}
+				True.Macro.num1.rt1320AmpNum { }
+			}
+
 			EnableSequence [
 				cset "name='Speaker Switch' on"
 			]
```

---

#### Step 4: Persist Volume & Mixer State

Save the unmuted mixer state to `/var/lib/alsa/asound.state` so settings persist across reboots:

```bash
sudo alsactl store
```

#### Step 5: Verification

Verify that both RT1320 output switches report `[on]` and test speaker playback:

```bash
# Check switch states
amixer -c0 sget "rt1320-1 OT23 L"
amixer -c0 sget "rt1320-1 OT23 R"

# Play sound using browser or audio player
```

All 4 speakers (2 regular tweeters + 2 bass woofers) will now fire concurrently, restoring full dynamic range and bass.

---

### 2. Auto-Rotation & 2-in-1 Sensor Hub (Intel ISH Fix)

#### Background
The 2-in-1 tablet/tent mode sensors, accelerometer, and gyroscope are managed by the **Intel Integrated Sensor Hub (ISH)** (PCI ID `[8086:e445]`, Subsystem ID `[17aa:3818]`).

Under Linux, the kernel ISH driver attempts to load firmware via the ISHTP bootloader. The generic firmware image included in `linux-firmware` (`ish_ptl.bin`) is signed with Intel reference/engineering keys. Lenovo hardware security fuses reject this image during validation, causing `cmd 2 failed 10` errors and disabling auto-rotation:

```text
Output of `sudo dmesg | grep -i ish`:

intel_ish_ipc 0000:00:12.0: ISH loader: load firmware: intel/ish/ish_ptl.bin
intel_ish_ipc 0000:00:12.0: ISH loader: cmd 2 failed 10
intel_ish_ipc 0000:00:12.0: Timed out waiting for HW ready
```

To resolve this, you need to copy the Lenovo's production-signed OEM binary from Windows (`PTL_ishS_SI_5.8.1.7785.bin`, 477,696 bytes).

---

#### Step 1: Obtain the Lenovo OEM Firmware Binary

You can extract this file directly from the Windows partition driver store:

* **Location in Windows**:
  `C:\Windows\System32\DriverStore\FileRepository\ishheciextensiontemplate.inf_amd64_*\FwImage\0004\PTL_ishS_SI_5.8.1.7785.bin`
* *Details*: File size is exactly `477,696 bytes` (project version `1.0.0.13558`).


---

#### Step 2: Deploy Firmware to Linux System Directory

The Linux ISH driver checks for OEM-specific firmware matching the DMI CRC32 checksum (`sys_vendor: LENOVO` = `53c4ffad`, `product_sku` = `4da64bbf`) before falling back to generic names.

Copy the binary and create fallback symlinks:

```bash
# Ensure target directory exists
sudo mkdir -p /lib/firmware/intel/ish

# Copy the OEM binary with the DMI SKU filename
sudo cp PTL_ishS_SI_5.8.1.7785.bin /lib/firmware/intel/ish/ish_ptl_53c4ffad_4da64bbf.bin

# Create fallback symlinks
sudo ln -sf ish_ptl_53c4ffad_4da64bbf.bin /lib/firmware/intel/ish/ish_ptl_53c4ffad.bin
sudo ln -sf ish_ptl_53c4ffad_4da64bbf.bin /lib/firmware/intel/ish/ish_ptl.bin
```

---

#### Step 3: Rebuild the Initramfs

The ISH driver initializes in early boot (~1.3s), before the root filesystem is mounted. Rebuild your initramfs so the new firmware is packaged into early boot memory:

* On **Fedora / RHEL**:
  ```bash
  sudo dracut -f
  ```
* On **Arch Linux**:
  ```bash
  sudo mkinitcpio -P
  ```
* On **Ubuntu / Debian**:
  ```bash
  sudo update-initramfs -u
  ```

---

#### Step 4: Perform a Cold Reboot

```bash
sudo reboot
```

*Explanation*: A full hardware reboot resets the ISH microcontroller if it entered an error lockout state after previous failed boot attempts.

---

#### Step 5: Verification

After rebooting, check that the firmware loaded cleanly and verify sensor output:

```bash
# Check dmesg for successful firmware load
dmesg | grep -i "ish loader"

# Test real-time sensor output (accelerometer & orientation)
monitor-sensor
```

`intel_ish_ipc` will now successfully attach, and display auto-rotation in tent, tablet, and portrait modes will work seamlessly in GNOME and KDE Plasma.

---

## Minor note: OLED Screen Care & Burn-in Prevention

This model is equipped with a vibrant OLED display panel. While modern OLED panels feature improved organic materials and subpixel wear-leveling that greatly reduce burn-in risks, Linux desktop environments (like GNOME and KDE Plasma) do not enable automated OLED protection measures by default.

To preserve display longevity and ensure uniform pixel degradation over years of usage, consider the following best practices:

1. **Use Dark Mode**:
   * Set your system theme and web browser to Dark Mode (*Settings &rarr; Appearance*).
   * OLED pixels produce black by turning off completely, saving battery and cutting OLED emitter stress to zero on dark regions.
2. **Shorten Screen Inactivity Timeout**:
   * Configure the screen blank timeout to a short duration (e.g., 3 to 5 minutes of inactivity) in *Settings &rarr; Power &rarr; Screen Blank*.
3. **Auto-Hide Persistent Interface Elements**:
   * Static interface elements (taskbars, docks, and top bars) are the most common source of uneven pixel wear.
   * **GNOME**: Use the **Dash to Dock** extension with "Intelligent Auto-hide" enabled, or the **Hide Top Bar** extension to retract the status bar when not needed.
   * **KDE Plasma**: Right-click the panel &rarr; *Enter Edit Mode* &rarr; *More Custom Options* &rarr; set *Visibility* to **Auto-hide** or **Dodge Windows**.
4. **Dynamic Wallpapers**:
   * Set wallpapers to rotate periodically or use a dynamic wallpaper tool to avoid static desktop imagery.

---

## References

* [johnmeade/linux-yoga-9i-2-in-1-aura](https://github.com/johnmeade/linux-yoga-9i-2-in-1-aura) – Linux installation and quirk guide for Yoga 9i Aura Edition.
* [alsa-project/alsa-ucm-conf](https://github.com/alsa-project/alsa-ucm-conf) – ALSA Use Case Manager configuration files.
* [Intel Sound Open Firmware (SOF)](https://thesofproject.github.io/latest/index.html) – Sound Open Firmware documentation.
* [Linux Kernel Intel ISH HID Driver](https://github.com/torvalds/linux/tree/master/drivers/hid/intel-ish-hid) – Kernel documentation for Intel Integrated Sensor Hub.


PS: AI is so powerful now, most of these are figured out by antigravity (google's coding agent). If you are blocked at any step, just ask AI.
