# ThinkPad X1 Carbon Gen 10 - Ubuntu Lid-Close Battery Drain Fix

## Problem

Closing the laptop lid did not stop power consumption - the battery drained
completely within about a day while the machine was supposedly asleep.

- Machine: ThinkPad X1 Carbon Gen 10 (21CBA002CD), BIOS N3AET89W (1.54)
- OS: Ubuntu 26.04 LTS, kernel 7.0.0-27-generic

## Root Cause

Two separate issues were identified:

1. **Default sleep mode was `s2idle` (Modern Standby).**
   Out of the box the kernel used `s2idle`, which keeps the system in a
   low-power idle state that drains the battery quickly on this model.
   The firmware does support real S3 (`deep`) sleep, but it was not the
   default after boot.

2. **Spurious wake events from USB/Thunderbolt controllers.**
   Even when suspending into deep S3, the machine woke up ~11 seconds
   later. The `XHCI` (USB) and `TXHC` (Thunderbolt) ACPI wake sources were
   enabled, so devices such as the fingerprint reader, Quectel EM05-CE LTE
   modem, or AX211 Bluetooth could wake the laptop. It then sat there
   fully powered-on with the lid closed until the battery died.

## Diagnosis Commands

```bash
# Which sleep modes are supported / selected ([x] = current)
cat /sys/power/mem_sleep
# -> s2idle [deep]     (deep = S3, supported but not default at boot)

# ACPI wake sources (enabled devices can wake the machine)
grep -E 'enabled|disabled' /proc/acpi/wakeup

# Suspend history - look for short sleep cycles or immediate wakes
journalctl -b 0 | grep -E 'PM: suspend entry|Waking up from|suspend exit'
# Bad:  entry 22:45:31 (deep) -> Waking up 22:45:42  (11s later!)

# Attached USB devices (potential wake sources)
lsusb
```

## Fix

### 1. Make deep S3 the default at boot

Add `mem_sleep_default=deep` to the kernel command line:

```bash
sudo sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT="/GRUB_CMDLINE_LINUX_DEFAULT="mem_sleep_default=deep /' /etc/default/grub
sudo update-grub
```

Verify after the next reboot with `cat /proc/cmdline`.

### 2. Disable USB/Thunderbolt ACPI wake sources

`/proc/acpi/wakeup` resets on every boot, so persist the toggles with a
systemd oneshot service:

```bash
sudo tee /etc/systemd/system/disable-usb-wake.service <<'EOF'
[Unit]
Description=Disable USB/Thunderbolt ACPI wake sources
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'for d in XHCI TXHC TDM0 TDM1 TRP0 TRP2 PEG0; do grep -q "$d.*enabled" /proc/acpi/wakeup && echo $d > /proc/acpi/wakeup || true; done'

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable --now disable-usb-wake
```

`LID` stays enabled so opening the lid still wakes the machine.

### 3. Optional extras

- BIOS (F1): disable "Always On USB" to reduce standby drain further.
- `sudo fwupdmgr get-updates` - keep firmware current.
- If the LTE modem is unused: `sudo systemctl disable --now ModemManager`.

## Verification

```bash
journalctl -b 0 | grep -E 'PM: suspend entry|Waking up from|suspend exit'
```

- The suspend entry should say `(deep)`.
- The next "Waking up" line should be exactly when the lid was opened -
  nothing in between.

### Results

| Scenario | Sleep duration | Wake events | Battery drain |
|---|---|---|---|
| Before fix | 5h34m in s2idle | machine effectively on | dead in ~1 day |
| Short test | 13m45s in deep S3 | 0 (lid only) | negligible |
| Overnight | 10h10m in deep S3 | 0 (lid only) | ~1% (71% -> 70%) |

Expected long-term drain in suspend: roughly 1% per 10 hours (~2.5%/day).

## References

Note: this issue was diagnosed from on-machine logs; there is no official
Ubuntu or Lenovo document describing it. The pages below are general Linux
references - the mechanisms (kernel sleep modes, ACPI wakeup) are
distro-agnostic, though the Arch Wiki pages use Arch-specific commands in
places (e.g. mkinitcpio). They may be especially useful if switching to
Arch Linux.

- Kernel docs (official): System Sleep States
  https://docs.kernel.org/admin-guide/pm/sleep-states.html
  Documents `/sys/power/mem_sleep` and the `mem_sleep_default` kernel
  parameter; notes that some ACPI systems default to `s2idle` even when
  S3 ("deep") is supported.
- Arch Wiki: Power management/Suspend and hibernate
  https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate
  "Changing suspend method" covers `mem_sleep_default=deep` and the
  `MemorySleepMode=deep` option in systemd's sleep.conf.
- Arch Wiki: Wakeup triggers
  https://wiki.archlinux.org/title/Wakeup_triggers
  Documents "instantaneous wakeup after suspending" and disabling ACPI
  wake sources via `/proc/acpi/wakeup`.
