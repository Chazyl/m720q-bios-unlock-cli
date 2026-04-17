# 🛡️ M720q BIOS Unlock Guide (Linux/Bazzite)
**BIOS Version:** M1UKT6AA | **Target:** ThinkCentre M720q / M920q / P330 Tiny

This guide explains how we bypassed the physical BIOS menu constraints to surgically unlock **CFG Lock** and **Overclocking Lock** directly from the Linux terminal. This enables undervolting via `intel-undervolt` or `ThrottleStop`.

---

## ⚠️ WARNING
Modifying UEFI variables (`efivars`) carries a risk of bricking your system. This procedure involves direct hex manipulation of the BIOS NVRAM. If the system fails to POST, you must perform a CMOS reset (remove the coin battery or use the CLR_CMOS jumper).

---

## 1. Prerequisites
Ensure `efivarfs` is mounted read-write (standard on most modern Linux distros like Bazzite/Fedora).

```bash
mount | grep efivars
# Output should show (rw,...)
```

Identify your `Setup` variable GUID (the ID varies by BIOS version). For `M1UKT6AA`, it is:
`Setup-ec87d643-eba4-4bb5-a1e5-3f3e36b20da9`

---

## 2. Identify the Offsets
For BIOS version **M1UKT6AA**, the critical offsets in the `Setup` variable are:
- **0x3C (CFG Lock):** Controls MSR 0xE2 (0x01 = Locked, 0x00 = Unlocked)
- **0x7BD (Overclocking Lock):** Controls Voltage Offset access (0x01 = Locked, 0x00 = Unlocked)

*Note: In the Linux `efivars` file system, these files have a 4-byte header, so the physical file offsets are Offset + 4.*

---

## 3. The Surgical Unlock Procedure

### Step A: Remove Immutability
Linux protects UEFI variables with an immutable flag. You must remove it to write changes.

```bash
sudo chattr -i /sys/firmware/efi/efivars/Setup-ec87d643-eba4-4bb5-a1e5-3f3e36b20da9
```

### Step B: Modify the Variables (Python Script)
Run this Python script to surgically flip the bits at the correct hex locations without corrupting the rest of the 5004-byte variable.

```python
import os

guid = "-ec87d643-eba4-4bb5-a1e5-3f3e36b20da9"
setup_path = "/sys/firmware/efi/efivars/Setup" + guid

with open(setup_path, "rb") as f:
    data = bytearray(f.read())

# Apply +4 byte header offset for Linux efivars
# 0x3C -> Index 64
# 0x7BD -> Index 1985
print(f"Current CFG Lock (0x3C): {data[64]:02x}")
print(f"Current OC Lock (0x7BD): {data[1985]:02x}")

# Set to 0x00 (Disabled)
data[64] = 0x00
data[1985] = 0x00

with open(setup_path, "wb") as f:
    f.write(data)
print("SUCCESS: BIOS bits flipped surgically.")
```

### Step C: Restore Protection
Re-enable the immutable flag to prevent unauthorized or accidental BIOS corruption.

```bash
sudo chattr +i /sys/firmware/efi/efivars/Setup-ec87d643-eba4-4bb5-a1e5-3f3e36b20da9
```

---

## 4. Verification & Application
**Reboot the system.**

Once back in Linux, test if the undervolt applies:
```bash
sudo intel-undervolt apply
sudo intel-undervolt read
```

If successful, you will see your requested offsets (e.g., `-80.00 mV`) instead of `-0.00 mV`.

---

## 5. Persistence (Bazzite/systemd)
Create a systemd service to ensure the undervolt and power limits (PL1/PL2) are applied every time you boot.

**File:** `/etc/systemd/system/m720q-tuning.service`
```ini
[Unit]
Description=M720q CPU Undervolt & Power Tuning
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/bin/intel-undervolt apply

[Install]
WantedBy=multi-user.target
```

Enable it:
```bash
sudo systemctl enable --now m720q-tuning.service
```
