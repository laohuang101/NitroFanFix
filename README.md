# NitroFanFix

A fix for Acer Nitro AN515-58 fan issues (vibration, abnormal noise, stuck fan behavior) on Linux using direct EC register control.

## Problem

On the Acer Nitro AN515-58 running Linux (tested on CachyOS / Arch), the laptop fan may exhibit:
- Abnormal vibration or rattling noise
- Unusual sound at certain speeds
- Fan behavior that seems "stuck" in a bad state

This is not always a hardware failure. The embedded controller (EC) can end up in a bad register state, and resetting it via EC register writes can restore normal fan behavior.

## Solution

This guide uses the [Jebaitedneko/Acer-Nitro-5-AN515-58-EC-Control-Linux](https://github.com/Jebaitedneko/Acer-Nitro-5-AN515-58-EC-Control-Linux) script to write directly to the EC registers that Acer's NitroSense software uses on Windows.

## Prerequisites

- Acer Nitro AN515-58 (may work on similar Nitro models, YMMV)
- Linux with `ec_sys` kernel module available
- `sudo` access
- `lm-sensors` installed (`sudo pacman -S lm_sensors` on Arch/CachyOS)

## Step 1: Install lm-sensors (if not already installed)

```bash
sudo pacman -S lm_sensors
sensors-detect --auto
```

## Step 2: Verify ec_sys module is available

```bash
find /lib/modules/$(uname -r) -type f -iname "*ec_sys*"
```

You should see something like:
```
/lib/modules/7.2.9-2-cachyos/kernel/drivers/acpi/ec_sys.ko.zst
```

If nothing shows up, the module isn't available for your kernel. On Arch/CachyOS you may need the `linux-headers` package.

## Step 3: Download the script

```bash
curl -sL https://raw.githubusercontent.com/Jebaitedneko/Acer-Nitro-5-AN515-58-EC-Control-Linux/main/nitrosense -o ~/nitrosense
chmod +x ~/nitrosense
```

## Step 4: Load the EC module

```bash
sudo modprobe ec_sys write_support=1
```

Verify the EC interface is accessible:

```bash
sudo bash ~/nitrosense r
```

If you see hex dump output, the EC interface is working. If you see "EC interface not enabled", check for errors with `sudo dmesg | tail`.

## Step 5: Check current temperatures

```bash
sensors
```

Note your current temps so you have a baseline.

## Step 6: Reset the EC fan registers

The fix that resolved the vibration/abnormal noise was simply writing to the EC registers, which resets the fan controller to a clean state:

```bash
sudo bash ~/nitrosense pc 0
```

This sets Performance power + Custom fan at 0%. In practice, the EC's self-protection mechanism keeps the fan spinning (it won't allow the fan to fully stop when there's heat), but the register write resets the fan controller, clearing the bad state.

> **Note:** The fan will spin fast briefly because this command also sets PL1/PL2 to 95W (Performance mode), which increases heat output. The important thing is that the abnormal vibration/noise should be gone after this.

## Step 7: Return to normal operation

After the EC is reset, switch back to a normal operating mode:

```bash
# Default power + Auto fan (recommended for normal use)
sudo bash ~/nitrosense da

# Quiet power + Auto fan (lowest noise, 10W TDP)
sudo bash ~/nitrosense qa
```

Verify the fan is running normally (no vibration, no abnormal sound):

```bash
sensors
```

## Available Commands

| Command | Description |
|---------|-------------|
| `sudo bash nitrosense` | Show help |
| `sudo bash nitrosense da` | Default power + Auto fan |
| `sudo bash nitrosense dm` | Default power + Max fan |
| `sudo bash nitrosense pc 50` | Performance power + Custom fan at 50% |
| `sudo bash nitrosense qa` | Quiet power + Auto fan (10W, lowest noise) |
| `sudo bash nitrosense r` | Read/dump EC registers (debug) |

### Switch reference

- **Power:** `q`=quiet (10W), `d`=default (75W), `p`=performance (95W)
- **Fan:** `a`=auto (EC managed), `c`=custom (needs percentage), `m`=max

## Make it persistent across reboots

The `ec_sys` module needs to be loaded on every boot. You can make it persistent:

### Option A: systemd module load

```bash
echo "ec_sys" | sudo tee /etc/modules-load.d/ec_sys.conf
echo "options ec_sys write_support=1" | sudo tee /etc/modprobe.d/ec_sys.conf
```

### Option B: kernel cmdline (alternative)

Add `ec_sys.write_support=1` to your kernel parameters in `/etc/default/grub`:

```bash
sudo nano /etc/default/grub
```

Find the `GRUB_CMDLINE_LINUX_DEFAULT` line and add `ec_sys.write_support=1`:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet ... ec_sys.write_support=1"
```

Then regenerate grub config:

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

## Disclaimer

- Writing directly to EC registers carries risk. If wrong values are written, it could affect hardware behavior.
- This guide is specifically for the Acer Nitro AN515-58. Other models may use different EC register mappings.
- The `nitrosense` script is from [Jebaitedneko's repo](https://github.com/Jebaitedneko/Acer-Nitro-5-AN515-58-EC-Control-Linux) — credit to the original author.
- If your fan has a genuine hardware failure (bearing damage, physical obstruction), software fixes won't help. You'll need to physically clean or replace the fan.

## Credits

- Original script: [Jebaitedneko/Acer-Nitro-5-AN515-58-EC-Control-Linux](https://github.com/Jebaitedneko/Acer-Nitro-5-AN515-58-EC-Control-Linux)
