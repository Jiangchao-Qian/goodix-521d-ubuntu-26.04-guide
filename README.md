# Goodix 27c6:521d Fingerprint Reader on Ubuntu 26.04 — Progress Log

**Hardware:** ASUS ROG Zephyrus G14 (2020 GA401), Ryzen 9 4900HS
**Sensor:** Shenzhen Goodix Technology USB `27c6:521d` (built-in)
**OS:** Ubuntu 26.04, Python 3.14
**Date:** 2026-09-21 → 2026-09-22
**Status:** ✅ Fully working — enrollment, verification, and PAM login (sudo + GDM) all functional

---

## What works

- ✅ Firmware flashed to `GFUSB_GM168SEC_APP_10019` (persists on chip)
- ✅ OTP read fixed — the 2-byte payload is a little-endian length (`0x40,0x00` = 64 bytes)
- ✅ Community driver `arsfeld/libfprint-goodixtls` built and installed to `/usr/local` (shadows distro lib)
- ✅ Driver frame-assembly bug **fixed** (see section 5) — verification now matches reliably
- ✅ Distro packages pinned (`libfprint-2-2`, `fprintd`, `libpam-fprintd` on hold)
- ✅ `fprintd` recognizes the sensor as "Goodix TLS Fingerprint Sensor 52XD"
- ✅ Fingerprint enrollment completes
- ✅ `fprintd-verify` returns `verify-match` with genuine NBIS scores (19/24, 30/24)
- ✅ PAM fingerprint auth enabled — sudo and GDM login accept fingerprint

## Practical notes

- The 521d sensing area is tiny (64×80 px). Broader/flatter fingers (thumbs) verify far more reliably than narrower ones — both thumbs enrolled and working here.
- The finger-name labels (`-f right-thumb` etc.) are just labels; the sensor records whatever finger you place during enrollment. Keep labels accurate to avoid confusion.

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
fprintd-enroll -f right-thumb    # 5 presses with the actual finger
fprintd-enroll -f left-thumb
fprintd-list charles-tsien      # confirm both stored
```

PAM (keep a `sudo -i` shell open in another terminal as a safety net first!):
```bash
sudo pam-auth-update           # check "Fingerprint authentication"
```

Test with `sudo -k && sudo -v` — the sensor prompts for your finger. Password auth still works as fallback (fingerprint is "sufficient", not "required").

---

## 5. The verify-no-match bug: root cause and fix (2026-09-22)

After the driver was built, enrollment succeeded but `fprintd-verify` **always** returned `verify-no-match`, with the matcher scoring `0/24`. The sensor hardware and per-frame capture were proven good — the corruption was in the driver's image assembly.

### How it was debugged

A temporary instrumentation patch (`goodix52xd-dump.patch`, kept in this folder) was applied to `goodix52xd.c`. It wrote each normalized 64×80 frame and the final assembled `FpImage` to `/tmp/goodix-dump/` as PGM files, and logged per-frame movement deltas.

One capture produced 22 debug lines: 10 frame dumps, 10 movement deltas, and 1 assembled image. The deltas were the smoking gun — frames 1–9 all reported approximately `delta_x=-8, delta_y=-79`: near-full-frame "motion" between frames of a perfectly stationary finger press.

The dumped images confirmed it visually:
- Each individual 64×80 frame showed **clean, clear fingerprint ridges** — the sensor and OTP-calibrated capture path were fine.
- The assembled image fed to NBIS was **192×791 pixels**: a diagonal stair-step of 10 disjoint tiles, each offset by the hallucinated motion. Not a fingerprint image at all.

### Root cause

The 521d is a **static press sensor**, but the driver applied **swipe-sensor** frame assembly (`fpi_do_movement_estimation` + `fpi_assemble_frames`, with `image_width = WIDTH*3`). The movement estimator — whose search cannot represent the true zero-motion case — hallucinated ~79 px of vertical motion per frame and smeared 10 identical static frames into an unmatchable strip. NBIS extracted no usable minutiae → `0/24` → `verify-no-match`, every time.

(The earlier width/height-swap theory was rejected: decoded frames are correctly 64×80.)

### The fix

Replaced the swipe-style assembly with a press-sensor strategy: **average the 10 static frames into a single clean 64×80 image** (`goodix52xd-fix.patch`, kept in this folder):

```c
/* The 521d is a static press sensor, not a swipe sensor: the captured
 * frames all show a stationary finger. Swipe-style movement estimation
 * hallucinates motion and smears them into an unmatchable strip, so
 * average the frames into a single clean press image instead. */
FpImage* img = fp_image_new(GOODIX52XD_WIDTH, GOODIX52XD_HEIGHT);
{
    guint nframes = g_slist_length(frames);
    gsize npix = (gsize) GOODIX52XD_WIDTH * GOODIX52XD_HEIGHT;
    for (gsize i = 0; i < npix; i++) {
        guint sum = 0;
        for (GSList *l = frames; l; l = l->next)
            sum += ((struct fpi_frame *) l->data)->data[i];
        img->data[i] = (guint8) (sum / nframes);
    }
}
```

The now-unused `fpi_do_movement_estimation` / `fpi_assemble_frames` calls and the `get_pix` helper were removed. (A temporary dump of the final image was included during testing and removed after verification.)

### Verification

After rebuilding and reinstalling to `/usr/local`, old (garbage) enrollments were deleted and fingers re-enrolled fresh:

```
Verify result: verify-match (done)   # × multiple runs
```

The fprintd debug log showed genuine matcher scores — `score 19/24`, `score 30/24` — versus the permanent `0/24` before the fix. The matcher also correctly **rejects** non-enrolled fingers, confirming it is discriminating real minutiae, not matching noise. PAM fingerprint auth was then re-enabled: sudo and GDM login both accept the enrolled fingers.

---

## Security notes

- The community driver terminates the sensor's TLS with an **all-zero PSK**. Treat fingerprint login as convenience, not load-bearing security.
- Fingerprint login does **not** unlock the GNOME keyring — you'll still need your password once per session.
- Windows Hello enrollments are not reusable under Linux; fresh Linux enrollment is required.

---

## Files in this folder

- `README.md` — this progress log
- `goodix52xd-dump.patch` — temporary instrumentation used to diagnose the bug (dumps frames + deltas; not for production use)
- `goodix52xd-fix.patch` — the actual fix: press-sensor frame averaging replacing swipe assembly (includes a temporary final-image dump block, removed after verification)
- `sim/` — early static simulation of the frame assembly (superseded by real hardware dumps)

---

## References

- Firmware dump tool: `knauth/goodix-521d-explanation` (`goodix-fp-dump/run_521d.py`)
- Driver fork: `arsfeld/libfprint-goodixtls` (branch `goodixtls-521d-1.94.100`)
- Upstream (unsupported for 521d): libfprint 1.94.100
