# Goodix 27c6:521d Fingerprint Reader on Ubuntu 26.04 — Progress Log

**Hardware:** ASUS ROG Zephyrus G14 (2020 GA401), Ryzen 9 4900HS  
**Sensor:** Shenzhen Goodix Technology USB `27c6:521d` (built-in)  
**OS:** Ubuntu 26.04, Python 3.14  
**Date:** 2026-09-21  
**Status:** Driver built and installed, enrollment works, verification matching still broken (driver bug under investigation)

---

## What works

- ✅ Firmware flashed to `GFUSB_GM168SEC_APP_10019` (persists on chip)
- ✅ OTP read fixed — the 2-byte payload is a little-endian length (`0x40,0x00` = 64 bytes)
- ✅ Community driver `arsfeld/libfprint-goodixtls` built and installed to `/usr/local` (shadows distro lib)
- ✅ Distro packages pinned (`libfprint-2-2`, `fprintd`, `libpam-fprintd` on hold)
- ✅ `fprintd` recognizes the sensor as "Goodix TLS Fingerprint Sensor 52XD"
- ✅ Fingerprint enrollment completes (tested with 2 fingers)
- ✅ PAM integration triggers the sensor on sudo (currently disabled pending verify fix)

## What does not work yet

- ❌ **Verification never matches** (`fprintd-verify` returns `verify-no-match` even for freshly enrolled fingers). Suspected driver image-processing bug — see "Known bug" below.

---

## Step-by-step: what was done

### 1. Firmware flash (knauth's goodix-fp-dump)

Cloned `knauth/goodix-521d-explanation`, set up a venv, ran `run_521d.py`:

```bash
cd ~/goodix-521d-explanation/goodix-fp-dump
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt   # needs python3.14-venv and python3.14-dev first
sudo .venv/bin/python3 run_521d.py
```

The flash succeeded: `Firmware: GFUSB_GM168SEC_APP_10019`, `Valid PSK: True`. The large hex dump during flashing is normal verbose output, not damage.

### 2. The OTP fix (key discovery)

After flashing, the script failed at `read_otp()` with `ValueError: Invalid OTP` — the OTP read returned 0 bytes.

**Root cause:** the OTP request's 2-byte payload is a little-endian u16 *length*. The script sent `b"\x00\x00"` (request 0 bytes). This unit's firmware honors the length field (older firmware ignored it and dumped all 64).

**Fix in `goodix.py`:** changed the payload from `b"\x00\x00"` to `b"\x40\x00"` (64 decimal = 0x40):

```python
# before
encode_message_protocol(b"\x00\x00", COMMAND_READ_OTP)
# after
encode_message_protocol(b"\x40\x00", COMMAND_READ_OTP)
```

This returned the genuine 64-byte OTP (header `4e4b35594c2e` = "NK5YL.", per-unit calibration tail). After patching, the full smoke test passed: OTP ✓, TLS ✓, FDT mode ✓, finger-press detected ✓, 80×64 image captured ✓.

> **Lesson:** never `^C` the script while it's in FDT/finger-detection mode — it wedges the sensor's MCU. Recovery is a USB port reset via the `USBDEVFS_RESET` ioctl (stronger than unbind/bind), not a reboot:
>
> ```python
> import fcntl
> fd = open('/dev/bus/usb/<bus>/<dev>', 'wb')
> fcntl.ioctl(fd, (ord('U') << 8) | 20)  # USBDEVFS_RESET
> ```

### 3. Building the libfprint driver

```bash
sudo apt install -y meson ninja-build libglib2.0-dev libgusb-dev \
    libpixman-1-dev libssl-dev libgirepository1.0-dev gtk-doc-tools libudev-dev
git clone https://github.com/arsfeld/libfprint-goodixtls.git
cd libfprint-goodixtls
# branch: goodixtls-521d-1.94.100
```

**Driver-side OTP fix:** the fork's `goodix_send_read_otp` (`libfprint/drivers/goodixtls/goodix.c`) already attempted the `{0x40, 0x00}` payload, but declared it as a scalar instead of an array:

```c
// before (bug: sizeof == 1, sends truncated 1-byte payload)
guint8 payload = {0x40, 0x00};
// after (sends the full 2 bytes the sensor requires)
guint8 payload[] = {0x40, 0x00};
```

**Build** (only the 52xd driver; udev rules/hwdb disabled since fprintd runs as root):

```bash
meson setup build --prefix=/usr/local -Ddoc=false -Dintrospection=false \
    -Ddrivers=goodixtls52xd -Dudev_rules=disabled -Dudev_hwdb=disabled
ninja -C build
sudo ninja -C build install
sudo ldconfig
```

**Verify shadowing:**
```bash
ldconfig -p | grep libfprint-2.so.2
# /usr/local/... must come FIRST
ldd /usr/libexec/fprintd | grep libfprint
# must point to /usr/local/lib/x86_64-linux-gnu/libfprint-2.so.2
```

**Pin packages** so Ubuntu updates don't overwrite the driver:
```bash
sudo apt-mark hold libfprint-2-2 fprintd libpam-fprintd
```

### 4. Enrollment and PAM

```bash
sudo systemctl restart fprintd
fprintd-enroll                 # right index (5-6 presses)
fprintd-enroll -f left-index-finger
fprintd-list charles-tsien     # confirm both stored
```

PAM (keep a `sudo -i` shell open in another terminal as a safety net first!):
```bash
sudo pam-auth-update           # check "Fingerprint authentication"
```

Test with `sudo -k && sudo -v` — the sensor is invoked, but verification currently fails (see below), so PAM was disabled again via `pam-auth-update` until the driver is fixed. Password auth is unaffected.

---

## Known bug: verification never matches

`fprintd-verify` consistently returns `verify-no-match`, even immediately after a successful enrollment. Enrollment captures images fine; the failure is in the match stage.

**Suspects** (under investigation):
- The driver does swipe-style frame assembly (`GOODIX52XD_CAP_FRAMES=10`, `fpi_do_movement_estimation`, `image_width = WIDTH*3`) for what is physically a press sensor.
- Possible width/height swap: the sensor reports 80×64 images, the driver defines `WIDTH=64, HEIGHT=80`.
- `rotate_frame()` in `goodix52xd.c` contains a broken transpose for non-square dimensions (dead code — never called — but indicates the orientation handling was never validated).

**Next step:** dump the assembled `FpImage` from both enroll and verify paths and compare them visually to identify the corruption.

---

## Security notes

- The community driver terminates the sensor's TLS with an **all-zero PSK**. Treat fingerprint login as convenience, not load-bearing security.
- Fingerprint login does **not** unlock the GNOME keyring — you'll still need your password once per session.
- Windows Hello enrollments are not reusable under Linux; fresh Linux enrollment is required (done).

---

## References

- Firmware dump tool: `knauth/goodix-521d-explanation` (`goodix-fp-dump/run_521d.py`)
- Driver fork: `arsfeld/libfprint-goodixtls` (branch `goodixtls-521d-1.94.100`)
- Upstream (unsupported for 521d): libfprint 1.94.100
